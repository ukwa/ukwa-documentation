# Crawler Operations

## Introduction

This page describes how to a manage our crawl services and jobs. This applies to all crawl jobs:

- The Domain Crawl (`dc`)
- The Frequent Crawls (`fc`) which consist of two different crawls (NPLD `npld` and By-Permission `bypm`)


## Pre-requisites

This page assumes that a suitable host server has already been set up, with sufficient disk space, and with Docker Swarm installed. To meet NPLD requirements, the server must have direct internet access, and use a public IP address that is registered under a domain name that makes it clear who we are.  This means setting up the DNS records so that site owners can look up who we are based on the IP address alone ( IP => hostname ).

For further details, see:

- _TBA: Link to local crawler setup_
- [AWS Domain Crawl Setup](./crawler_setup_aws_dc)

### Checking Service Status

You can check if any stacks are currently deployed using:

    docker stack ls

And check the services for a stack with (e.g.):

    docker stack ps fc_kafka

You should check you understand the current status of the services before you proceed. For example, if you just rebooted the server, then all the services will come up automatically, but you'll still have to wait for Kafka before launching a crawl (see below).

If the state of the system cannot be made clear, the simplest thing is to start afresh:

1. as per instructions below, remove all running Docker Stacks.
2. reboot the server.

The system will then come back with no live services, and you may launch them as described below, and resume the relevant crawls as appropriate.

### Cleaning Up

Note that before doing any service changes, it is often helpful to clear out any information associated with services that are no longer running. e.g. this avoids confusion when inspecting the logs of services.

    docker system prune -f

## Reviewing the Deployment Configuration

_TBA: VARS_

## Launching the Kafka stack

