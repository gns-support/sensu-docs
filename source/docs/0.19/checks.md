---
version: 0.19
category: "Reference Docs"
title: "Checks"
next:
  url: "handlers"
  text: "Handlers"
---

# Overview

This reference document provides information to help you:

- Understand what a Sensu check is
- Understand how a Sensu check works
- Write a Sensu check definition
- Use built-in Sensu check definition attributes
- Use custom Sensu check definition attributes

# What are Sensu checks? {#what-are-sensu-checks}

Sensu checks allow you to monitor services or measure resources, they are executed on machines running the Sensu client. Checks are essentially commands (or scripts) that output data to `STDOUT` or `STDERR` and produce an exit status code to indicate a state. Common exit status codes used are `0` for `OK`, `1` for `WARNING`, `2` for `CRITICAL`, and `3` or greater to indicate `UNKNOWN` or `CUSTOM`. Sensu checks use the same specification as Nagios, therefore, Nagios check plugins may be used with Sensu.

## Example check plugin {#example-check-plugin}

The following is an example Sensu check plugin, a script located at `/etc/sensu/plugins/check-chef-client.rb`. This check plugin uses the running process list to determine if the Chef client process is running. This check plugin is written in Ruby, but Sensu plugins can be written in any language, e.g. Python, shell, etc.

~~~ ruby
#!/usr/bin/env ruby

# get the current list of processes
processes = `ps aux`

# determine if the chef-client process is running
running = processes.lines.detect do |process|
  process.include?('chef-client')
end

# return appropriate check output and exit status code
if running
  puts 'OK - Chef client process is running'
  exit 0
else
  puts 'WARNING - Chef client process is NOT running'
  exit 1
end
~~~

# Check definition

A Sensu check definition is a JSON configuration file describing a Sensu check. A definition declares how a Sensu check is executed:

- The command to run (e.g. script)
- How frequently it should be executed (interval)
- Where it should be executed (which machines)

## Example check definition {#example-check-definition}

