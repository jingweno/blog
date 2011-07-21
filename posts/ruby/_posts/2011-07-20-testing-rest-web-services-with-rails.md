---
layout: post
title: Testing REST Web Services with Rails
tags: ruby rails web_service REST
---

Testing REST web services has never been easy. It requires a running web
container, multiple threads, network conection and complex transaction management.

Ideally, web service client test should have the following characteristics:

1. The experience of testing web service API is similar to that of
   testing a ActiveRecord model
2. Start up and shut down the web server for the purpose of running REST web services
3. Rollback test data after each test
4. Control fixture creation for REST web services

In this article, I demonstrate solutions to each of those mentioned.

As an example throughout the article, let's assume we are creating and testing web services for
a model called *Task*. Here is a sample action in the *TasksController* of the web server:

{% highlight ruby %}
# server/app/controllers/tasks_controller.rb

def index
  @tasks = Task.all
  render :status => :ok, :json => @tasks
end
{% endhighlight %}

We render *@tasks* as the JSON format where *to_json* is automatically
called on the object. When you run "*curl http://localhost:3000/tasks.json*", you will
get the following result:

{% highlight bash %}
$ curl http://localhost:3000/tasks.json
[{"id":1,"name":"Write a blog post","created_at":"2011-07-20T04:05:41Z","updated_at":"2011-07-20T04:05:41Z","ends_at":"2011-08-20T03:15:00Z"}]
{% endhighlight %}

#### ActiveResource

In order to test our web services, we need a web service client. There
are [lots of them][6] out there, but I found [ActiveResource][5] the most
enjoyable to use in a less complex situation. ActiveResource provides ActiveRecord
compatible APIs, so when writing web service client tests, we will feel like we are
writing unit tests for a ActiveRecord model.

To start with, we just need to extend it from ActiveResource::Base and 
give it the web server URL and representation format. That's it!

{% highlight ruby %}
# client/app/models/task.rb

class Task < ActiveResource::Base
  self.site = "http://localhost:3000"
  self.format = :json
end
{% endhighlight %}

And we are using it as if you are using an ActiveRecord object:

{% highlight ruby %}
# client/spec/models/task_spec.rb

describe Task do
  it "should return all the tasks" do
    @tasks= Task.all
    @tasks.size.should == 1
  end
end
{% endhighlight %}

#### Web Server

To maintain a zero-setup test environment, we’ll have our test control the stratup and shutdown of
a web server. By having each test start and stop the web server, tests can be easily run with
no external dependencies.

To control the startup and shutdown of a web server before and after all
suites run, it's as simple as having something like this:

{% highlight ruby %}
# client/spec_helper.rb

RSpec.configure do |config|
  config.before(:suite) do
    @server = Server.new(server_path)
    @server.start
  end

  config.after(:suite) do
    @server.stop
  end
end
{% endhighlight %}

The implementation of *Server* is also dead simple. Execute "script/rails server -d" to
daemonize the server and issue a kill to stop it:

{% highlight ruby %}
# client/lib/server.rb

Class Server
  def initialize(server_path)
    @server_path = server_path
  end

  def start
    `#{rails_script} server -d -e test`
  end

  def stop
    pid = File.read(pidfile)
    `kill -9 #{pid}`
  end

  private

  def rails_script
    File.join(@server_path, 'script', 'rails')
  end

  def pidfile
    File.join(@server_path, 'tmp', 'pids', 'server.pid')
  end
end
{% endhighlight %}

#### Transaction Rollback

For testing strategies of web services, you probably will find most people
suggest to either truncate the test data on each run or to mock out the request and response.
These approaches are less ideal because it's either less effective or 
not testing the full stack of the web services.

Making transaction rollback for web service calls is difficult for the
following reasons:

* Tests and web server are running in two separate threads, web server's
  transactional boundary can't expand to tests

* Web service calls may commit its transaction

* Web server doesn't know when to rollback the test data

To overcome these problems, we’ll apply a little trickery:
we’ll make use of [dRuby][1] to directly control the lifecycle of transaction
on the web server.

For those who are not familiar with it, dRuby is as the **Remote Method Invocation** to Java as to Ruby.
it allows methods to be called in one Ruby process upon a Ruby object located in another Ruby process.
It's such a perfect match for controlling the lifecycle of the [ActiveRecord::Base.connection][2] object on the web server.

