Resque
======

Resque (pronounced like "rescue") is a Redis-backed library for creating
background jobs, placing those jobs on multiple queues, and processing
them later.

  - [![Code Climate](https://codeclimate.com/github/resque/resque.png)](https://codeclimate.com/github/resque/resque)
  - [![Build Status](https://travis-ci.org/resque/resque.png?branch=master)](https://travis-ci.org/resque/resque)
  - [![Coverage Status](https://coveralls.io/repos/resque/resque/badge.png?branch=master)](https://coveralls.io/r/resque/resque)


## Installation

To install Resque, add the gem to your Gemfile:

```
gem "resque", "~> 2.0.0", github: "resque/resque"
```

Then run `bundle`. If you're not using Bundler, just `gem install resque`.
You'll also need Redis installed. On OS X this can be installed with the 
Homebrew backage manager: `brew install redis`.

### Requirements

Resque is used by a large number of people, across a diverse set of codebases.
There is no official requirement other than Ruby 1.9.3 or newer. We of course
recommend Ruby 2.0.0, but test against many Rubies, as you can see from our
[.travis.yml](https://github.com/resque/resque/blob/master/.travis.yml).

We would love to support non-MRI Rubies, but they may have bugs. We would love
some contributions to clear up failures on these Rubies, but they are set to
allow failure in our CI.

We officially support Rails 2.3.x and newer, though we recommend that you're on
Rails 3.2 or 4.

## Usage

After adding Resque to your Gemfile, you'll need to setup jobs and add them 
to the queue. A basic example can be found in the `examples/simple` directory.

A Redis server must be running, which can be run locally with `redis-server`.
Resque assumes by default that the server is hosted at `localhost`.

Jobs are defined as modules and in Rails often kept in separate ruby files in 
`app/jobs`.

Each job will need a name; a queue name, and a module method of `self.perform` 
which can optionally take arguments.

```ruby
module JobName
  @queue = :default

  def self.perform
    sleep 1
    puts "This job just waits one second. Code here is executed outside of Rails"
    puts "If the worker has access to the environment variables,"
    puts "it can access anything a Rake task could."
  end
end
```

Then anywhere in your Rails code you can queue the job to be excuted later:

```ruby
  Resque.enqueue(Job)
```

If the job needed arguments, the `self.perform` method would look like 
`self.perform(arg1, arg2)` and when queued the `enqueue` method would 
have two extra arguments and would look like `Resque.enqueue(Job, one, two)`.

This unit of work will be stored in Redis. We can spin up a worker from your 
terminal to grab some work off of the queue and do the work:

```
$ resque work
```

This process polls Redis, grabs any jobs that need to be done, and then processes
them.

### Jobs

What deserves to be a background job? Anything that's not always super fast.
There are tons of stuff that an application does that falls under the 'not
always fast' category:

* Warming caches
* Counting disk usage
* Building tarballs
* Firing off web hooks
* Creating events in the db and pre-caching them
* Building graphs
* Deleting users

And it's not always web stuff, either. A command-line client application that
does web scraping and crawling is a great use of jobs, too.

### Workers

You can start up a worker with 

```
$ resque work
```

This will basically loop over and over, polling for jobs and doing the work.
You can have workers work on a specific queue with the `--queue` option:

```
$ resque work --queues=high,low
$ resque work --queues=high
```

This starts two workers working on the `high` queue, one of which also polls
the `low` queue.

You can control the length of the poll with `interval`:

```
$ resque work --interval=1
```

Now workers will check for a new job every second. The default is 5.

Outside of Rails, you'll probably need to tell the resque worker where the
worker modules are manually, which can be done with the `-r` flag

```
$ resque work -r ./my_job.rb
```

Resque workers respond to a few different signals:

    QUIT - Wait for child to finish processing then exit
    TERM / INT - Immediately kill child then exit
    USR1 - Immediately kill child but don't exit
    USR2 - Don't start to process any new jobs
    CONT - Start to process new jobs again after a USR2

### In Redis

Jobs are persisted in Redis via JSON serialization. Basically, the job looks
like this:

```json
{
    "class": "Email",
    "vars": {
      "to": "foo@example.com",
      "from": "steve@example.com"
    }
}
```

When Resque fetches this job from Redis, it will do something like this:

```ruby
json = fetch_json_from_redis

job = json["class"].constantize.new
json["vars"].each {|k, v| job.instance_variable_set("@#{k}", v) }
job.work
```

### Failure

When jobs fail, the failure is stored in Redis, too, so you can check them out
and possibly re-queue them. You can inspect these by running the following 
from a Ruby console.

To view the number of failed jobs:

```ruby
Resque::Failure.count
```

To inspect the individual failed jobs:

```ruby
Resque::Failure.all(0,20).each { |job|
   puts "#{job["exception"]}  #{job["backtrace"]}"
}
```

To retry all failed jobs:

```ruby
(Resque::Failure.count-1).downto(0).each { |i| Resque::Failure.requeue(i) }
```

## Configuration

You can configure Resque via a `.resque` file in the root of your project:

```
--queue=*
--interval=1
```

These act just like you passed them in to `resque work`.

You can also configure a [Procfile](https://devcenter.heroku.com/articles/procfile), 
which is commonly used with a service such as Heroku, or locally with 
[Foreman](https://github.com/ddollar/foreman):

`worker: env QUEUE=* bundle exec rake resque:work`


## Hooks and Plugins

Separate documentation files are available on [hooks](https://github.com/resque/resque/blob/master/docs/HOOKS.md) and [plugins](https://github.com/resque/resque/blob/master/docs/PLUGINS.md).


### A note about branches

This branch is the master branch, which contains work towards Resque 2.0. If
you're currently using Resque, you'll want to check out [the 1-x-stable
branch](https://github.com/resque/resque/tree/1-x-stable), and particularly
[its README](https://github.com/resque/resque/blob/1-x-stable/README.markdown),
which is more accurate for the code you're running in production.

Also, this README is written first, so lots of things in here may not work the
exact way that they say they might here. Yay 2.0!

### Backwards Compatibility

Resque uses [SemVer](http://semver.org/), and takes it seriously. If you find
an interface regression, please [file an issue](https://github.com/resque/resque/issues)
so that we can address it.

If you have previously used Resque 1.23, the transition to 2.0 shouldn't be
too painful: we've tried to upgrade _interfaces_ but leave _semantics_ largely
in place. Check out
[UPGRADING.md](https://github.com/resque/resque/blob/master/UPGRADING.md) for
detailed examples of what needs to be done.

## Contributing

Please see [CONTRIBUTING.md](https://github.com/resque/resque/blob/master/CONTRIBUTING.md) 
for information on running tests, legacy code and contributing to this project.

## Anything we missed?

If there's anything at all that you need or want to know, please email either
[the mailing list](mailto:resque@librelist.com) or [Steve
Klabnik](mailto:steve@steveklabnik.com) and we'll get you sorted.
