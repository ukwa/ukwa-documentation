The goal is a stable and robust crawl lifecycle, automated end-to-end. The frequent crawl stream should run with no manual intervention, and while the domain crawl may be initiated manually, the ingest and processing of content collected by the crawl should use the same framework.

This document covers the current and near-future architecture. Legacy components will not be described in detail here.

Contents
========

Overall Design
==============

During long-running processes, failure is expected. Therefore, everything is built to be idempotent, as it may be retried many times. Transient failure are reported but not critical unless persistent.

Overall, we use a small number of larger, modular components (mostly written in Java), and then ‘glue’ them together using Python scripts. Where appropriate, these scripts use the [*Luigi*](https://github.com/spotify/luigi) task framework to help ensure the processes are robust and recoverable. All services are wrapped as Docker containers for deployment.

The major components are:

-   Developed by us and/or IIPC members:

    -   [*W3ACT*](https://github.com/ukwa/w3act) (UKWA-only)

    -   [*Heritrix3*](https://github.com/internetarchive/heritrix3)

    -   [*warcprox*](https://github.com/internetarchive/warcprox) (Python)

    -   [*webarchive-discovery*](https://github.com/ukwa/webarchive-discovery)

    -   [*OutbackCDX*](https://github.com/nla/outbackcdx) (a.k.a. tinycdxserver)

    -   [*OpenWayback*](https://github.com/iipc/openwayback)

    -   The UKWA Website engine (currently under development as [*ukwa/marsspiders*](https://github.com/ukwa/marsspiders))

-   Standard open source components:

    -   PostgreSQL

    -   Solr

    -   Hadoop (HDFS, Map-Reduce, and in time likely HBase too)

    -   ClamD

    -   PDFtoHTMLex

    -   Jupyter Notebooks

The [*python-shepherd*](https://github.com/ukwa/python-shepherd) code base (rename to ukwa-ingest?) contains all the ‘glue’ code that orchestrates these crawl processes. All lifecycle task code should be in here, from ingest to discovery. Note this this work is currently on the [*hadoop-first*](https://github.com/ukwa/python-shepherd/tree/hadoop-first) branch of that project.

All events, across all production systems, are initiated by cron jobs on sh.wa.bl.uk. For example, a cron job on sh.wa.bl.uk may initiate a Luigi task on the the ingest server that scans crawl engines for new content to upload to HDFS.

Monitoring
----------

The [*ukwa-monitor*](https://github.com/ukwa/ukwa-monitor) codebase monitors and reports on all UKWA processes, including ingest. This provides reports/dashboards/alerts, by probing the production system. No part of any production system depends on it, however, and it runs it’s own cron jobs (rather than being initiated by sh.wa.bl.uk). If it's down, nothing else should be affected. If transient failures persist, it is the role of the monitor engine to raise the alert.

Deployment
----------

The [*pulse*](https://github.com/ukwa/pulse) codebase pulls the different crawl components together using [*Docker*](https://www.docker.com/), and uses [*Docker Compose*](https://docs.docker.com/compose/) to define different deployments, from local development to full production.

The intention is that the supplied [*docker-compose.yml*](https://github.com/ukwa/pulse/blob/master/docker-compose.yml) file can be use to run an end-to-end test of a crawl, all within a set of Docker containers running locally or as part of a Travis CI automated build.

Production containers are built on [*Docker Hub*](https://hub.docker.com/u/ukwa/), using automated builds based on tagged versions of the relevant GitHub codebases, e.g. [*here’s the build history for W3ACT*](https://hub.docker.com/r/ukwa/w3act/builds/).

Harvest & Ingest
================

In the crawl engine, one front-end (currently the ‘shepherd’) provides their services to stop/start crawls. It downloads the information on what should be crawled using the ‘crawl feed’ API of W3ACT, and launched or re-launches crawls appropriated. It also performs any routine cleanup and management operations. It should not itself perform other processes like moving content to HDFS. (It does at the moment!)

The ingest server manages the ingest process and verification of ingest, both to HDFS and DLS, and manages the other routine analysis and indexing processes.

Each crawl is identified by a job name and a launch date, and this identifier is used to manage the content as it arrives. For example, the Weekly crawl launched on the 21st of March 2017 has the identifier weekly/20170321162430.

The ingest server (pushes and) verifies content on HDFS, and permits deletion of content from the crawl engines. Our approach is that no other processing should be initiated based on the content locally-stored on each crawl engine. Nothing is done until the content is on HDFS.

The content for this crawl is stored on HDFS under the following file convention:

> /heritrix/output/**{TYPE}**/weekly/20170321162430

In usual crawling, we get WARCs, WARCs containing ‘nullified’ suspected viruses, and log files:

> /heritrix/output/warcs/weekly/20170321162430
>
> /heritrix/output/viral/weekly/20170321162430
>
> /heritrix/output/logs/weekly/20170321162430

Every crawl is set to ‘checkpoint’ itself every few hours, and as part of this it rolls-over any WARC or log files that are currently in use. So, for each checkpoint, we get a crawl log file, like this:

> .../tweekly/20170321162430/crawl.log.cp00005-20170321222911

This log file can be scanned to see which WARCs it refers to, and together these logs and WARCs form a coherent ‘chunk of crawl’.

Note that the screenshot WARCs are slightly more difficult to handle as the checkpointing mechanism cannot currently be extended to simultaneously synchronize Heritrix3 and warcprox output. Here, we make a best effort to ensure the right screenshot WARCs are bundled with the right crawl logs, based on the timestamps of the resources.

Moving content to HDFS
----------------------

The most critical step in our workflow is moving crawled data from the crawl engine to HDFS, deleting it from the origin crawl server so that the crawlers don’t fill up and halt.

To make do this as carefully as possible, we perform a two-stage copy-and-verify then verify-and-delete. Each stage computes the SHA-512 hash of each transferred file, and we use two separate HDFS hashing operations based on separate codebases.

There are two implementations of the copy-then-verify step. One is designed to run locally on the source machine, and the other runs on a central machine and uses ssh to remotely log into a crawl server and scan the contents. See here:

[*https://github.com/ukwa/python-shepherd/tree/hadoop-first/tasks/ingest*](https://github.com/ukwa/python-shepherd/tree/hadoop-first/tasks/ingest)

These are called variations of ‘move to HDFS’ and can be used to carry out a one-step copy-verify-delete if required (but I’d like to move to this two-step approach).

The process to scan HDFS, hash all the files, and store the hashes somewhere useful is still under development: [*https://github.com/ukwa/python-shepherd/blob/hadoop-first/tasks/process/hadoop/hasher.py\#L170*](https://github.com/ukwa/python-shepherd/blob/hadoop-first/tasks/process/hadoop/hasher.py#L170)

The open question is how best to batch HDFS files for batching and how best to store the hashes so the ‘verify and delete’ process can compare the local server with HDFS and compare the Java-based and Python-based hashes to check all are consistent. The Python-derived hashes can be store in the assembled

Analyse
-------

As content appears on HDFS, we first need to identify the ‘chunks of crawl’ described above. We start this by scanning log files.

For each new set of logs, we scan for associated WARC files, potential documents, run basic stats analysis.

Verify
------

This would be the logical spot to run the Map-Reduce hasher and double-check the hashes are as expected, at which point we can clear the files on the crawl servers for deletion.

Assembe
-------

We take the initial chunks of logs and warc and assemble them into a single description that also contains any required metadata in associated ZIP files. This also contains the SHA-512 of the files and any necessary ARK identifiers.

The main result is a JSON description of the crawl at a given point in time. Any crawl process that generates these can be plumbed into the downstream processes.

Package & Submit
----------------

Take each chunk and assemble SIPs incrementally

Submit them to DLS

Validate
--------

We check the DLS export, and we check against our DLS Access Point.

Analysis & Indexing
===================

...following on from Assembly, we can now start to process chunks of crawl for access.

See [*2017 Web Archive Search Strategy*](http://drive.google.com/open?id=1CJUvyI1XPOZt6oEl_2oFRJHXrv8K3_jT36AkOSEcjzw)

### Next Steps in Development

The largest gap in the current indexing stack is problem of ‘reduplication’ - compensating for the ‘revisit’ records which we use to avoid storing multiple copies of the same resource. At index time, we need to reverse the de-duplication so we can search through the content of these resources over the whole time-frame. For this to work, we need a large index where we can store information associated with a URI, but drawn from different WARC records.

This kind of lookup table (effectively a very large JOIN), would also have other uses:

-   Associating Request records with the relevant Response.

-   Storing full timelines for each URI, e.g. when we visited and it was gone, based on the crawl logs.

-   Associating metadata or annotations with the target resource, e.g. taking advantage of the gov.uk content API during indexing, or exploiting additional metadata extracted using tools like [*any23*](https://any23.apache.org/).

-   Estimate URI lifespan, to add as a search facet.

-   At crawl time, as a lookup that replaces and improves upon the current methods of deduplication and ‘already-seen-URIs’ checks, further reducing the amount of stage managed by the Heritrix crawler itself and significantly lowering the memory footprint due to the removal of the Bloom-filter-based already-seen URI lookup method.

In this model, the crawl logs and WARCs would be processed to populate this large lookup table. The simplest approach is probably a single ‘timeline’ index (CDX-style), grouping pointers to records by target URI. As we index WARCs, we could look up the current URI to look for additional information to add to the record. It could probably be done using OutbackCDX, but if there are scaling issues (or if a more sophisticated data model is needed) it would be a good fit for a dedicated Solr index, or possibly HBase if very high performance and very large scale are required.

A more sophisticated tactic would be to use HBase as the main data store, not just for lookups. The WARC processing would populate HBase rather than Solr, but we could generate the Solr index from HBase. This would allow more complex *ad hoc* analysis and may make re-generating Solr indexes slightly easier. However, it’s a more difficult approach to scale *down*, so would probably be of less interest to our IIPC colleagues.

The webarchive-discovery project needs some attention, once we’ve settled on plan. I intend to propose something along the lines of:

-   A cleared front-end/back-end separation, as for [*solrmarc*](https://github.com/solrmarc), where we encourage a common Solr document model and indexing process, but accept that front-end technologies might vary.

-   A plan to bring Shine and UKWA together as our front-end.

-   Some kind of plan for how to provide additional services, like link-graphs, or format data, etc.

Monitoring
==========

At each stage, we can add monitoring processes that ‘reach into’ the above process chains and generate HTML reports, dashboards, alerts etc.

The Dashboard

Crawl Reports

HDFS Reports

Alerts

Ideas
=====

See also [*https://trello.com/b/nJCKUtYf/ukwa-next-generation-services*](https://trello.com/b/nJCKUtYf/ukwa-next-generation-services)

Development Work Ideas
----------------------

-   W3ACT:

    -   Browser plugin for curation to complement W3ACT?

-   Crawler development:

    -   External Frontier, storing queues outside H3 and in e.g. Redis/HBase

    -   CDX-based (re-using the Timeline Index described above) already-seen and de-duplication

    -   Container-level crawl job management

    -   Finer-grained jobs and launch windows

-   Analysis & Processing

    -   Robust lifecycle management and alerting

    -   Automated screenshot comparison

    -   Report generation

-   Access:

    -   High-fidelity playback without re-writing:

        -   Dedicated desktop browser in ‘proxy mode’, not unlike [*WAIL*](https://github.com/N0taN3rd/wail)

        -   and/or Netcapsule old browsers over the web [*http://oldweb.today/*](http://oldweb.today/)

    -   Jupyter notebooks for general analysis.

    -   Secondary (especially non-consumptive) datasets.

    -   Full-text search across all holdings, combining data sources.

The ‘big’ parts are the externalised frontier and the timeline index.
