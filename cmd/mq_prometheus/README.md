# MQ Exporter for Prometheus monitoring

This directory contains the code for a monitoring solution
that exports queue manager data to a Prometheus data collection
system. It also contains configuration files to run the monitor program

The monitor collects metrics published by an MQ V9 queue manager
or the MQ appliance. Prometheus than calls the monitor program
at regular intervals to pull those metrics into its database, where
they can then be queried directly or used by other packages
such as Grafana.

You can see data such as disk or CPU usage, queue depths, and MQI call
counts.

An example Grafana dashboard is included, to show how queries might
be constructed. The data shown is the same as in the corresponding
InfluxDB-based dashboard, also in this repository.
To use the dashboard,
create a data source in Grafana called "MQ Prometheus" that points at your
database server, and then import the JSON file.


## Building
* This github repository contains the monitoring program and
the ibmmq package that links to the core MQ application interface. It
also contains the mqmetric package used as a common component for
supporting alternative database collection protocols.

* You also need access to the Prometheus Go client interface.

  The command `go get -u github.com/prometheus/client_golang` should pull
  down the client code and its dependencies.

* The error logger package may need to be explicitly downloaded

  On my system, I also had to forcibly download the logger package,
  using `go get -u github.com/Sirupsen/logrus`.

Run `go build -o <directory>/mq_prometheus cmd/mq_prometheus/*.go` to compile
the program and put it to a specific directory.

## Configuring MQ
It is convenient to run the monitor program as a queue manager service.

This directory contains an MQSC script to define the service. In fact, the
service definition points at a simple script which sets up any
necessary environment and builds the command line parameters for the
real monitor program. As the last line of the script is "exec", the
process id of the script is inherited by the monitor program, and the
queue manager can then check on the status, and can drive a suitable
`STOP SERVICE` operation during queue manager shutdown.

Edit the MQSC script and the shell script to point at appropriate directories
where the program exists, and where you want to put stdout/stderr.
Ensure that the ID running the queue manager has permission to access
the programs and output files.

The monitor listens for calls from Prometheus on a TCP port. The default
port, reserved for this use in the Prometheus list, is 9157. If you
want to use a different number, then use the `-ibmmq.httpListenPort`
command parameter.

The monitor always collects all of the available queue manager-wide metrics.
It can also be configured to collect statistics for specific sets of queues.
The sets of queues can be given either directly on the command line with the
`-ibmmq.monitoredQueues` flag, or put into a separate file which is also
named on the command line, with the `ibmmq.monitoredQueuesFile` flag. An
example is included in the startup shell script.

Note that **for now**, the queue patterns are expanded only at startup
of the monitor program. If you want to change the patterns, or new
queues are defined that match an existing pattern, the monitor must be
restarted with a `STOP SERVICE` and `START SERVICE` pair of commands.


## Configuring Prometheus
The Prometheus server has to know how to contact the MQ monitor. The
simplest way is just to add a reference to the monitor in the
server's configuration file. For example, by adding this block
to /etc/prometheus/prometheus.yml.

```
  # Adding a reference to an MQ monitor. All we have to do is
  # name the host and port on which the monitor is listening.
  # Port 9157 is the reserved default port for this monitor.
  - job_name: 'ibmmq'
          scrape_interval: 15s

    static_configs:
          - targets: ['hostname.example.com:9157']
```

The server documentation has information on more complex
options, including the ability to pull information on which hosts
should be monitored from a variety of discovery tools.

## Metrics
Once the monitor program has been started, and Prometheus refreshed to
connect to it, you will see metrics being available in the prometheus
console. All of the metrics are given the jobname prefix of **ibmmq**.

More information on the metrics collected through the publish/subscribe
interface can be found in the [MQ KnowledgeCenter]
(https://www.ibm.com/support/knowledgecenter/SSFKSJ_9.0.0/com.ibm.mq.mon.doc/mo00013_.htm)
with further description in [an MQDev blog entry]
(https://www.ibm.com/developerworks/community/blogs/messaging/entry/Statistics_published_to_the_system_topic_in_MQ_v9?lang=en)

The metrics shown in the Prometheus console are named after the descriptions
that you can see when running the amqsrua sample program, but with some
minor modifications to match the required style.