# MQ Exporter for Collectd monitoring

This directory contains the code for a monitoring solution
that sends queue manager data to collectd system.
It also contains configuration files to run the monitor program

The monitor collects metrics published by an MQ V9 queue manager
or the MQ appliance. The monitor program prints
these metrics to stdout, in a format that is recognised by
collectd. Once processed by collectd, they may be forwarded to
storage systems, where
they can then be queried directly or used by other packages
such as Grafana.

You can see data such as disk or CPU usage, queue depths, and MQI call
counts.

Example Grafana dashboards are included, to show how queries might
be constructed, when collectd is configured to send the data to
an OpenTSDB or Graphite database. To use the dashboard,
create data sources in Grafana called "CollectD TSDB" or "CollectD Graphite"
that point at your database server, and then import the JSON file.

## Building
* This github repository contains both the monitoring program and
the ibmmq package that links to the core MQ application interface. It
also contains the mqmetric package used as a common component for
supporting alternative database collection protocols.

* Get the error logger package used by all of these monitors
using `go get -u github.com/Sirupsen/logrus`.

Run `go build -o <directory>/mq_coll cmd/mq_coll/*.go` to compile
the program and put it to a specific directory.

## Configuring MQ

No MQ configuration is required, as (unlike the other monitors in this
repository) the program does not run as an MQ Service.

The monitor always collects all of the available queue manager-wide metrics.
It can also be configured to collect statistics for specific sets of queues.
The sets of queues can be given either directly on the command line with the
`-ibmmq.monitoredQueues` flag, or put into a separate file which is also
named on the command line, with the `ibmmq.monitoredQueuesFile` flag. An
example is included in the startup shell script.

Note that **for now**, the queue patterns are expanded only at startup
of the monitor program. If you want to change the patterns, or new
queues are defined that match an existing pattern, the monitor must be
restarted.

## Configuring collectd
There are several steps needed to configure collectd. It will use the
'Exec' interface to manage the MQ collection.
* Edit the mq.conf file to point at the shell script that starts the real
program. The name of the queue manager to be monitored is also given here.
* Edit the shell script mq_coll.sh to set additional parameters for the
program, and ensure the program is being invoked from the correct directory.
* Put mq.conf in the /etc/collectd.d directory. That is read during
startup, telling collectd how to use MQ.
* Edit /etc/collectd.conf to include a reference to the list of metrics
that MQ generates. Do this by adding a reference to the mqtypes.db
file to the TypesDB line. For example

```
    TypesDB "/usr/share/collectd/types.db" "/usr/local/mqgo/mqtypes.db"
```

* Restart collectd to pick up new configuration. For example

```
    systemctl restart collectd
```

## Metrics
Once the monitor program has been started,
you will see metrics being available. The metric names sent to collectd
start with "qmgr" followed by the queue manager name. The next element
is the actual metric (queue_depth, system_cpu etc) followed by the
queue name if appropriate. You can check that the mq_coll program is running
with the ps command. That shows correct configuration of the collectd plugin
and the mq_coll.sh script.

There is some minor rewriting of object names, replacing "." with "_" to
conform with the collectd Exec interface; the metric names may get rewritten
again by the conversion to the backend storage (re-replacing "_" with "."
perhaps).

The example Grafana dashboard shows how queries can be constructed to extract data
about specific queues or the queue manager.

More information on the metrics collected through the publish/subscribe
interface can be found in the [MQ KnowledgeCenter]
(https://www.ibm.com/support/knowledgecenter/SSFKSJ_9.0.0/com.ibm.mq.mon.doc/mo00013_.htm)
with further description in [an MQDev blog entry]
(https://www.ibm.com/developerworks/community/blogs/messaging/entry/Statistics_published_to_the_system_topic_in_MQ_v9?lang=en)

The metrics stored in the database are named after the
descriptions that you can see when running the amqsrua sample program, but with some
minor modifications to match a more useful style.