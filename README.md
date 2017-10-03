UKWA Documentation
==================

Technical documentation for the UK Web Archive.

<!-- MarkdownTOC depth=2 autolink=true bracket=round lowercase_only_ascii=true -->

- [Introduction](#introduction)
	- [Services](#services)
	- [Management](#management)
	- [Monitoring](#monitoring)
	- [Major Components & Interfaces](#major-components--interfaces)
- [Life-cycle Overview](#life-cycle-overview)
	- [Harvesting Workflow](#harvesting-workflow)
- [Further Information](#further-information)

<!-- /MarkdownTOC -->


Introduction
------------

This document outlines the current and near-future web archiving system. It is intended to give an introduction to the major components and tie together the different source code repositories that are involved.

There are three major areas to cover: Services, Management & Monitoring. The Services are the main components of the crawl system, often deployed as Docker containers. The Management system coordinates the tasks that act upon the services, in order to automate whole content management life-cycle. The Monitoring system runs independently of the Management system, but automates checks that help ensure the whole system is running correctly.

### Services ###

There are two main sets of services, each of which has one or more [*Docker Compose*](https://docs.docker.com/compose/) files to define different deployments, from local development to full production.

- [*pulse*](https://github.com/ukwa/pulse) (rename to ukwa-ingest-services?) with separate Docker Compose definitions for the crawl engine (Heritrix3 etc.) and for the frontend services (e.g. W3ACT etc.) 
- [*ukwa-access-services*](https://github.com/ukwa/ukwa-access-services) which covers end-user services like our website, APIs etc.

### Management ###

The [*python-shepherd*](https://github.com/ukwa/python-shepherd) code base (rename to ukwa-manage?) contains all the ‘glue’ code that orchestrates our content life-cycle management, implemented as [*Python Luigi*](https://github.com/spotify/luigi) tasks.

All events, across all production systems, are initiated by cron jobs on `wash`. For example, a cron job on `wash` may initiate a Luigi task on the the `ingest` server that scans crawl engines for new content to upload to HDFS. Both the `ingest` and `access` servers have a copy of `python-shepherd` installed, and each runs their own LuigiD task scheduler (which provides a UI and manages task execution).

n.b. we may split the management codebase into a library and three separate task sets (for ingest, storage and access) in the future.

### Monitoring ###

The [*ukwa-monitor*](https://github.com/ukwa/ukwa-monitor) codebase monitors and reports on all UKWA processes, including ingest. This provides HTML reports, a dashboard, and raises alerts by probing the production system. It is also based on Luigi tasks, but no part of any production system depends on it, and it runs it’s own cron jobs (rather than being initiated by `wash`) and it's own LuigiD scheduler. If it's down, nothing else should be affected. 

The monitoring tasks check that the necessary services are running, and that critical management tasks have been executed successfully. If services are down, or transient failures become persistant, it is the role of the monitor engine to raise the alert.

### Major Components & Interfaces ###

The curation front end, the frequent crawl engine, the shepherd, the domain crawler, the storage, the indexes, the access services.


Life-cycle Overview
-------------------

High-level workflows are described here - more detailed documentation on individual tasks will be available [here](http://ukwa-shepherd.readthedocs.io/en/latest/) one day!

### Harvesting Workflow ###

n.b. * UNDER DEVELOPMENT *

| Step              | Description   |
| ----------------- | ------------- |
| Start/stop crawls |  |
| Move to HDFS      |  |
| Analyse           | As content appears on HDFS, we first need to identify the ‘chunks of crawl’ described above. We start this by scanning log files. For each new set of logs, we scan for associated WARC files, potential documents, run basic stats analysis. |
| Verify            | This would be the logical spot to run the Map-Reduce hasher and double-check the hashes are as expected, at which point we can clear the files on the crawl servers for deletion. |
| Assemble          | We take the initial chunks of logs and warc and assemble them into a single description that also contains any required metadata in associated ZIP files. This also contains the SHA-512 of the files and any necessary ARK identifiers. The main result is a JSON description of the crawl at a given point in time. Any crawl process that generates these can be plumbed into the downstream processes. |
| Package & Submit  | Take each chunk and assemble SIPs incrementally, Submit them to DLS |
| Validate          | We check the DLS export, and we check against our DLS Access Point. |
| Analysis & Indexing | ...following on from Assembly, we can now start to process chunks of crawl for access. See [*2017 Web Archive Search Strategy*](http://drive.google.com/open?id=1CJUvyI1XPOZt6oEl_2oFRJHXrv8K3_jT36AkOSEcjzw) |


Further Information
-------------------

* [[Design Principles]]
* [[System State Management]]
* [[Deployment]]
* [[Development]]
    * [[Working With Luigi]]
* [[Future Development]]
