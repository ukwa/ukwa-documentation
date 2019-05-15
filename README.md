UKWA Documentation
==================

Technical documentation for the UK Web Archive. We start with a very high-level summary, then 'zoom in'.

<!-- MarkdownTOC autolink=true bracket=round -->

- [What is the UK Web Archive?](#what-is-the-uk-web-archive)
	- [How does it work?](#how-does-it-work)
	- [Yes, but how does it _work_?](#yes-but-how-does-it-work)
	- [Fine, but _how_ does it work?](#fine-but-how-does-it-work)
		- [Capture](#capture)
			- [Curation](#curation)
			- [The Frequent Crawler](#the-frequent-crawler)
			- [The Domain Crawler](#the-domain-crawler)
		- [Preserve](#preserve)
		- [Access](#access)
	- [Introduction](#introduction)
	- [System Overview](#system-overview)
	- [Services & Repositories](#services--repositories)
		- [Services](#services)
		- [Management](#management)
		- [Monitoring](#monitoring)
		- [Major Components & Interfaces](#major-components--interfaces)

<!-- /MarkdownTOC -->

<!--
See the [ukwa-documentation](https://github.com/ukwa/ukwa-documentation#ukwa-documentation) for an overview.
-->


What is the UK Web Archive?
===========================


The UK Web Archive (UKWA) collects millions of websites each year, preserving them for future generations.


How does it work?
-----------------

We enable curators and collaborators to define what we should collect, and how often. We attempt to visit every UK website at least once a year, and for the sites the curators have identified we can visit much more frequently. For example, we collect news sites at least once per day.

We capture the websites using web crawling software, and converting the live sites into static records we can preserve for the future.

We use these records to reconstruct an imperfect facsimile of the original live site, allowing our readers and researchers to browse the UK web as it was in the past. We can also analyse these historical records as data, to extract historical trends, or to build access tools.


Yes, but how does it _work_?
----------------------------

* Capture:
    * Curators provide us with the URLs of interest, and instructions on the scope and limits of the crawl.
    * A web crawler takes the list of curated crawl jobs and executes them.
    * All know UK URLs are also passed to a large domain crawl job once per year.
    * The crawlers produce standardised WARC files and logs that capture what happened.
* Preserve:
    * The WARC files, logs and metadata are kept safe.
* Access:
    * The WARC files are processed to generate suitable indexes so users and tools can locate the information they need.
    * A playback service allows individual pages to be viewed, if the URL is known.
    * A search service allows trends to be explored, and pages to be discovered, based on their content.
    * Datasets summarising facts drawn from the content are made available too.


Fine, but _how_ does it work?
-----------------------------

### Capture

#### Curation 

Our curators work with our [W3ACT](https://github.com/ukwa/w3act) (World Wide Web Annotation and Curation Tool), which provide a user interface for configuring the crawl. This interface is very specific to the UK Web Archive, including our Non-Print Legal Deposit constraints and our additional licensing workflow. This results in a list of crawl jobs to be executed, which look like this:

```
[
    {
        "title": "gov.uk Publications",
        "seeds": [
            "https://www.gov.uk/government/publications"
        ],
        "schedules": [
            {
                "startDate": 1438246800000,
                "frequency": "MONTHLY",
                "endDate": null
            }
        ],
        "ignoreRobotsTxt": false,
        "depth": "DEEP",
        "scope": "root",
        "watched": false,
        "loginPageUrl": null,
        "secretId": null,
        "documentUrlScheme": null,
        "logoutUrl": null,
        "id": 1
    },
    ...
 ```


#### The Frequent Crawler

The 'frequent crawler' implements the curated crawl jobs, and depends only on the crawl job specification shown above. A set of Python scripts are used to manage a set of crawl jobs configured to this specification.


#### The Domain Crawler

### Preserve

### Access



Introduction
------------

This document outlines the current and near-future web archiving system. It is intended to give an introduction to the major components and tie together the different source code repositories that are involved.


System Overview
---------------

High-level workflows are described here - more detailed documentation on individual tasks will be available [here](http://ukwa-shepherd.readthedocs.io/en/latest/).

![High-level Architectural Overview](./drawings/ng-was-phase-3.jpg)

The goal is a stable and robust crawl lifecycle, automated end-to-end. The frequent crawl stream should run with no manual intervention, and while the domain crawl may be initiated manually, the ingest and processing of content collected by the crawl should use the same framework.

During long-running processes, failure is expected. Therefore, everything is built to be idempotent, as it may be retried many times. Transient failure are reported but not critical unless persistent.

Overall, we use a small number of larger, modular components (mostly written in Java), and then ‘glue’ them together using Python scripts. Where appropriate, these scripts use the [*Luigi*](https://github.com/spotify/luigi) task framework to help ensure the processes are robust and recoverable. All services are wrapped as Docker containers for deployment.

On the *ingest* side, we can make use of any crawl engine that:

* Can consume a list of seeds and honor an appropriate scope:
    * Either the 'crawl feed' from W3ACT that defines our frequent crawls (seeds, caps, crawl launch dates and frequencies, etc.)
    * Or the domain crawl seed list, with a scope that permits GeoIP lookup as well as U.K. domain scoping.
* Emits the results in the form we require:
    * A set of WARC files, and associated logs.
    * In a standard folder structure reflecting the crawl stream and the crawl launch date.
    * Along with any necessary metadata e.g. crawl configuration files, usually packaged as a ZIP.

This material is cached on HDFS, from where it can be indexed for access and also wrapped for submission to the DLS. All critical data is stored here as simple files in the standard folder structure, and all irreplaceable data should be store in the same way. All downstream processes depend only upon the state of the HDFS system (either directly or in faster, cached forms). i.e. ideally there should be no coupling between the *ingest* and *access* sides (a notable exception is the Q.A. Wayback service exposed via W3ACT, which shares the same back-end as the reader access systems in order to avoid unexpected differences between the two).

Hadoop jobs are used to index the WARC files for access and search, and WARC records can be streamed from the HDFS storage backend as required. The indexes and HDFS provide the primary APIs that power the front-end services.

All *ingest* and *access* tasks are launched via cron from a central service (known as `wash`), but each is scheduled by a dedicated instance of the Python Luigi Daemon. For example, a cron job on `wash` may initiate a Luigi task on the the *ingest* server that scans crawl engines for new content to upload to HDFS.

An entirely independent *monitor* layer runs it's own tasks to inspect the status of the services and the various data stores, and stores the results in ElasticSearch for inspection in Grafana and/or Kibana. We recognise the [monitoring and QA are the same thing](https://plus.google.com/+RipRowan/posts/eVeouesvaVX) and aim to raise the bar for automated monitoring an QA of web archiving processes.

* [Design Principles](Design-Principles.md)
* [System State Management](System-State-Management.md)
* [Future Development](Future-Development.md)

Services & Repositories
-----------------------

<table width="100%" style="text-align: center; display: table; border-collapse: collapse;">
	<thead width="100%">
	    <tr><th></th><th>Ingest</th><th>Storage</th><th>Access</th></tr>
    </thead>
    <tbody width="100%">
		<tr>
			<th rowspan="2">Services</th>
			<td><a href="https://github.com/ukwa/ukwa-ingest-services"><i>ukwa-ingest-services</i></a></td>
			<td>HDFS, OutbackCDX</td>
			<td><a href="https://github.com/ukwa/ukwa-access-services"><i>ukwa-access-services</i></a></td>
		</tr>
		<tr>
			<td>On Docker</td>
			<td>Native</td>
			<td>On Docker</td>
		</tr>
		<tr>
			<th rowspan="2">Management</th>
			<td colspan="3" align="center"><a href="https://github.com/spotify/luigi"><i>Python Luigi</i></a> tasks defined in <a href="https://github.com/ukwa/ukwa-manage"><i>ukwa-manage</i></a></td>
		</tr>
		<tr>
			<td><i>Ingest Task Scheduler</i></td>
			<td></td>
			<td><i>Access Task Scheduler</i></td>
		</tr>
		<tr>
			<th>Monitoring Services</th>
			<td colspan="3" align="center"><a href="https://github.com/ukwa/ukwa-monitor"><i>ukwa-monitor</i></a></td>
		</tr>
	</tbody>
</table>

There are three major areas to cover: Services, Management & Monitoring. The Services are the main components of the crawl system, usually deployed as Docker containers. The Management system coordinates the tasks that act upon the services, in order to automate whole content management life-cycle. The Monitoring system runs independently of the Management system, but automates checks that help ensure the whole system is running correctly.

### Services ###

There are two main sets of services, each of which has one or more [*Docker Compose*](https://docs.docker.com/compose/) files to define different deployments, from local development to full production.

When developing, the system should be run via the `docker-compose` command, and using the `docker-compose.yml` file held in the root directory of the component that you are working on.

When deploying service into production (or into a formal beta/alpha/test environment), separate orchestration files should be used (currently Docker Swarm compose files held in the following repositories:

- [*ukwa-ingest-services*](https://github.com/ukwa/ukwa-ingest-services) with separate Docker Compose definitions for the crawl engine (Heritrix3 etc.) and for the frontend services (e.g. W3ACT etc.) 
- [*ukwa-access-services*](https://github.com/ukwa/ukwa-access-services) which covers end-user services like our website, APIs etc.

See [Development Walkthrough](Development-Walkthrough.md) for more details.

### Management ###

The [*ukwa-manage*](https://github.com/ukwa/ukwa-manage) code base contains all the ‘glue’ code that orchestrates our content life-cycle management, implemented as [*Python Luigi*](https://github.com/spotify/luigi) tasks.

All events, across all production systems, are initiated by cron jobs on `wash`. For example, a cron job on `wash` may initiate a Luigi task on the the `ingest` server that scans crawl engines for new content to upload to HDFS. Both the `ingest` and `access` servers have a copy of `python-shepherd` installed, and each runs their own LuigiD task scheduler (which provides a UI and manages task execution).

Some low-level documentation can be found at http://ukwa-manage.readthedocs.io/ but the overall workflows and usage are described here:

* Ingest:
    * [HDFS Content Workflows](./workflows/ingest-listings.md)
* Access
    * [Indexing Workflows](./workflows/access-indexing.md)
* [Development](Development.md)
    * [Working With Luigi](Working-With-Luigi.md)

### Monitoring ###

The [*ukwa-monitor*](https://github.com/ukwa/ukwa-monitor) codebase monitors and reports on all UKWA processes, including ingest. This provides a dashboard and raises alerts by probing the production system. It runs independendly, inspecting the state of the system and reporting on it. If it's down, nothing else should be affected. If any part of the production system is down, or if important tasks fail to run, the monitoring system should raise the alert.

To allow the alerts to be handled, the monitoring system will also hold any useful debugging information to help direct the web archiving team to the problematic service component.

For more details, see [Monitoring Services](Monitoring-Services.md).

### Major Components & Interfaces ###

The curation front end, the frequent crawl engine, the shepherd, the domain crawler, the storage, the indexes, the access services.

- Developed by us and/or IIPC members:
    - [*W3ACT*](https://github.com/ukwa/w3act) (UKWA-only)
    - [*Heritrix3* with UKWA extensions](https://github.com/ukwa/ukwa-heritrix)
    - [*warcprox*](https://github.com/internetarchive/warcprox) (Python)
    - [*OutbackCDX*](https://github.com/nla/outbackcdx) (previously known as *tinycdxserver*)
    - [*OpenWayback*](https://github.com/iipc/openwayback)
    - [*webarchive-discovery*](https://github.com/ukwa/webarchive-discovery)
    - The UKWA Website engine [*ukwa/ukwa-ui*](https://github.com/ukwa/ukwa-ui)