The current crawl engine is integrated with _Apache Kafka_, and so this service needs be started before Heritrix.  This means the `fc_kafka` and `dc_kafka` stacks for the frequent and domain crawls respectively. These configurations are available at [fc-kafka](https://github.com/ukwa/ukwa-services/tree/master/ingest/fc/fc-kafka), [dc-kafka](https://github.com/ukwa/ukwa-services/tree/master/ingest/dc/dc-kafka).

```{figure} ../../arch-plots/stacks/fc_kafka.svg
---
height: 500px
align: center
name: fc-kafka-stack
---
A visualiation of the main components of the `fc_kafka` Docker Stack configuration.
```

To deploy the Kafka service stack. e.g. for `dc`:

    cd ingest/dc/dc-kafka
    ./deploy-kafka-prod.sh

We can then check what's happening either using:

    docker service ps --no-trunc dc_kafka_kafka_1

    docker service logs --tail 100 -f dc_kafka_kafka_1

If the Kafka instance is brand new, it will be up and running very quickly. However, if this is any kind of restart, Kafka can take quite some time to start up, with regular logging like this:

    ...Loading producer state from snapshot files...

Once it's checked all the partitions of the topics it contains, the logging should settle down. You can also check this by looking at the _Trifecta_ user interface, which should be accessible on port 9000 of the crawling machine.  Note that it does tend to get confused if it is accessed before Kafka is ready. In this case, it'll need to be restarted, like this:

    docker service update --force dc_kafka_ui


## Launching the Crawler stack

The `fc_crawl` and `dc_crawl` stack definitions can be found at [fc-crawl](https://github.com/ukwa/ukwa-services/tree/master/ingest/fc/fc-crawl), [dc-crawl](https://github.com/ukwa/ukwa-services/tree/master/ingest/dc/dc-crawl).
    
```{figure} ../../arch-plots/stacks/fc_crawl.svg
---
height: 500px
align: center
name: fc-crawk-stack
---
A visualiation of the main components of the `fc_crawl` Docker Stack configuration.
```

The `fc_crawl` stack is the more complex than the `dc` equivalent, as it includes the components needed to render some web pages using a web browser behind an archiving proxy (`warcprox`).  There are also some differences in how the crawls are configured. The details of the Heritrix configuration are described in _TBA_.

_TBC_

- Check surts and exclusions. 
- Check GeoIP DB (GeoLite2-City.mmdb) is installed and up to date.
- JE cleaner threads
- je.cleaner.threads to 16 (from the default of 1) - note large numbers went very badly causing memory exhaustion
- Bloom filter
- MAX_RETRIES=4

## Crawl Jobs

The current crawl engine relies on Heritrix3 state management to keep track of crawl state, and this was not designed to cope under un-supervised system restarts. i.e. rather than being stateless, or delegating state management to something that ensures the live state is preserved immediately, we need to manage ensuring the runtime state is recorded on disk. This is why crawler operations are more complex than other areas.

Specifically, when running long crawls, operational managment of Heritrix tends to focus on managing crawl state and the _checkpoints_ that preserve the state of the crawl frontier so that crawls can be resumed if needed.

Note that as well as the domain crawl, we run two Heritrix 'frequent crawl' services, one for NPLD content and one for by-permission crawling. When performing a full stop of the frequent crawls, both services need to be dealt with cleanly.


:::{table} Crawls and Service Endpoints. These are only accessible from UKWA networks and by UKWA staff.
:name: crawls-table
:align: center

| Crawl                  | Heritrix Service              | Kafka UI                      | Prometheus                    |
| ---------------------- | ----------------------------- | ----------------------------- | ----------------------------- | 
| Frequent NPLD          | https://crawler06.bl.uk:8443/ | https://crawler06.bl.uk:9000/ | https://crawler06.bl.uk:9191/ |
| Frequent By-Permission | https://crawler06.bl.uk:9443/ | _as above_                    | _as above_                    |
| Domain                 | https://crawler07.bl.uk:8443/ | https://crawler07.bl.uk:9000/ | https://crawler07.bl.uk:9191/ |

:::

## Starting Crawl Jobs

Crawl jobs are started via the Heritrix service. Every UKWA Heritrix service has a job called `frequent` that is used to perform the crawl for that instance of Heritrix, e.g. <https://crawler06.bl.uk:8443/engine/job/frequent> for the Frequent NPLD crawl.

Visiting those pages, we see that the job is `Unbuilt`, so the first thing to do is use the `BUILD` action to set up the job. This takes the configuration file known as 'crawler beans' and uses it to generate a set of live Java objects that will run the crawl. If all goes well, the job should then report as `Ready`.

At this point, we can either launch a new job, or launch from a chosen checkpoint. In general, we want to resume a crawl, so we pick the checkpoint with the most recent timestamp and click `LAUNCH`.  If expected checkpoints are not present, this means something went wrong while writing them. This should be reported to try to determine and address the root cause, but there's not much to be done other than select the most recent valid checkpoint.  If no recent checkpoints are valid, or if we want to clear out the space, or if we're making a significant change to how the crawler works, then we'll have to start afresh.

While the launch/resume process is happening, the crawler will report as `PREPARING`. Once that's done, the crawl will either immediately start crawling (i.e. straight into the `RUNNING` state), or launch into the `PAUSED` state. This depends on the Heritrix configuration, and in general we let the frequent crawls start running immediately, but start the domain crawls in `PAUSED` state so we can keep a closer eye on things.

- Unbuilt -> Ready
- Active: PREPARING, RUNNING, EMPTY, PAUSING PAUSED, STOPPING, Finished/ABORTED

(crawl-op-unpause)=
## Unpausing crawl job(s)

When the crawler is in the `PAUSED` state, just click the `UNPAUSE` button to begin or resume a crawl.

## Monitoring Crawl Jobs

When the crawl is running happily, the Heritrix job page should report a reasonable number of URI/s processed. Usually at least 10 URI/s, but peaking in the hundreds for larger crawls.

Every crawl stack includes an embedded Prometheus instance that gathers crawl metrics and makes them available for direct assessment and federated aggregation into our production Prometheus monitoring service. 

## Stopping Crawl Jobs

If possible, we wish to preserve the current state of the crawl, so we try to pause and cleanly shut down while making a checkpoint to restart from.

(crawl-op-pause)=
## Pause the crawl job(s)

For all Heritrixes in the Docker Stack: log into the Heritrix3 control UI, and pause any job(s) on the crawler that are in the `RUNNING` state. After the `PAUSE` button is pressed, the crawl enters the `PAUSING` state, while it waits for various threads to shutdown neatly. This can take a while (say up to two hours) if a given `ToeThread` is doing something like processing many thousands of discovered URLs from a site map.  If all goes well, all `RUNNING` jobs will eventually go from `PAUSING` to `PAUSED`.

Sometimes pausing never completes because of some bug. For example, under some cirumstances a `ToeThread` can die due to a run time error, and after that happens, and attempt to pause will get stuck in the `PAUSING` state. 

If the crawler is being paused due to a temporary network outage, it doesn't matter if the crawler is `PAUSING` rather than `PAUSED`. In either case, we can go ahead and [unpause](crawl-op-unpause) when the outage is over.  However, it does not hurt to take this opportunity to checkpoint each crawl job, in case something goes wrong while we're `PAUSING/PAUSED`.

However, if some critical component is down (like a disk drive or remote service like the crawl-time CDX index), then the `ToeThreads` may all get stuck trying to shut down. If this happens, there's not a lot we can do except accept shut down and accept that we may need to restart from an older checkpoint.

## Checkpoint the job(s)

Via the UI, request a checkpoint. If there's not been one for a while, this can be quite slow (tens of minutes). If it works, a banner should flash up with the checkpoint ID, which should be noted so the crawl can be resumed from the right checkpoint. If the checkpointing fails, the logs will need to be checked for errors, as unless a new checkpoint is succefully completed, it will likely not be valid.

As an example, under some circumstances the log rotation does not work correctly. This means non-timestamped log files may be missing, which means when the next checkpoint runs, there are errors like:

    $ docker logs --tail 100 fc_crawl_npld-heritrix-worker.1.h21137sr8l31niwsx3m3o7jri
    ....
    SEVERE: org.archive.crawler.framework.CheckpointService checkpointFailed  Checkpoint failed [Wed May 19 12:47:13 GMT 2021]
    java.io.IOException: Unable to move /heritrix/output/frequent-npld/20210424211346/logs/runtime-errors.log to /heritrix/output/frequent-npld/20210424211346/logs/runtime-erro
    rs.log.cp00025-20210519124709

These errors can be avoided by adding empty files in the right place, e.g.

    touch /mnt/gluster/fc/heritrix/output/frequent-npld/20210424211346/logs/runtime-errors.log

But immediately re-attempting to checkpoint a paused crawl will usually fail with:

    Checkpoint not made -- perhaps no progress since last? (see logs)

This is because the system will not attempt a new checkpoint if the crawl state has not changed. Therefore, to force a new checkpoint, it is necessary to briefly un-pause the crawl so some progress is made, then re-pause and re-checkpoint.


## Stop the Crawl Job(s)

At this point, all activity should have stopped, so it should not make much difference how exactly the service is halted.  To attempt to keep things as clean as possible, first terminate the job. The status should go to `STOPPING` and then to `ABORTED`, although in some cases the system can get stuck in `STOPPING`. If that happens, give it about ten minutes and then proceed to teardown the job(s) via the Heritrix UI.

You can now shut down the services at the Docker level.

## Shut down the Crawl Services

First, remove the crawl stack, e.g.:

    docker stack rm fc_crawl

The Heritrix containers are configured to give themselves a few minutes to shut down neatly.  If this is unnecessary, the container can be killed directly using `docker rm <container.id>` .

Then shut down Kafka

    docker stack rm fc_kafka
    
If Docker actions are not responsive, it may be necessary to restart Docker itself. This means all the services get restarted with the current deployment configuration.

    service docker restart
    
Even this can be quite slow sometimes, so be patient.

If all else fails, force a reboot.

