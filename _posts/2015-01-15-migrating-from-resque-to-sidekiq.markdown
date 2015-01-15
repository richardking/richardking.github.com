---
layout: post
title: Migrating from Resque to Sidekiq
categories: ruby rails
---


###Migrating from Resque to Sidekiq

Our API service relies heavily on background processing to cache endpoints to serve to desktop and mobile. Recently we migrated over from Resque to Sidekiq; in the past few months/years, Sidekiq has been more actively developed and has become the de-facto library for background processing.

While on Resque, at peak traffic, we've spun up as much as 36 c1.xlarge AWS servers, each running 12 Resque workers, to try and keep up with the queues. However, even then, at times jobs back up and we fall far behind.

Sidekiq boasts an architecture that can process jobs in a multi-threaded environment. Thus, instead of spinning up an instance of Rails for each worker, Sidekiq just needs to start one instance of your environment with a customizable "concurrency" setting, which is the number of workers processing in the same environment.

#### Moving the worker code over

Sidekiq claims to be a drop-in replacement for Resque, which I found for the most part to be true.

The first thing I updated were the workers. The main difference here is that worker methods for Resque are class methods, but worker methods for Sidekiq are instance methods. Within a worker, we ```include Sidekiq::Worker```, change ```def self.perform(args)``` to ```def perform(args)```, and that enables Sidekiq to access the workers. We also specify the queues in our workers, so we change ```@queue = :priority``` to ```sidekiq_options queue: :priority```.

The majority of our calls to Resque workers were in the form of:
```Resque.enqueue(WorkerName, arg1, arg2)```
In these cases, I just needed to change ```Resque``` to ```Sidekiq::Client```; the rest of the statement stays the same:
```Sidekiq::Client.enqueue(WorkerName, arg1, arg2)```. An alias for that syntax in Sidekiq is ```WorkerName.perform_async(arg1, arg2)```, which is a little less wordy and more straightforward.

#### Configuring Sidekiq

