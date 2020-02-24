
<!-- MarkdownTOC depth=2 autolink=true bracket=round lowercase_only_ascii=true -->

- [For NG Arch Pages](#for-ng-arch-pages)
    - [Document Harvesting Done Right](#document-harvesting-done-right)
- [Introduction](#introduction)
    - [Current Workflow](#current-workflow)
    - [Development](#development)
    - [Installation](#installation)
    - [Future Plans](#future-plans)
- [Moving to Luigi](#moving-to-luigi)
- [APIs & Formats](#apis--formats)
    - [The Crawl Feeds](#the-crawl-feeds)
- [Operating Notes](#operating-notes)

<!-- /MarkdownTOC -->


For NG Arch Pages
=================

* Separate Solr (w/ compatible schema) for frequently crawled and curated content? Pain point is keeping those hosts out of the other indexes. I guess host or SURT-prefixed mapping is probably the right tactic.

Document Harvesting Done Right
------------------------------

 - Document Harvester UI decoupled from the W3ACT data model. Pulls lists of documents and 'first guess' metadata from a (dedicated) Solr rather than stored in the W3ACT DB. i.e. PDF's with no DDHAPT status, from these hosts/prefixes.
     - Can Luigi be used to store output of manual curation? Should it?
 - Luigi used to store state of extraction, and can be used to re-run extraction.
 - Write a WARCDocumentSource that implements any23's [DocumentSource](https://any23.apache.org/apidocs/org/apache/any23/source/DocumentSource.html)
 - Include [any23](https://any23.apache.org/dev-data-extraction.html) in the extraction process, if only for Watched Targets.
 - Use data from multiple pages to infer the most likely 'landing page' rather than relying on harvester discovery path.
 - Implement the 'back-track-to-nearest-preceding-header' logic as one of the fall-back title extractors.
 - Also implement an extractor that pulls in manually curated metadata from DDHAPT.
 - Assemble all the metadata that 'points to' a given Document, probably in Solr. Use this to pre-populate catalogue.


Introduction
============

This repository contains our main 'frequent' crawl engine. It handles the bulk of our regular crawling activities, such as visiting news sites on a daily basis.

The orchestration of processes to manage the crawls is handled as a series of Python [tasks](http://docs.celeryproject.org/en/latest/userguide/tasks.html) executed on the [Celery](http://www.celeryproject.org/) distributed task queue system. This uses a instance of [RabbitMQ](https://www.rabbitmq.com/) to reliably store and distribute the work to be done. These tasks perform actions like starting and stopping crawl jobs, and packaging up the output. They also perform some mid-crawl processing, like extracting metadata for documents found during the crawl.

This is quite closely tied to our systems, e.g. it includes submission to the British Library's archival store, virus scanning mid-crawl, and other processes that depend on local policy. However, it is hoped that by working in the open we can maximise the chance of finding which components can be shared.


Current Workflow
----------------

The commands and Celery tasks described here are held in the `python-shepherd` submodule. The current code is in `python-shepherd/crawl`, and most of the tasks are in `python-shepherd/crawl/tasks.py`.

* A cron job is used to run the `pulse` command which enqueues a message indicating a particular H3 job should be (re)started (e.g. `pulse start daily`).
* `stop_start_job` reads [the crawl feeds](#feeds) and (re)starts the specified crawl:
    * [ ] TODO The H3 crawl job is started up and each seed gets rendered synchronously via [a dedicated processor](https://github.com/ukwa/bl-heritrix-modules/blob/master/src/main/java/uk/bl/wap/crawler/processor/WrenderProcessor.java).
    * `uri_to_index` As the job proceeds, each successfully-crawled URI is posted to a message queue and sent on to tinycdxserver.
    * `uri_of_doc` For document havesting, a subset of those messages is selected and processed by:
        * Attempting to automatically extract appropriate metadata
        * Pushing the resulting object to W3ACT for further attention/cataloguing.
    * `movetohdfs` A background process ferries WARCs from the crawl engine to HDFS
    * When a job is stopped, the job and lauch IDs are passed to a queue for processing ()
* `assemble_job_output` A dedicated task packages up the files associated with a job and ensures they are on HDFS
* `build_sip` A second task them packages them up as a SIP
* `submit_sip` Then the SIP is submitted
* `verify_sip` Then validated (NIY)
* `index_sip` (NIY)


Development
-----------

We are using [Docker](https://www.docker.com/) to make development of multi-component systems more manageable. Specifically, we define the set of services we need in a `docker-compose.yml` file (as per [Docker Compose](https://docs.docker.com/compose/)). Where possible, we use stable binary images for these services, but as a number of our own services are under active development, we [build](https://docs.docker.com/compose/reference/build/) them locally rather than downloading them.

To make it easier to split these off into separate development processes as they stabilise, they are included here as [git submodules](https://git-scm.com/book/en/v2/Git-Tools-Submodules). This means you must make sure you perform a [recursive clone](http://stackoverflow.com/questions/3796927/how-to-git-clone-including-submodules) if you want to work on these components.

    $ git clone --recursive git@github.com:ukwa/pulse.git

Then, in the `pulse` directory, with Docker installed, you should be able to do this to build and run the whole engine:

    $ docker-compose up

As development proceeds, you may find you need to run (or even force) a rebuild of the locally-defined components, to make sure changes get picked up:

    $ docker-compose build

or to force a full rebuild of a specific component:

    $ docker-compose build --no-cache w3act


Note that `w3act` is set up to mount your Ivy2 folder (`~/.ivy2/) so it can avoid re-downloading the required dependencies over and over.

n.b. seems like 3t annotation for fetchAttempts will be common, due to pre-requisites? yes ip:93.184.216.34,3t

* [ ] IP annotation missing.
* [ ] WarcFilename and WarcOffset missing.
* [ ] Content from warcprox is not virus scanned.
* [ ] Move to WARC processing task chain (checksum, copy-to-hdfs, virus-scan, cdx-index, extract-docs, solr-index)
    * In this case, the H3 indexing part can be removed, and the crawl packager would initiate and then await the WARCs being uploaded.
    * Note that we really should support checkpoint accumulation of content.
* [ ] ...

 
Installation
------------

TBA.


Future Plans
------------

Rendering Seeds

Want to render seeds with browser if possible, but fall back on H3 crawling if not. We do not want to crawl things twice. So, we want H3 to know it's crawled the URI, and our previous work in this area made this difficult - we could pass seeds to a renderer via AMQP but the seed would still be crawled again by H3. So, the plan would be to create a Processor that uses a web-render service to process content and extract links. If that works, it records the extracted links and skips the rest of the crawl chain. If it fails, the failure is logged and the item is simply passed on to the next processor, which is the usual H3 FetchHTTP implementation.

* Dead seeds report needed, along with crawl logs and reports.
* Recrawl versus refresh, news sites especially.

* Store system status somewhere and generate dashboard from that rather than doing both in one.
* Propose multiple H3 workers, running through warcprox to unify WARCs
* Proposed post-H3 processing:
    * Each job is registered with Bamboo.
    * As each WARC file is closed, it is registered with Bamboo and enqueued for processing.
    * A sequence of message queues is used to:
        * Scan for viruses
        * Upload to HDFS
        * Index into CDX server
        * Perform full-text indexing
        * Scan for documents and extract metadata
    * At each point, the current status in Bamboo is updated via the Bamboo API.

Moving to Luigi
===============

The Celery code works okay, but is rather brittle as it requires a lot of 'boilerplate' to make it properly robust. There are a number of production-quality workflow engines that we could use instead, and that provide better robustness and monitoring, like AirFlow and Luigi. AirFlow has a richer dashboard and uses Celery+RabbitMQ as it's back-end, which makes it very scaleable, but it is very complex to use. The job definitions are hard to understand, basic things like how parameters are passed through are unclear, and the job state is dependend on a central database.

Luigi, on the other hand, is not as scaleable but is much easier to understand and work with ([here's an intro](http://bytepawn.com/luigi.html). Task inputs and outputs can be specified, and it makes it easy to use local or HDFS files as 'markers' in a manner similar to Makefiles, i.e "if this files exists, this job has been done". This very much suites the way we work, and if we use HDFS where we can, it will be more robust and transparent than RabbitMQ. Of course, storing many small files in HDFS is wasteful, but you can fit 15,625 64MB files in a TB of HDFS storage, so we should not over-optimise for this. In practice, we can use local files for fine-grained work, and then shift the results to HDFS when chunks of work are complete.

This approach will keep our original workflow, but rather than using message queues, we will use file outputs to record the state.

* `stop_job(jobName)` stops a job, outputting a /jobs/jobName-launchId-stopped file that contains data when the job has been stopped.
* `start_job(jobName)` looks to see the jobName is stopped, and re-starts, leaving a /jobs/jobName-launchId-started ?
* `assemble_job_output` pulls together the output from the job and pops in on HDFS, leaving a hdfs:/1_data/jobs/jobName/launchId/stopped file when done.
* `build_sip` creates the SIP, leaving a hdfs:/1_data/heritrix/sips/....tar.gz when done. Also leave ARKs somewhere useful?
* `submit_sip` submits the sip, leaving a submitted file when done (?)

Something, somehow, processes WARCs for a job... Maybe yielded by assemble_job_output

* `move_to_hdfs(warc)` does move_to_hdfs if the target file does not exist (hash-check?)

Then, independently of other processes (?), something should check for new WARCs and...

* `cdx_index(warc)`
* `solr_index(warc)`

Also, periodic backups of local status files to HDFS would be something to build on this engine.

On top, I need to allow checkpoint incremental processing. i.e. checkpoint_job(jobName) should lead to 'assemble_job_output(jobName, launchId, cp)', which should re-build a new sip with the checkpoint-ID added. `submit_sip` should be aware of this and prevent an earlier checkpoint being submitted.


Note [webhdfs is available](http://luigi.readthedocs.io/en/stable/api/luigi.contrib.hdfs.webhdfs_client.html#luigi.contrib.hdfs.webhdfs_client.WebHdfsClient) and the underlying library is the one we were already using.

It can do [remote/SSH actions](http://luigi.readthedocs.io/en/stable/api/luigi.contrib.ssh.html), so we may be able to use this to avoid having to run on the same server as Heritrix (e.g. for populating the job files etc. via [RemoteTarget](http://luigi.readthedocs.io/en/stable/api/luigi.contrib.ssh.html#luigi.contrib.ssh.RemoteTarget))

This system has excellent support for date-ranges, so it might make sense to bundle up the overall status into by-date summary files. This would make 'backfill' easier, i.e. re-scanning over a period of time and checking/reprocessing as necessary.

NOTE requires allows tasks to run in parallel - task yielded from a run() will be serialised.

Second version derives partial packages from checkpoints, before the final DONE package aggregates everything.

So, final 'stopped' should look to process all crawl.log*

output/logs/daily/20161019234428/alerts.log
output/logs/daily/20161019234428/alerts.log.cp00001-20161020204551
output/logs/daily/20161019234428/crawl.log
output/logs/daily/20161019234428/crawl.log.cp00001-20161020204551
output/logs/daily/20161019234428/frontier.recover.gz
output/logs/daily/20161019234428/frontier.recover.gz.cp00001-20161020204551
output/logs/daily/20161019234428/nonfatal-errors.log
output/logs/daily/20161019234428/nonfatal-errors.log.cp00001-20161020204551
output/logs/daily/20161019234428/progress-statistics.log
output/logs/daily/20161019234428/progress-statistics.log.cp00001-20161020204551
output/logs/daily/20161019234428/runtime-errors.log
output/logs/daily/20161019234428/runtime-errors.log.cp00001-20161020204551
output/logs/daily/20161019234428/uri-errors.log
output/logs/daily/20161019234428/uri-errors.log.cp00001-20161020204551

each should be packaged up, i.e.

 * Take crawl.log.cp00001-20161020204551 and make [ job-name.launch-id.cp00001-20161020204551.info.json, job-name.launch-id.cp00001-20161020204551.logs.zip
     * If we do mid-crawl packages, then we would make a job-name.launch-id.cp00001-20161020204551.package.json describing the aggregate content until that point.
 * If job-name.launch-id.stopped, do the same for crawl.log, but grab ALL the accounted and un-accounted for WARCs, creating job-name.launch-id.final.info.json
 * Collect together all the mid-crawl data, creating job-name.launch-id.final.package.json
 * Build SIP from that, once all dependencies are available on HDFS.


### Job Stop/Start Task Chain ###

| Task                                  | Requires                                            | Task Output                                  | 
| ------------------------------------- | --------------------------------------------------- | -------------------------------------------- |
| StopJob(job)                          | _none_                                              | 01.jobs._{job.name}.{launch_id}_.stopped     |
| StartJob(job)                         | StopJob(job)                                        | 01.jobs._{job.name}.{launch_id}_.started     |


### Assembly Task Chain ###

| _output_.Task                         | Requires                                                               | Task Output                                    | 
| ------------------------------------- | ---------------------------------------------------------------------- | ---------------------------------------------- |
| CheckJobStopped(job, launch_id)       | _pulse.StopJobExternalTask_                                            | 01.jobs._{job.name}.{launch_id}_.stopped       |
| PackageLogs(job, launch_id, stage)    | if stage == 'final': <br/> pulse.CheckJobStopped(job, launch_id)       | 02.logs._{job.name}.{launch_id}_.stage.zip     |
| AssembleOutput(job, launch_id, stage) | PackageLogs(job, launch_id, stage)                                     | 03.outputs._{job.name}.{launch_id}_.stage      |
| ProcessOutputs(job, launch_id)        | for each stage (checkpoint):<br/>AssembleOutput(job, launch_id, stage) | 04.assembled._{job.name}.{launch_id}_.complete |
| ScanForOutputs(date_interval)         | for each (job, launch_id): <br/> ProcessOutputs(job, launch_id)        | _none_                                         |


* `output.ScanForOutputs(date_interval)` <br/> `for each (job, launch_id)`:
    * `output.ProcessOutputs(job, launch_id)` <br/> `for each stage (checkpoint/final)`:
        * `output.AssembleOutput(job, launch_id, stage)`:
            * `output.PackageLogs(job, launch_id, stage)` <br/> `if stage == 'final'`:
                * `pulse.CheckJobStopped(job, launch_id)`:
                    * **Result:** `01.jobs._{job.name}.{launch_id}_.stopped`
                * **Result:** `02.logs._{job.name}.{launch_id}_.stage.zip`
            * **Result:** `03.outputs._{job.name}.{launch_id}_.stage`
        * **Result:** `04.assembled._{job.name}.{launch_id}_.complete`
    * **Result:** _none_

### Packaging Task Chain ###

* _package.ScanForPackages(date_interval)_:
    * *Output:* _none_

### HDFS Transfer Task Chain ###

* `files.ScanForOutputFiles(date_interval)`:
    * `files.CalculateLocalChecksum(date_interval)`:
        * `files.VerifyOnHDFS(date_interval)`:
            * `files.CopyToHDFS(date_interval)`:
    * *Output:* _none_


APIs & Formats
==============

The Crawl Feeds
---------------

TODO consider harmonising with https://github.com/internetarchive/brozzler/blob/master/job-conf.rst

[JSON Lines](http://jsonlines.org/) for data


Operating Notes
===============

Issue, track down:

PULSE tasks logs indicate a H3 exception:

./docker-prod.sh logs --tail 1000 ukwa-heritrix-weekly

OOM
