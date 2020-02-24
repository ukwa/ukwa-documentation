Monitoring Services
===================

<!-- MarkdownTOC depth=2 autolink=true bracket=round lowercase_only_ascii=true -->

- [Introduction](#introduction)
- [Overall Monitoring Approach](#overall-monitoring-approach)
- [Alerts & Statistics](#alerts--statistics)
- [Log Indexing & Debugging Console](#log-indexing--debugging-console)

<!-- /MarkdownTOC -->

Introduction
------------

We need to monitor services to raise alerts when things go wrong, to collect and track statistics on behaviour, and to allow us to debug issues when they arise. In the past, our services have been few in number, and deployed on only a few machines, and much of this has been handled using traditional tools which are not well-suited to highly 'elastic' containerised or cloud-oriented systems. ([This presentation](https://www.youtube.com/watch?v=hCBGyLRJ1qo) explores some of these issues.)

As our container-oriented crawl system is likely to contain large numbers of separate services, we need to move to a scalable approach to monitoring, based on tracking critical metrics. As elsewhere, we prefer to configure and use well-established best-of-breed open source applications to handle these kind of infrastructural issues, and attempt to minimise the amount of custom code we need.

Overall Monitoring Approach
---------------------------

![Monitoring for Alerts, Statistics & Debugging](./drawings/ng-was-monitoring.jpg)

The monitoring system handles two classes information - metrics and logs. The metrics are used to generate alerts and statistics, and the logs are cached so we can debug problems.

Alerts & Statistics
-------------------

The metric monitoring system is focussed on exposing metrics that can be used to check the health of the system. For long-running services, this means ensuring those services are up and they are responding in a timely fashion. Batch jobs and tasks can't be queried in this way, so instead are modified to update suitable metrics upon completion, and the monitoring system checks those metrics are being updated as expected.

We use the [Prometheus](https://prometheus.io/) metrics service to record and track these metrics, and [Grafana](https://grafana.com/) to provide a sutiable service summary screen. We can add [alert rules](https://prometheus.io/docs/alerting/rules/) to watch out for problems, and Prometheus will notify us via the [alert manager](https://prometheus.io/docs/alerting/alertmanager/) component.

We use Prometheus's ['blackbox exporter'](https://github.com/prometheus/blackbox_exporter) to monitor our groups of HTTP services, and there are a large number of other [exporters and integrations](https://prometheus.io/docs/instrumenting/exporters/) we use to monitor common software applications.

Services that are specific to us require additional ['exporters'](https://prometheus.io/docs/instrumenting/writing_exporters/) or [instrumentation](https://prometheus.io/docs/instrumenting/clientlibs/) to make the metrics available, but these are few in number and mostly just require us to re-factor our existing monitoring code.

For batch tasks, each should post suitable metrics to the [Push Gateway](https://prometheus.io/docs/instrumenting/pushing/) when completed, and the metric dashboard and alerts should be configured to monitor that the task has executed successfully within the expected time-frame.

For example, when we back-up the W3ACT database, [once the task chain complete successfully](http://luigi.readthedocs.io/en/stable/tasks.html#events-and-callbacks), we post a timestamp and the size of the backup (in bytes) as metrics via the Push Gateway. These are polled by Prometheus, from where we can use Grafana to check that a backup has occurred in the last X hours, and that the new backup is of the expected size (roughly the same or larger than the previous backup). Raising alerts in this way, based on tests of expected outcomes, is much more robust than expecting that we will always be reliably alerted when a task fails to run.

Areas to monitor:

- Check HTTP-based services are up and responsive (in groups)
- Check HDFS storage status and increase. (requires a custom 'exporter' to scrape HTML tags)
- Check Crawler status (requires a custom 'exporter' scanning Heritrix instances and summarising metrics)
- Check Crawler local disk space etc.
- Check AMQP/Kafka queues.


Log Indexing & Debugging Console
--------------------------------

Service logs and other events (like crawl events) routed from servers e.g. using `filebeat` to send logs to a Kafka service from which `logstash` can consume events and then push the results to `elasticsearch`. This data can be inspected via [Kibana](https://www.elastic.co/products/kibana), which acts as a 'debugging console' that be used to work our what's happening. Using Kafka as the transport makes it possible for all such logging processes to be done in a consistent manner without them being tightly-coupled.

In contrast to the alerting & statistics system, only the recent logs are kept here, although some useful metrics may be extracted from the logs and passed to the metrics service for tracking. Any 'formal' reporting should be implemented as ingest or access tasks rather than done here.

Debugging of specific batch tasks should be possible via the Luigi Scheduler interfaces. In this case, the goal of the monitoring layer is to route you to the correct Luigi Scheduler and task result.

For problems with long-running services, the goal is to summarise recent activity and direct you to the problematic system or systems.


