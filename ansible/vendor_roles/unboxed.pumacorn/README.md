## What is ansible-pumacorn? [![Build Status](https://secure.travis-ci.org/nickjj/ansible-pumacorn.png)](http://travis-ci.org/nickjj/ansible-pumacorn)

It is an [ansible](http://www.ansible.com/home) role to manage either a puma or unicorn rails process.

### What problem does it solve and why is it useful?

It is usually a good idea to create a service out of your rails application. This allows you to easily manage it in the future without having to worry about things like sourcing environments because the service script would handle all of the dirty work for you.

It gives you access to these commands after it's setup:

- `sudo service yourapp start`
- `sudo service yourapp stop`
- `sudo service yourapp restart`
- `sudo service yourapp reload`
- `sudo service yourapp status`

#### What's the difference between restart and reload?

**Restart** will have a small amount of downtime because it will kill off the master process for whatever web server you're using and fully restart it.

**Reload** will apply new code changes without downtime and the web server will cycle its workers automatically. Keep in mind if you do a reload then you will have (2) versions of your application running at the same time. If your application code is not setup to work with both versions then you will need to do a restart instead.

## Role variables

Below is a list of default values along with a description of what they do.

```
# Are you using puma or unicorn? This would be the binary name.
pumacorn_server: puma

# The path to the config file.
pumacorn_config_path: "{{ rails_deploy_path }}/config/{{ pumacorn_server }}.rb"

# Should a restart be forced?
pumacorn_force_restart: false
```

## Example playbook

For the sake of this example let's assume you have a group called **app** and you have a typical `site.yml` file.

To use this role edit your `site.yml` file to look something like this:

```
---
- name: ensure app servers are configured
- hosts: app

  roles:
    # Insert other roles here to provision the server before managing the rails process.
    - { role: nickjj.pumacorn, tags: pumacorn }
```

Let's say you want to edit a few defaults, you can do this by opening or creating `group_vars/all.yml` which is located relative to your `inventory` directory and then making it look something like this:

```
---
pumacorn_server: unicorn
```

#### Does it automatically restart a crashed process?

Nope, init.d scripts don't do that. What you get is a service that starts on bootup, restarts when a migration was ran and reloads with no downtime when no migrations were ran. If you want the process to be monitored you will want to hook up something like [monit](http://mmonit.com/monit), [runit](http://smarden.org/runit) or [god](http://godrb.com/).
 
I personally prefer monit because the API for adding new things to monitor is very nice and it optionally exposes a web front end. If you're interested here's a link to my [nickjj.monit role](https://github.com/nickjj/ansible-monit).

## Installation

`$ ansible-galaxy install nickjj.pumacorn`

## Dependencies

It depends on the [nickjj.rails](https://github.com/nickjj/ansible-rails) role. This role must be ran before pumacorn.

## Requirements

Tested on ubuntu 12.04 LTS but it should work on other versions that are similar.

## Ansible galaxy

You can find it on the official [ansible galaxy](https://galaxy.ansible.com/list#/roles/946) if you want to rate it.

## Example puma config

To do phased restarts with puma you must have at least 2 workers running and you cannot pre-load your application. Below is a bare bones puma config file that you can use. If you happen to use [orats](https://github.com/nickjj/orats) to generate your projects then all of this is taken care of for you.

`config/puma.rb`:

```
environment ENV['RAILS_ENV']

threads 0,16
workers 2

# It would be a good idea to abstract away the `/full/path/to/your/project` path to
# an ENV variable so that you don't have to mess around with your
# production deploy path directly in your config file.
pidfile "/full/path/to/your/project/tmp/puma.pid"

if ENV['RAILS_ENV'] == 'production'
  bind "unix:///full/path/to/your/project/tmp/puma.sock"
else
  port '3000'
end

prune_bundler

restart_command 'bundle exec bin/puma'

on_worker_boot do
  require 'active_record'
  config_path = File.expand_path('../database.yml', __FILE__)

  ActiveRecord::Base.connection.disconnect! rescue ActiveRecord::ConnectionNotEstablished
  ActiveRecord::Base.establish_connection(ENV['DATABASE_URL'] || YAML.load_file(config_path)[ENV['RAILS_ENV']])
end
```

## Example unicorn config

I don't use unicorn but I borrowed gitlab's unicorn config from [gitlabhq](https://github.com/gitlabhq/gitlabhq/blob/master/config/unicorn.rb.example). The only exception is your pid file must be written to `{{ rails_deploy_path}}/tmp/unicorn.pid`. If you go to about line 15 in the config file below you will see the `pid` command that you need to adjust.

`config/unicorn.rb`:

```
# Sample verbose configuration file for Unicorn (not Rack)
#
# This configuration file documents many features of Unicorn
# that may not be needed for some applications. See
# http://unicorn.bogomips.org/examples/unicorn.conf.minimal.rb
# for a much simpler configuration file.
#
# See http://unicorn.bogomips.org/Unicorn/Configurator.html for complete
# documentation.

# WARNING: See config/application.rb under "Relative url support" for the list of
# other files that need to be changed for relative url support
#
# ENV['RAILS_RELATIVE_URL_ROOT'] = "/gitlab"

# replace this with the same value as {{ rails_deploy_path }}
pid "/full/path/to/your/project/tmp/unicorn.pid"

# Use at least one worker per core if you're on a dedicated server,
# more will usually help for _short_ waits on databases/caches.
worker_processes 2

# Since Unicorn is never exposed to outside clients, it does not need to
# run on the standard HTTP port (80), there is no reason to start Unicorn
# as root unless it's from system init scripts.
# If running the master process as root and the workers as an unprivileged
# user, do this to switch euid/egid in the workers (also chowns logs):
# user "unprivileged_user", "unprivileged_group"

# Help ensure your application will always spawn in the symlinked
# "current" directory that Capistrano sets up.
working_directory "/home/git/gitlab" # available in 0.94.0+

# listen on both a Unix domain socket and a TCP port,
# we use a shorter backlog for quicker failover when busy
listen "/home/git/gitlab/tmp/sockets/gitlab.socket", :backlog => 64
listen "127.0.0.1:8080", :tcp_nopush => true

# nuke workers after 30 seconds instead of 60 seconds (the default)
timeout 30

# By default, the Unicorn logger will write to stderr.
# Additionally, some applications/frameworks log to stderr or stdout,
# so prevent them from going to /dev/null when daemonized here:
stderr_path "/home/git/gitlab/log/unicorn.stderr.log"
stdout_path "/home/git/gitlab/log/unicorn.stdout.log"

# combine Ruby 2.0.0dev or REE with "preload_app true" for memory savings
# http://rubyenterpriseedition.com/faq.html#adapt_apps_for_cow
preload_app true
GC.respond_to?(:copy_on_write_friendly=) and
  GC.copy_on_write_friendly = true

# Enable this flag to have unicorn test client connections by writing the
# beginning of the HTTP headers before calling the application.  This
# prevents calling the application for connections that have disconnected
# while queued.  This is only guaranteed to detect clients on the same
# host unicorn runs on, and unlikely to detect disconnects even on a
# fast LAN.
check_client_connection false

before_fork do |server, worker|
  # the following is highly recomended for Rails + "preload_app true"
  # as there's no need for the master process to hold a connection
  defined?(ActiveRecord::Base) and
    ActiveRecord::Base.connection.disconnect!

  # The following is only recommended for memory/DB-constrained
  # installations.  It is not needed if your system can house
  # twice as many worker_processes as you have configured.
  #
  # This allows a new master process to incrementally
  # phase out the old master process with SIGTTOU to avoid a
  # thundering herd (especially in the "preload_app false" case)
  # when doing a transparent upgrade.  The last worker spawned
  # will then kill off the old master process with a SIGQUIT.
  old_pid = "#{server.config[:pid]}.oldbin"
  if old_pid != server.pid
    begin
      sig = (worker.nr + 1) >= server.worker_processes ? :QUIT : :TTOU
      Process.kill(sig, File.read(old_pid).to_i)
    rescue Errno::ENOENT, Errno::ESRCH
    end
  end
  #
  # Throttle the master from forking too quickly by sleeping.  Due
  # to the implementation of standard Unix signal handlers, this
  # helps (but does not completely) prevent identical, repeated signals
  # from being lost when the receiving process is busy.
  # sleep 1
end

after_fork do |server, worker|
  # per-process listener ports for debugging/admin/migrations
  # addr = "127.0.0.1:#{9293 + worker.nr}"
  # server.listen(addr, :tries => -1, :delay => 5, :tcp_nopush => true)

  # the following is *required* for Rails + "preload_app true",
  defined?(ActiveRecord::Base) and
    ActiveRecord::Base.establish_connection

  # if preload_app is true, then you may also want to check and
  # restart any other shared sockets/descriptors such as Memcached,
  # and Redis.  TokyoCabinet file handles are safe to reuse
  # between any number of forked children (assuming your kernel
  # correctly implements pread()/pwrite() system calls)
end
```

## License

MIT