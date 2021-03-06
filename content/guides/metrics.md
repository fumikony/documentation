---
title: Sending Metrics with DogStatsD
kind: guide
listorder: 7
js_dd_docs_methods:
  - metricsGuidePage
code_languages:
  - Python
  - Ruby
---
<!--
======================================================
OVERVIEW
======================================================
-->

<h3 id="overview">Overview</h3>

This guide explains how to send your application's custom metrics to Datadog.
Sending your application's custom metrics to Datadog will let you correlate
what's happening with your application, your users and your system.

Metrics are collected by sending them to StatsD, a small metrics aggregation
server that is bundled with the Datadog Agent. You can read about how it works <a
href="https://docs.datadoghq.com/guides/dogstatsd/">here</a>. If you want to dive into code right away,
read on.

In this tutorial, we'll cover some common instrumentation use cases, like:

- How to count web page views
- How to time database queries
- How to measure the amount of free memory

<!--
======================================================
SETUP
======================================================
-->

<h3 id="setup">Setup</h3>

First off, <a href="https://app.datadoghq.com/account/settings#agent">install</a>
the Datadog Agent (version 3 or greater), which
contains our StatsD server, and make sure it's running.

Next, let's set up a client library for your language.

{{< code-tabs section="setup" >}}

<div class="tab-content">

  <div class="tab-pane active fade in" id="setup-python">
First, install the module:

{{< highlight console >}}
$ pip install datadog
{{< /highlight >}}
And import it, so it's ready to use:

{{< highlight python >}}
from datadog import statsd
{{< /highlight >}}
  </div>

  <div class="tab-pane fade in" id="setup-ruby">
First, install the module:
{{< highlight console >}}
$ gem install dogstatsd-ruby
{{< /highlight >}}
And add it to your code:
{{< highlight ruby >}}
# Import the library
require 'datadog/statsd'

# Create a statsd client instance.
statsd = Datadog::Statsd.new
{{< /highlight >}}
  </div>
  <p>Now we're ready to roll.</p>
  <div class="alert info-block">
    This tutorial has examples for Python and Ruby, but check out the
    <a href="https://docs.datadoghq.com/libraries/">libraries page</a> if you use another language.
  </div>

</div>

