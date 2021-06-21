# Crawler Operations

## Introduction

This page describes how to a manage our crawl services and jobs. This applies to all crawl jobs:

- The Domain Crawl (`dc`)
- The Frequent Crawls (`fc`) which consist of two different crawls (NPLD `npld` and By-Permission `bypm`)

This page assumes that a suitable crawler has already been set up, with sufficient disk space and with Docker Swarm installed. For details, see:

- _TBA: Link to local crawler setup_
- [AWS Domain Crawl Setup](./crawler_setup_aws_dc)

## Launching the Crawl Services

Before doing any service changes, it is often helpful to clear out any information associated with services that are no longer running. e.g. this avoids confusion when inspecting the logs of services.

    docker system prune -f

### Reviewing the Deployment Configuration

_TBA: VARS_

### Launching Kafka

The current crawl engine is integrated with _Apache Kafka_, and so this service needs be started before Heritrix.  This means the `fc_kafka` and `dc_kafka` stacks for the frequent and domain crawls respectively. These configurations are available at:

* https://github.com/ukwa/ukwa-services/tree/master/ingest/fc/fc-kafka
* https://github.com/ukwa/ukwa-services/tree/master/ingest/dc/dc-kafka

```{figure} ./fc_kafka_stack.svg
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


    
Check surts and exclusions. 
Check GeoIP DB (GeoLite2-City.mmdb) is installed and up to date.

JE cleaner threads
je.cleaner.threads to 16 (from the default of 1) - note large numbers went very badly causing memory exhaustion
Bloom filter
MAX_RETRIES=4

```{figure} ./fc_crawl_stack.svg
---
height: 500px
align: center
name: fc-crawk-stack
---
A visualiation of the main components of the `fc_crawl` Docker Stack configuration.
```


## Crawl Job Operations

The current crawl engine relies on Heritrix3 state management to keep track of crawl state, and this was not designed to cope under un-supervised system restarts. i.e. rather than being stateless, or delegating state management to something that ensures the live state is preserved immediately, we need to manage ensuring the runtime state is recorded on disk. This is why crawler operations are more complex than other areas.

### Starting Crawls

As stated above, before going any further, we need to ensure that Kafka has completed starting up and is ready for producers and consumers to connect.

- Build.
- Select Checkpoint. If expected checkpoints are not present, this means something went wrong while writing them. This should be reported to try to determine and address the root cause, but there's not much to be done other than select the most recent valid checkpoint.
- Launch.
- 

### Stopping Crawls

If possible, we wish to preserve the current state of the crawl, so we try to cleanly shut down while making a checkpoint to restart from.

Note that for our frequent crawls, we run two Heritrix services, one for NPLD content and one for by-permission crawling. When performing a full stop of the frequent crawls, both services need to be dealt with cleanly. When running on crawler06, this means:

- https://crawler06.bl.uk:8443/ is NPLD crawling.
- https://crawler06.bl.uk:9443/ is By-Permission crawling.

#### Pause the crawl job(s)

For all Heritrixes in the Docker Stack: log into the Heritrix3 control UI, and pause any job(s) on the crawler that are in the `RUNNING` state. This can take a while (say up to two hours) as each worker thread tries to finish it's work neatly. Sometimes pausing never completes because of some bug, in which case we proceed anyway and accept some inaccuracies in the crawl state. If it works, all `RUNNING` jobs will now be in the state `PAUSED`.

#### Checkpoint the job(s)

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


### Shutdown the Crawl Job(s)

At this point, all activity should have stopped, so it should not make much difference how exactly the service is halted.  To attempt to keep things as clean as possible, first terminate and then teardown the job(s) via the Heritrix UI.

You can now shut down the services...

## Shut down the Crawl Services

At this point, all activity should have stopped, so it should not make much difference how exactly the service is halted.  To attempt to keep things as clean as possible, first terminate and then teardown the job(s) via the Heritrix UI.

Then remote the crawl stack:

    docker stack rm fc_crawl
    
If this is not responsive, it may be necessary to restart Docker itself. This means all the services get restarted with the current deployment configuration.

    service docker restart
    
Even this can be quite slow sometimes, so be patient.
