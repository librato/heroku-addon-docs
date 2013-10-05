[Librato](http://addons.heroku.com/librato) is an [add-on](http://addons.heroku.com) for collecting, understanding, and acting on the metrics that matter to you.

![Librato Dashboard](https://librato_images.s3.amazonaws.com/heroku-docs/librato-rails-dashboard.png "Librato Dashboard")

Librato is a complete solution for monitoring and analyzing the metrics that impact your business at all levels of the stack. It provides everything you need to visualize, analyze, and actively alert on the metrics that matter to you. Librato integrates seamlessly with Heroku to provide you out-of-the-box detailed information about your application's performance, per-dyno system resources, and postgres database. But that's just the beginning!

With drop-in support for [Rails 3.x][rails-gem]/[Rack][rack-gem],
[JVM-based applications][coda-backend], and [other
languages][lang-bindings] you'll additionally have custom application
metrics streaming into Librato in minutes. From there you can build
tailored charts/dashboards, annotate them with one-time events, and set threshold-based alerts.  Collaboration is supported through multi-user access, private dashboard links, PNG chart snapshots, and seamless integration with popular third-party services like PagerDuty, Campfire, and HipChat.  Additionally Librato is first-and-foremost a platform whose complete capabilities are programmatically accessible via a [RESTful API][api-docs] with bindings available for [a growing host of languages][lang-bindings] including Ruby, Python, Java, Go, Clojure, Node.js, etc.

## Provisioning the add-on

Librato can be attached to a Heroku application via the CLI:

<div class="callout" markdown="1">
A list of all plans available can be found [here](http://addons.heroku.com/librato).
</div>

    :::term
    $ heroku addons:add librato
    -----> Adding librato to sharp-mountain-4005... done, v18 (free)

After successful provisioning, the native log-based metrics described
below will automatically begin flowing into your Librato account and
into a dashboard we've pre-configured to provide a basic overview of
your application and database performance. It may take 2-3 minutes for
data to begin showing up. You can go to the Librato interface now to
check things out or read on to learn more!

## Native log-based metrics

After provisioning the add-on, Librato immediately begins collecting
native Heroku metrics pertaining directly from your application's log
stream.

[Router metrics][log-router-metrics] are present in every Heroku
application log stream and provide detailed information (including
timing) about every request serviced by your application. As
these details are sourced directly from the Heroku routing layer itself,
it's the only true measure of performance as experienced by your
customers, accounting for any delays introduced by Heroku in addition to
your application's processing. Librato calculates a rich set of summary
statistics (described below in custom distributions) including median, 90th
percentile, and 99th percentile latency that provide
unprecedented insight into your application's performance. The
metrics (described in the linked Heroku router docs) available in
Librato currently are `router.connect`, `router.service`, and
`router.bytes`.

[Postgres metrics][log-postgres-metrics] are present in the log stream
of any Heroku application with a provisioned Heroku Postgres database.
These metrics are summary statistics gathered by the Heroku Postgres
service and are reported directly into Librato under their given names.

[Per-dyno runtime metrics][log-runtime-metrics] are available to any
Heroku application, but must be enabled by the user, as they are
disabled by default. They provide insight into both memory usage and
CPU load of each dyno and are reported under the dyno's logical role
e.g. `web.1` or `worker.2`. You can enable them with the following commands:

    :::term
    $ heroku labs:enable log-runtime-metrics
    Enabling log-runtime-metrics for ... done
    $ heroku restart

Once enabled your per-dyno runtime metrics will automatically begin
showing up in your Librato account. These metrics are summary statistics
gathered by the Heroku Runtime layer
and are reported directly into Librato under their given names.

## Custom log-based metrics

In addition to seamlessly extracting native metrics provided by
Heroku, Librato's log integration enable you to publish your own custom
metrics directly to Librato through your Heroku log stream. Simply write your measurements to standard
out (stdout) using one of the formats specified below and Librato will
automatically detect and collect your custom metrics. The examples below
are in Ruby syntax but can be replicated in any language that permits
writing to standard out.

### Custom counting
*Count* log lines are used to submit increments to Librato. You can
 submit increments as frequently as desired and every minute the current
total will be flushed to Librato and reset to zero. For example the following
combination would result in a value of 5 being recorded for the
enclosing minute in a Librato metric named `user.clicks`:

    :::ruby
    $stdout.puts("count#user.clicks=1")
    $stdout.puts("count#user.clicks=1")
    $stdout.puts("count#user.clicks=3")

### Custom distributions
*Measure* log lines are used to submit individual measurements that
 comprise a statistical distribution. The most common use case are *timings* i.e. latency
 measurements, but it can also be used to represent non-temporal
distributions such as counts. You can
 submit as many measures as you'd like (typically they are submitted
per-request) and every minute Librato will calculate/record a complete set of summary
statistics over the measures submitted in that interval:

* **min** - The smallest value.
* **max** - The largest value.
* **average** - The average of all values.
* **median** - The value that falls precisely in the middle of all values. Considered a good indicator of *typical* application performance.
* **95th percentile** - The value that is larger than 95% of all values.
* **99th percentile** - The value that is larger than 99% of all values.
* **sum** - The total of all values.
* **count** - The total number of values submitted.

Measures are submitted in a similar fashion to counts:

    :::ruby
    $stdout.puts("measure#database.query=200ms")

Note that measures create 4 separate metrics in Librato based on the
submitted measure name. Using the example above, you would find:

* `database.query` - A *complex* metric that contains the
  min/mean/max/sum/count values. You can switch between them using Librato's [summary statistics][librato-summary-statistics] support.
* `database.query.median` - The median value.
* `database.query.perc95` - The 95th percentile value.
* `database.query.perc99` - The 99th percentile value.

### Custom sample metrics
*Sample* metrics are used to convey simple key/numerical value pairs when you are already calculating some kind of summary statistic in your app and
merely need a simple transport mechanism to Librato.  Typically you would submit sample metrics on some periodic tick and set
said period on the metric in Librato. This example would create a metric
in Librato with the name `database.size`:

    :::ruby
    $stdout.puts("sample#database.size=40.9MB")

## Configuring additional integrations
Once Librato has been added, settings for `LIBRATO_USER` and
`LIBRATO_TOKEN` will be available in the app configuration and will
contain the credentials needed to authenticate any integrations to the [Librato API][api-docs]. This can be confirmed using the `heroku config:get` command.

    :::term
    $ heroku config:get LIBRATO_USER
    app123@heroku.com

After installing Librato you will need to explicitly set a value for `LIBRATO_SOURCE` in the app configuration. `LIBRATO_SOURCE` informs the Librato service that the metrics coming from each of your dynos belong to the same application.

    :::term
    $ heroku config:set LIBRATO_SOURCE=myappname

The value of `LIBRATO_SOURCE` must be composed of characters in the set `A-Za-z0-9.:-_` and no more than 255 characters long. You should use a permanent name, as changing it in the future will cause your historical metrics to become disjoint.

## Using with Ruby

Ruby is currently supported as either Rails 3 or Rack applications. For other Ruby environments please contact us through one of the methods described below in the *Support* section.

### Rails 3 Installation

Verify that the `LIBRATO_USER` and `LIBRATO_SOURCE` variables are set. Ruby-on-Rails applications need to add the following entry into their `Gemfile` specifying the Librato client library.

    :::ruby
    gem 'librato-rails'

Then update application dependencies with bundler.

    :::term
    $ bundle install

Finally re-deploy your application.

    :::term
    $ git commit -a -m "add librato-rails instrumentation"
    $ git push heroku master

The source code and a detailed `README` for `librato-rails` are [available on GitHub][rails-gem].

#### Automatic Rails Instrumentation

After installing the `librato-rails` gem and deploying your app you will see a number of metrics appear automatically in your Librato account.  These are powered by [ActiveSupport::Notifications][asn] and track request performance, sql queries, mail handling, etc.

Built-in performance metric names will start with either `rack` or `rails`, depending on the level they are being sampled from. For example: `rails.request.total` is the total number of requests rails has received each minute.

Support for optionally disabling automatic instrumentation in `librato-rails` is currently under development. In the interim as a workaround you can instead use `librato-rack` (installation described below) to access the same custom instrumentation primitives without any automatic Rails metrics.

### Rack Installation

Verify that the `LIBRATO_USER` and `LIBRATO_SOURCE` variables are set. Rack applications need to add the following entry into their `Gemfile` specifying the Librato client library.

    :::ruby
    gem 'librato-rack'

Then update application dependencies with bundler.

    :::term
    $ bundle install

Then in your rackup file (or equivalent), require and add the middleware:

    :::ruby
    require 'librato-rack'
    use Librato::Rack

Finally re-deploy your application.

    :::term
    $ git commit -a -m "add librato-rack instrumentation"
    $ git push heroku master

The source code and a detailed `README` for `librato-rack` are [available on GitHub][rack-gem].

### Custom Instrumentation

Once you've installed Librato in either your Rails 3 or Rack application, you can immediately and easily start adding your own custom instrumentation. There are four simple instrumentation primitives available:

#### increment

Use for tracking a running total of something _across_ requests, examples:

    # increment the 'sales_completed' metric by one
    Librato.increment 'sales_completed'

    # increment by five
    Librato.increment 'items_purchased', :by => 5

    # increment with a custom source
    Librato.increment 'user.purchases', :source => user.id

Other things you might track this way: user signups, requests of a certain type or to a certain route, total jobs queued or processed, emails sent or received.

Note that `increment` is primarily used for tracking the rate of occurrence of some event. Given this `increment` metrics are _continuous by default_ i.e. after being called on a metric once they will report on every interval, reporting zeros for any interval when increment was not called on the metric.

Especially with custom sources you may want the opposite behavior, i.e. reporting a measurement only during intervals where `increment` was called on the metric:

    # report a value for 'user.uploaded_file' only during non-zero intervals
    Librato.increment 'user.uploaded_file', :source => user.id, :sporadic => true

#### measure

Use when you want to track an average value _per_-request. Examples:

    Librato.measure 'user.social_graph.nodes', 212

#### timing

Like `Librato.measure` this is per-request, but specialized for timing information:

    Librato.timing 'twitter.lookup.time', 21.2

The block form auto-submits the time it took for its contents to execute as the measurement value:

    Librato.timing 'twitter.lookup.time' do
      @twitter = Twitter.lookup(user)
    end

#### group

There is also a grouping helper, to make managing nested metrics easier. So this:

    Librato.measure 'memcached.gets', 20
    Librato.measure 'memcached.sets', 2
    Librato.measure 'memcached.hits', 18

Can also be written as:

    Librato.group 'memcached' do |g|
      g.measure 'gets', 20
      g.measure 'sets', 2
      g.measure 'hits', 18
    end

Symbols can be used interchangably with strings for metric names.

### Troubleshooting with Ruby

Check the logs for messages such as this

    :::term
    [librato-rails] halting: source must be provided in configuration.

Both the `librato-rails` and `librato-rack` gems support multiple logging levels that are useful in diagnosing any issues with reporting metrics to Librato. These are controlled by the `LIBRATO_LOG_LEVEL` configuration.

    :::term
    $ heroku config:set LIBRATO_LOG_LEVEL=debug

Set your log level to `debug` to log detailed information about the settings the gem is seeing at startup and when it is submitting metrics back to the Librato service.

If you are having an issue with a specific metric, setting a log level of `trace` additionally logs the exact measurements being sent along with lots of other information about instrumentation execution.

Neither of these modes are recommended long-term in production as they will add quite a bit of volume to your log stream and will slow operation somewhat.  Note that submission I/O is non-blocking, submission times are total time - your process will continue to handle requests during submissions.

## Librato Interface

<div class="callout" markdown="1">
For more information on the features available within the Librato interface please see the [Librato knowledgebase](http://support.metrics.librato.com/knowledgebase).
</div>

The Librato interface allows you to build custom dashboards, set threhold-based alerts, rapidly detect and diagnose performance regressions in your production infrastructure, gain a deeper, shared understanding of your business across your team, and so much more!.

![Librato Dashboard](https://librato_images.s3.amazonaws.com/heroku-docs/librato-rails-dashboard.png "Librato Dashboard")

The interface can be accessed via the CLI:

    :::term
    $ heroku addons:open librato
    Opening librato for sharp-mountain-4005...

or by visiting the [Heroku apps web interface](http://heroku.com/myapps) and selecting the application in question. Select Librato from the Add-ons menu.

![Librato Add-ons Dropdown](https://librato_images.s3.amazonaws.com/heroku-docs/addon-menu.png "Librato Add-ons Dropdown")

## General Troubleshooting

It may take 2-3 minutes for the first results to show up in your Metrics account after you have deployed your app and the first request has been received.

Note that if Heroku idles your application, measurements will not be sent until it receives another request and is restarted. If you see intermittent gaps in your measurements during periods of low traffic this is the most likely cause.

For troubleshooting instructions more specific to your particular platform, please see our polyglot documentation provided above. The documentation for each supported platform ends with a troubleshooting subsection titled in the form *Troubleshooting with ...*.

## Migrating between plans

As long as the plan you are migrating to includes enough allocated measurements for your usage, you can migrate between plans at any time without any interruption to your metrics.

Use the `heroku addons:upgrade` command to migrate to a new plan.

    :::term
    $ heroku addons:upgrade librato:gold-10
    -----> Upgrading librato:gold-10 to sharp-mountain-4005... done, v18 ($49/mo)
           Your plan has been updated to: librato:gold-10

## Removing the add-on

Librato can be removed via the  CLI.

<div class="warning" markdown="1">This will destroy all associated data and cannot be undone!</div>

    :::term
    $ heroku addons:remove librato
    -----> Removing librato from sharp-mountain-4005... done, v20 (free)

Before removing Librato data can be exported through the [Librato
API][api-docs].

## Support

All Librato support and runtime issues should be submitted via one of the [Heroku Support channels](support-channels). Any non-support related issues or product feedback for Librato is welcome via [email](mailto:support@librato.com), [live chat](http://chat.librato.com), or the [support forum](http://support.metrics.librato.com).

[api-docs]: http://dev.librato.com/v1/metrics
[lang-bindings]: http://support.metrics.librato.com/knowledgebase/articles/122262-language-bindings
[rails-gem]: https://github.com/librato/librato-rails
[rack-gem]: https://github.com/librato/librato-rack
[coda-metrics]: http://metrics.codahale.com/
[coda-backend]: https://github.com/librato/metrics-librato
[asn]: http://api.rubyonrails.org/classes/ActiveSupport/Notifications.html
[log-router-metrics]: https://devcenter.heroku.com/articles/http-routing#heroku-router-log-format
[log-runtime-metrics]: https://devcenter.heroku.com/articles/log-runtime-metrics
[log-postgres-metrics]: https://devcenter.heroku.com/articles/heroku-postgres-metrics-logs
[librato-summary-statistics]: http://blog.librato.com/posts/2013/6/3/fine-grained-access-to-summary-statistics