Add the following code to web server's "config/environments/test.rb":

{% highlight ruby %}
# server/config/environments/test.rb

config.after_initialize do
  ActiveRecord::ConnectionAdapters::ConnectionPool.class_eval do
    alias_method :old_checkout, :checkout

    def checkout
      @cached_connection ||= old_checkout
    end
  end

  require 'drb'
  DRb.start_service("druby://127.0.0.1:61191", ActiveRecord::Base.connection)
end
{% endhighlight %}

The code snippet does two things:

1. Patch *ActiveRecord::ConnectionAdapters::ConnectionPool#checkout* to make sure only one connection is shared
across threads
2. Start a dRuby service for ActiveRecord::Base.connection to be used in
   tests

In case you are wondering why it's necessary to share one database
connection across threads: [ActiveRecord creates one database connection for each thread][3]
and this implementation makes it impossible to track which connection to rollback data in web services calls.
What we are doing here is to make sure there is only one connection created and we always rollback data for this connection.

After the aforementioned setup, we are able to expand the transaction boundary to
tests:

{% highlight ruby %}
# client/spec/models/task_spec.rb

describe Task do
  before :all do
    DRb.start_service
    @remote_connection = DRbObject.new nil, "druby://127.0.0.1:61191"
  end

  before :each do
    begin_transaction
  end

  after :each do
    rollback_transaction
  end

  it "should ..." do
    # test REST web services calls
  end

  private

  def begin_transaction
    @remote_connection.increment_open_transactions
    @remote_connection.transaction_joinable = false
    @remote_connection.begin_db_transaction
  end

  def rollback_transaction
    @remote_connection.rollback_db_transaction
    @remote_connection.decrement_open_transactions
    @remote_connection.clear_active_connections!
  end
end
{% endhighlight %}

Voila! With dRuby, we use begin+rollback to isolate changes of web services calls to the database,
instead of having to delete+insert for every test case. A huge performance boost!

#### Fixtures Creation

Most of the time, we create fixtures for tests to quickly define prototypes for each
of the models and ask for instances with properties that are important to the test at hand. But in the context
of REST web services, we can't create fixtures unless there is a REST
API defined. To break this constraint, we use dRuby to open up another channel to interact with fixture data on web server.

Assuming we are using the [factory_girl][4] gem for fixtures creation,
We create a dRuby service for port discovery and a dRuby service for each fixture instance:

{% highlight ruby %}
# server/lib/drb_active_record_instance_factory.rb

require 'factory_girl'

class DRbActiveRecordInstanceFactory
  @@pot = 9001

  def get_port_for_fixture_instance(factory_instance)
    port = get_new_port
    inst = Factory.create(factory_instance)
    DRb.start_sevice('druby://127.0.0.1:#{port}', inst)
    port
  end

  def get_new_port
    # create a random port
  end
end

DRb.start_service('druby://127.0.0.1:9000', DRvActiveRcordInstanceFactory.new)
{% endhighlight %}

In tests, we ask for the port of the fixture instance and query its corresponding remote reference:

{% highlight ruby %}
# client/spec/models/task_spec.rb

describe Task do
  before :all do
    @drb_factory = DRbObject.new(nil, 'druby://127.0.0.1:9000')
  end

  before do
    remote_task_port = @drb_factory.get_port_for_fixture_instance(:post)
    @remote_task = DRbActiveRecord.new(nil, "druby://127.0.0.1:#{remote_post_port}")
  end

  it "should ..."
    # test REST web services calls with @remote_task
  end
end
{% endhighlight %}

#### Summary

Testing REST web services can be less complex if we can fully control objects on the web server. ActiveResource and dRuby
stand out and help. They make writing web service client tests feel like writing local unit tests.

[1]: http://www.ruby-doc.org/stdlib/libdoc/drb/rdoc/classes/DRb.html
[2]: http://ar.rubyonrails.org/classes/ActiveRecord/Base.html#M000431
[3]: https://github.com/rails/rails/blob/master/activerecord/lib/active_record/connection_adapters/abstract/connection_pool.rb#L160
[4]: https://github.com/thoughtbot/factory_girl/
[5]: http://api.rubyonrails.org/classes/ActiveResource/Base.html
[6]: http://ruby-toolbox.com/categories/http_clients.html