Configuring Sidekiq is pretty straightforward as well too. Sidekiq is on a client/server framework, which is explained [here](https://github.com/mperham/sidekiq/wiki/The-Basics). Essentially the client process runs your web app process and allows you to push jobs into the background for processing. The server process pulls jobs from the Redis queue and processes them.

Thus you need to configure both ```Sidekiq.client``` and ```Sidekiq.server``` but thankfully they are quick. In ```config/initializers/sidekiq.rb```, you need to require the sidekiq files, find your Redis url and point both the Sidekiq client and server at it. Our ```sidekiq.rb``` looks like this:
```
require 'sidekiq'
require 'sidekiq/web'

redis_config = YAML.load(IO.read("config/resque.yml"))[Rails.env]['api']
redis_url = "http://#{redis_config["host"]}:#{redis_config["port"]}"

Sidekiq.configure_server do |config|
  config.redis = { url: redis_url }
end

Sidekiq.configure_client do |config|
  config.redis = { url: redis_url }
end
```

I've also added a few other configuration lines to that file. One so that sidekiq doesn't retry a job over and over again if it fails:
```
Sidekiq.default_worker_options = { 'retry' => 1 }
```

Then, to secure the Sidekiq dashboard with a username/password (similar to the Resque dashboard), we add these lines:
```
Sidekiq::Web.use(Rack::Auth::Basic) do |user, password|
    [user, password] == ["username", "password"]
end
```

Finally, we want to mount the dashboard in our ```routes.rb``` file so we can access the web UI:
```
mount Sidekiq::Web => '/sidekiq'
```

#### Migrating Resque scheduler to Sidekiq scheduler
We use a Resque plug-in called ```resque-scheduler``` to schedule jobs to run. We have a ```resque_scheduler.yml``` in ```/config``` directory and it lists out the jobs in the following format
```
priority_canary:
  every: 1m
  class: ResqueCanary
  args: priority
  description: "Updates a timestamp on redis for up check purposes"
```
I found a Sidekiq plugin called (surprise) ```sidekiq-scheduler```, which basically ported Resque scheduler over.

I wanted to keep the system architectural pattern the same; namely, while on Resque, we had one stack running Resque scheduler and one stack running Resque workers, so we could scale the Resque workers as needed. In Resque, to start up the scheduler, you run ```rake resque:scheduler```, and it just runs the scheduler. However, Sidekiq is a little different- when you run sidekiq, it loads up all of sidekiq and its config file. Thus, if by default, you load up your scheduler configuration, all of the sidekiq processes will also schedule jobs, which is not what we want.

To resolve this, I created a specific config just for the scheduler process. You can custom load Sidekiq config using:
```bundle exec sidekiq -C ./config/sidekiq_schedule.yml```
In that config, I put all the scheduling config, and set ```concurrency``` to 0. With that command, the scheduler starts up, loads all the schedules and does not have any worker processing any jobs. Thus it acts just like Resque's scheduler, and so we could keep the same scheduler/worker architecture.

One gotcha with that configuration is that, by default, Sidekiq sets a Redis connection pool size by adding 2 to the concurrency configuration. Thus, since I set concurrency to 0, it had a pool size of 2. I think this caused errors down the line. I would see Timeout errors after scheduler ran for a few days:
```
2014-11-19_03:02:07.97707 2014-11-19T03:02:07.976Z 20668 TID-fa81g INFO: Timeout::Error: Waited 1 sec
```

I manually set the connection pool size in the Sidekiq config, and monitored to see if it fixed the issue. It still occurred twice in  the 3 weeks prior, so it seems like it still didn't fix the issue. So based on Mike Perham's [recommended](https://github.com/mperham/sidekiq/wiki/Scheduled-Jobs) way of scheduling jobs, I've looked into replacing sidekiq-scheduler with the Clockwork gem or the Whenever gem.

##### Whenever gem
The Whenever gem lets you create jobs in a clean and clear syntax and then translates your code into cron syntax, so that your system cron can parse and use it.

The problem I ran into when trying to use whenever is that it didn't seem to work well with rvm. The cron commands needed to be run in the context of a particular gemset but I couldn't get it to switch properly.

##### Clockwork gem
Clockwork, on the other hand, is billed as a cron replacement. It does not leverage your system cron, but instead runs as a lightweight Ruby process. So it isn't quite as system-level as the Whenever gem, but hopefully because it is so lightweight, it is stable.

Also, it seems to be a good replacement for sidekiq-scheduler because it can easily be configured to run on just one server (scheduler server).

Syntax for scheduling a job:
```
  every(90.minutes, 'update_content_player') do
    UpdateContentPlayer.perform_async
  end
```

Syntax for scheduling a job while specifying a queue and an argument:
```
  every(1.minute, 'priority_canary') do
    Sidekiq::Client.push("queue" => "priority", "class" => BackgroundWorkerCanary, "args" => ["priority"])
  end
```

#### Tweaking workers per thread

Finally, one last setting that I tweaked was regarding the number of workers running on production. I originally left concurrency at default (25 workers), so each Sidekiq process had 25 workers running. Then I ran 2 Sidekiq processes per Production worker server. This seemed to cause the jobs to run slowly.

So I tried to use a bigger/faster instance with more cores, and run fewer workers per core. I ended up using a c3.2xlarge instance with 10 workers per thread and 8 threads per server.

Not exactly back down to Resque-level execution time, but it improved a lot from the 25 workers per thread. I'm thinking the 12 workers/server (Resque) vs 80 workers/server might account for the rest of the difference.

#### Troubleshooting tip
One feature that I found useful is that if you send a ```TTIN``` signal to a Sidekiq worker, each thread will print its backtrace in the log.

So, if you tail the Sidekiq log, it'll tell you the PID of the process:
```2014-12-10_22:28:55.22696 2014-12-10T22:28:55.226Z 31340 TID-u2nf0 [WorkerName] JID-f563b9dd180c5e0eb4f44186 INFO: done: 0.025 sec```
In the above example, it is ```31340```. Then you would send the signal:
```kill -TTIN 31340```.

Then tail the log again and you should see backtraces for all of the threads in the worker.