The following is an example Sensu check definition, a JSON configuration file located at `/etc/sensu/conf.d/check_chef_client.json`. This check definition uses the [example check plugin](#example-check-plugin) above, to determine if the Chef client process is running. The check is named `chef_client` and it runs `/etc/sensu/plugins/check-chef-client.rb` on Sensu clients with the `production` subscription, every `60` seconds (interval).

~~~ json
{
  "checks": {
    "chef_client": {
      "command": "/etc/sensu/plugins/check-chef-client.rb",
      "subscribers": [
        "production"
      ],
      "interval": 60
    }
  }
}
~~~

# Anatomy of a check definition

### Name

Each check definition has a unique check name, used for the definition key. Every check definition is within the `"checks": {}` definition scope.

- A unique string used to name/identify the check
- Cannot contain special characters or spaces
- Validated with `/^[\w\.-]+$/`
- e.g. `"chef_client": {}`

### Definition attributes

type
: description
  : The check type, either `standard` or `metric`. Setting type to `metric` will cause OK (exit 0) check results to create events.
: required
  : false
: type
  : String
: allowed values
  : `standard`, `metric`
: default
  : `standard`
: example
  : ~~~ shell
    "type": "metric"
    ~~~

command
: description
  : The check command to be executed.
: required
  : true (unless `extension` is configured)
: type
  : String
: example
  : ~~~ shell
    "command": "/etc/sensu/plugins/check-chef-client.rb"
    ~~~

extension
: description
  : The name of a Sensu check extension to run instead of a command. This is an _advanced feature_ and is not commonly used.
: required
  : true (unless `command` is configured)
: type
  : String
: example
  : ~~~ shell
    "extension": "system_profile"
    ~~~

standalone
: description
  : If the check is scheduled by the local Sensu client instead of the Sensu server (standalone mode).
: required
  : false
: type
  : Boolean
: default
  : false
: example
  : ~~~ shell
    "standalone": true
    ~~~

subscribers
: description
  : An array of Sensu client subscriptions that check requests will be sent to. The array cannot be empty and its items must each be a string.
: required
  : true (unless `standalone` is `true`)
: type
  : Array
: example
  : ~~~ shell
    "subscribers": ["production"]
    ~~~

publish
: description
  : If check requests are published for the check.
: required
  : false
: type
  : Boolean
: default
  : true
: example
  : ~~~ shell
    "publish": false
    ~~~

interval
: description
  : The frequency in seconds the check is executed.
: required
  : true (unless `publish` is `false`)
: type
  : Integer
: example
  : ~~~ shell
    "interval": 60
    ~~~

timeout
: description
  : The check execution duration timeout in seconds (hard stop).
: required
  : false
: type
  : Integer
: example
  : ~~~ shell
    "timeout": 30
    ~~~

ttl
: description
  : The time to live (TTL) in seconds until check results are considered stale. If a client stops publishing results for the check, and the TTL expires, an event will be created for the client. The check `ttl` must be greater than the check `interval`, and should accomodate time for the check execution and result processing to complete. For example, if a check has an `interval` of `60` (seconds) and a `timeout` of `30` (seconds), an appropriate `ttl` would be a minimum of `90` (seconds).
: required
  : false
: type
  : Integer
: example
  : ~~~ shell
    "ttl": 100
    ~~~

handle
: description
  : If events created by the check should be handled.
: required
  : false
: type
  : Boolean
: default
  : true
: example
  : ~~~ shell
    "handle": "false"
    ~~~

handler
: description
  : The Sensu event handler (name) to use for events created by the check.
: required
  : false
: type
  : String
: example
  : ~~~ shell
    "handler": "pagerduty"
    ~~~

handlers
: description
  : An array of Sensu event handlers (names) to use for events created by the check. Each array item must be a string.
: required
  : false
: type
  : Array
: example
  : ~~~ shell
    "handlers": ["pagerduty", "email"]
    ~~~

low_flap_threshold
: description
  : The flap detection low threshold (% state change) for the check. Sensu uses the same [flap detection algorithm as Nagios](http://nagios.sourceforge.net/docs/3_0/flapping.html).
: required
  : false
: type
  : Integer
: example
  : ~~~ shell
    "low_flap_threshold": 20
    ~~~

high_flap_threshold
: description
  : The flap detection high threshold (% state change) for the check. Sensu uses the same [flap detection algorithm as Nagios](http://nagios.sourceforge.net/docs/3_0/flapping.html).
: required
  : true (if `low_flap_threshold` is configured)
: type
  : Integer
: example
  : ~~~ shell
    "high_flap_threshold": 60
    ~~~

source
: description
  : The check source, used to create a [JIT Sensu client](clients#jit-clients) for an external resource (e.g. a network switch).
: required
  : false
: type
  : String
: validated
  : `/^[\w\.-]+$/`
: example
  : ~~~ shell
    "source": "switch-dc-01"
    ~~~

subdue
: description
  : A set of attributes that determine when a check is subdued.
: required
  : false
: type
  : Hash
: example
  : ~~~ shell
    "subdue": {}
    ~~~

#### Subdue attributes

The following attributes are configured within the `"subdue": {}` check definition attribute scope.

at
: description
  : Where the check is subdued, either `publisher` or `handler`, at the check request publisher or event handler.
: required
  : false
: type
  : String
: allowed values
  : `publisher`, `handler`
: example
  : ~~~ shell
    "at": "handler"
    ~~~

days
: description
  : An array of days of the week the check is subdued. Each array item must be a string and a valid day of the week.
: required
  : false
: type
  : Array
: example
  : ~~~ shell
    "days": ["monday", "wednesday"]
    ~~~

begin
: description
  : Beginning of the time window when the check is subdued. Parsed by Ruby's `Time.parse()`. Time may include a time zone.
: required
  : true
: type
  : String
: example
  : ~~~ shell
    "begin": "5PM PST"
    ~~~

end
: description
  : End of the time window when the check is subdued. Parsed by Ruby's `Time.parse()`. Time may include a time zone.
: required
  : true
: type
  : String
: example
  : ~~~ shell
    "end": "9AM PST"
    ~~~

exceptions
: description
  : Subdue time window (`begin`, `end`) exceptions. An array of time window exceptions. Each array item must be a hash containing valid `begin` and `end` times.
: required
  : false
: type
  : Array
: example
  : ~~~ shell
    "exceptions": [{"begin": "8PM PST", "end": "10PM PST"}]
    ~~~

### Sensu plugin attributes

The [Sensu plugin project](https://github.com/sensu-plugins) provides a Ruby library ([sensu-plugin](https://github.com/sensu-plugins/sensu-plugin/)) to help Sensu plugin authors and offer users a variety of features. The sensu-plugin features make use of select custom check definition attributes. When using a [Sensu event handler](handlers) that makes use of the sensu-plugin library, you can configure the following attributes in any check definition.

occurrences
: description
  : The number of event occurrences that must occur before an event is handled for the check.
: required
  : false
: type
  : Integer
: default
  : `1`
: example
  : ~~~ shell
    "occurrences": 3
    ~~~

refresh
: description
  : Time in seconds until the event occurrence count is considered reset for the purpose of counting `occurrences`, to allow an event for the check to be handled again. For example, a check with a refresh of `1800` will have its events (recurrences) handled every 30 minutes, to remind users of the issue.
: required
  : false
: type
  : Integer
: default
  : `1800`
: example
  : ~~~ shell
    "refresh": 3600
    ~~~

dependencies
: description
  : An array of check dependencies. Events for the check will not be handled if events exist for one or more of the check dependencies. A check dependency can be a check executed by the same Sensu client (eg. `check_app`), or a client/check pair (eg.`db-01/check_mysql`).
: required
  : false
: type
  : Array
: example
  : ~~~ shell
    "dependencies": [
      "check_app",
      "db-01/check_mysql"
    ]
    ~~~