<h4>Metric names</h4>
There are a few rules to stick to when naming metrics:
<ul>
<li>Metric names must start with a letter</li>
<li>Can only contain ascii alphanumerics, underscore and periods (other characters will get converted to underscores)</li>
<li>Should not exceed 200 characters (though less than 100 is genearlly preferred from a UI perspective)</li>
<li>Unicode is not supported</li>
<li>We recommend avoiding spaces</li>
</ul>
Metrics reported by the Agent are in a
pseudo-hierarchical dotted format (e.g. http.nginx.response_time). We say
pseudo-hierarchical because we're not actually enforcing a hierarchy or doing
anything with it, but we have aspirations to use it to infer things about
servers (e.g. "hey, I see hostA and hostB are reporting 'http.nginx.*', those
must be web frontends").


<!--
======================================================
COUNTERS
======================================================
-->


<h3 id="counters">Counters</h3>

Counters are used to (ahem) count things. Let's walk through a common example -
counting web page views. To achieve this, we'll increment a metric called
`web.page_views` each time our `render_page` function is called.


{{< code-tabs section="counters-metrics" >}}

<div class="tab-content">
  <div class="tab-pane active fade in" id="counters-metrics-python">
{{< highlight python >}}
def render_page():
    """ Render a web page. """
    statsd.increment('web.page_views')
    return 'Hello World!'
{{< /highlight >}}
  </div>
  <div class="tab-pane fade in" id="counters-metrics-ruby">
{{< highlight ruby >}}
def render_page()
  # Render a web page.
  statsd.increment('web.page_views')
  return 'Hello World!'
end
{{< /highlight >}}
  </div>
</div>

That's it. With this one line of code we can start graphing the data.
Here's an example:

{{< img src="guides/metrics/graph-guides-metrics-page-views.png" >}}

Note that StatsD counters are normalized over the flush interval to report
per-second units. In the graph above, the marker is reporting
35.33 web page views per second at ~15:24. In contrast, if one person visited
the webpage each second, the graph would be a flat line at y = 1. To increment or
measure values over time, please see <a href='#gauges'>gauges</a>.

We can also count by arbitrary numbers. Suppose we wanted to count the number
of bytes processed by a file uploading service. We'll increment a metric
called `file_service.bytes_uploaded` by the size of the file each time our
`upload_file` function is called:

{{< code-tabs section="counters-uploaded" >}}

<div class="tab-content">
  <div class="tab-pane active fade in" id="counters-uploaded-python">
{{< highlight python >}}
def upload_file(file):
    statsd.increment('file_service.bytes_uploaded', file.size())
    save_file(file)
    return 'File uploaded!'
{{< /highlight >}}
  </div>
  <div class="tab-pane fade in" id="counters-uploaded-ruby">
{{< highlight ruby >}}
def upload_file(file)
  statsd.count('file_service.bytes_uploaded', file.size())
  save_file(file)
  return 'File uploaded!'
end
{{< /highlight >}}
  </div>
</div>

Note that for counters coming from another source that are ever-increasing and never
reset -- for example, the number of queries from MySQL over time -- we track the
rate between flushed values. While there currently isn't an elegant solution to
get raw counts within Datadog, you may want to apply a function to
your series like cumulative sum or integral. There is more information on those
<a href="http://docs.datadoghq.com/graphing/#functions">here</a>.

<!--
======================================================
GAUGES
======================================================
-->

<h3 id="gauges">Gauges</h3>

Gauges measure the value of a particular thing over time. Suppose a developer
wanted to track the amount of free memory on a machine, we can periodically
sample that value as the metric `system.mem.free`:

{{< code-tabs section="guagesmeasure" >}}

<div class="tab-content">
  <div class="tab-pane active fade in" id="guagesmeasure-python">
{{< highlight python >}}
# Record the amount of free memory every ten seconds.
while True:
    statsd.gauge('system.mem.free', get_free_memory())
    time.sleep(10)
{{< /highlight >}}
  </div>
  <div class="tab-pane fade in" id="guagesmeasure-ruby">
{{< highlight ruby >}}
# Record the amount of free memory every ten seconds.
while true do
    statsd.gauge('system.mem.free', get_free_memory())
    sleep(10)
end
{{< /highlight >}}
  </div>
</div>

<!--
======================================================
HISTOGRAMS
======================================================
-->

<h3 id="histograms">Histograms</h3>

Histograms measure the statistical distribution of a set of values.
Suppose we wanted to measure the duration of a database query,
we can sample each query time with the metric `database.query.time`.

{{< code-tabs section="histograms" >}}

<div class="tab-content">
  <div class="tab-pane active fade in" id="histograms-python">
{{< highlight python >}}
# Track the run time of the database query.
start_time = time.time()
results = db.query()
duration = time.time() - start_time
statsd.histogram('database.query.time', duration)

# We can also use the `timed` decorator as a short-hand for timing functions.
@statsd.timed('database.query.time')
def get_data():
    return db.query()
{{< /highlight >}}
  </div>
  <div class="tab-pane fade in" id="histograms-ruby">
{{< highlight ruby >}}
start_time = Time.now
results = db.query()
duration = Time.now - start_time
statsd.histogram('database.query.time', duration)

# We can also use the `time` helper as a short-hand for timing blocks
# of code.
statsd.time('database.query.time') do
  return db.query()
end
{{< /highlight >}}
  </div>
</div>

The above instrumentation will produce the following metrics:

- `database.query.time.count` - the number of times this metric was sampled
- `database.query.time.avg` - the average time of the sampled values
- `database.query.time.median` - the median sampled value
- `database.query.time.max` - the maximum sampled value
- `database.query.time.95percentile` - the 95th percentile sampled value

These metrics give insight into how different each query time is. We can see
how long the query usually takes by graphing the `median`. We can see how long
most queries take by graphing the `95percentile`.

{{< img src="guides/metrics/graph-guides-metrics-query-times.png" >}}

For this toy example, let's say a query time of 1 second is acceptable.
Our median query time (graphed in purple) is usually less than 100
milliseconds, which is great. But unfortunately, our 95th percentile (graphed in
blue) has large spikes sometimes nearing three seconds, which is unacceptable.
This means most of our queries are running just fine, but our worst ones are
very bad. If the 95th percentile was close to the median, than we would know
that almost all of our queries are performing just fine.

<p class="alert alert-warning">
Histograms aren't just for measuring times. They can be used to measure the
distribution of any type of value, like the size of uploaded files or classroom
test scores.
</p>

<!--
======================================================
SERVICE CHECKS
======================================================
-->

<h3 id="service-checks">Service Checks</h3>

Service checks are used to send information about the status of a service.

{{< code-tabs section="service-checks" >}}

<div class="tab-content">
  <div class="tab-pane active fade in" id="service-checks-python">
{{< highlight python >}}
from datadog.api.constants import CheckStatus

# Report the status of an app.
name = 'web.app1'
status = CheckStatus.OK
message = 'Response: 200 OK'

statsd.service_check(check_name=name, status=status, message=message)
{{< /highlight >}}
  </div>
  <div class="tab-pane fade in" id="service-checks-ruby">
{{< highlight ruby >}}
# Report the status of an app.
name = 'web.app1'
status = Datadog::Statsd::OK
opts = {
  'message' => 'Response: 200 OK'
}

statsd.service_check(name, status, opts)
{{< /highlight >}}
  </div>
</div>

After a service check has been reported, you can use it to trigger a Custom Check monitor.

<!--
======================================================
SETS
======================================================
-->


<h3 id="sets">Sets</h3>

Sets are used to count the number of unique elements in a group. If you want to
track the number of unique visitors to your site, sets are a great way to do
that.

{{< code-tabs section="sets" >}}

<div class="tab-content">
  <div class="tab-pane active fade in" id="sets-python">
{{< highlight python >}}
def login(self, user_id):
    # Log the user in ...
    statsd.set('users.uniques', user_id)
{{< /highlight >}}
  </div>
  <div class="tab-pane fade in" id="sets-ruby">
{{< highlight ruby >}}
def login(self, user_id)
    # Log the user in ...
    statsd.set('users.uniques', user_id)
end
{{< /highlight >}}
  </div>
</div>


<!--
======================================================
TAGS
======================================================
-->

<h3 id="tags">Tags</h3>

Tags are a way of adding dimensions to metrics, so they can be sliced, diced,
aggregated and compared on the front end. Suppose we wanted to measure the
performance of two algorithms in the real world. We could sample one metric
`algorithm.run_time` and specify each version with a tag:

{{< code-tabs section="tags" >}}

<div class="tab-content">
  <div class="tab-pane active fade in" id="tags-python">
{{< highlight python >}}
@statsd.timed('algorithm.run_time', tags=['algorithm:one'])
def algorithm_one():
    # Do fancy things here ...

@statsd.timed('algorithm.run_time', tags=['algorithm:two'])
def algorithm_two():
    # Do fancy things here ...
{{< /highlight >}}
  </div>
  <div class="tab-pane fade in" id="tags-ruby">
{{< highlight ruby >}}
def algorithm_one()
  statsd.timed('algorithm.run_time', :tags => ['algorithm:one']) do
    # Do fancy things here ...
  end
end

def algorithm_two()
  statsd.timed('algorithm.run_time', :tags => ['algorithm:two']) do
    # Do different fancy things here ...
  end
end
{{< /highlight >}}
  </div>
</div>

On the front end, the metric instances can be aggregated as sums or averages,
or the minimum/maximum can be reported. The metrics can also be broken down by
each tag 'key' (in the key:value syntax). For instance, we could run a
query like this:

`avg:algorithm.run_time{*} by {algorithm}`

In this query, all instances of this metric (e.g. across all hosts, indicated by `*`) are averaged
(`avg`) and broken down by the tag key `algorithm`.


<p class="alert alert-warning">
We store one time series per host + metric + tag combination on our backend,
thus we cannot support infinitely bounded tags. Please don't include endlessly
growing tags in your metrics, like timestamps or user ids. Please limit each
metric to 1000 tags.
</p>


Tags must start with a letter, and after that may contain alphanumerics,
underscores, minuses, colons, periods and slashes. Other characters will get
converted to underscores. Tags can be up to 200 characters long and support
unicode. Tags will be converted to lowercase as well.

For optimal functionality, we recommend constructing tags that use the key:value
syntax. Examples of commonly used metric tag keys are `env`, `instance`, `name`, and `role`.
Note that `device`, `host`, and `source` are treated specially and cannot be specified
in the standard way. Check out some of our other docs for how to use these:

- <a href="http://docs.datadoghq.com/api/#metrics">metrics in the API</a>
- <a href="http://docs.datadoghq.com/api/#tags">tags in the API</a>
- <a href="http://docs.datadoghq.com/guides/agent_checks/">Agent Checks</a>
- <a href="http://docs.datadoghq.com/guides/logs/">log parsing</a>

<h3 id="sample-rates">Sample Rates</h3>

Each metric point is sent over UDP to the StatsD server. This can incur a lot
of overhead for performance intensive code paths. To work around this, StatsD
supports sample rates, which allows sending a metric a fraction of the time
and scaling up correctly on the server.

{{< code-tabs section="sample-rates" >}}

The following code will only send points half of the time:

<div class="tab-content">
  <div class="tab-pane active fade in" id="sample-rates-python">
{{< highlight python >}}
while True:
  do_something_intense()
  statsd.increment('loop.count', sample_rate=0.5)
{{< /highlight >}}
  </div>
  <div class="tab-pane fade in" id="sample-rates-ruby">
{{< highlight ruby >}}
while true do
  do_something_intense()
  statsd.increment('loop.count', :sample_rate => 0.5)
end
{{< /highlight >}}
  </div>
</div>


<h3 id="methods">Other Submission Methods</h3>

Using the StatsD server bundled with the Datadog Agent is the simplest
way of submitting metrics to Datadog, but it's not
the only one. Here are some other ways of getting your metrics data into
Datadog:

<ul>
  <li>
    Submit metrics directly to Datadog's <a href="https://docs.datadoghq.com/api/">HTTP API</a>
  </li>
  <li>
    Use Dropwizard's Java <a
    href="https://github.com/dropwizard/metrics">metrics</a> library, with the
<a href="https://github.com/coursera/metrics-datadog">metrics-datadog</a>
    backend (thanks to the good folks at
    <a href="http://www.vistarmedia.com/">Vistar Media</a>,
    <a href="https://www.coursera.org">Coursera</a>, and
    <a href="http://www.bazaarvoice.com">Bazaarvoice</a> for the great
    contributions).
  </li>
</ul>

<h3 id="custom">Seeing Your Custom Metrics</h3>

The quickest way to see your custom metric is to use the metrics explorer. You
can navigate to it by clicking the "Metrics" link in the top navigation bar.

Once you're at the metrics explorer you can type in the custom metric you set
in the "Graph:" field and it should autocomplete. If it doesn't autocomplete,
then it might mean that we haven't received data for that metric in the last
few hours.

You can also filter the metric by tag using the "Over:" field, or graph the
metric by tag group using the "One graph per:" field.

Note that the metrics explorer doesn't save any of these graphs. If you've
created some graphs that you'd like to save, you need to click one of the
save buttons at the bottom left, either saving to a new dashboard or to
an existing one.