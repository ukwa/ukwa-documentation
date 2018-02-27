Ingest Tasks Based on HDFS Content
==================================

<!-- MarkdownTOC depth=2 autolink=true bracket=round lowercase_only_ascii=true -->

- [Introduction](#introduction)
- [Overall Workflow](#overall-workflow)
- [Checking & Reporting on Crawled Content](#checking--reporting-on-crawled-content)
- [Ingest to DLS](#ingest-to-dls)
	- [Hashing and ARK minting](#hashing-and-ark-minting)
	- [Pre-packaging preparation](#pre-packaging-preparation)

<!-- /MarkdownTOC -->


Introduction
------------

Crawl processes deposit WARCs and logs onto our HDFS service. These tasks take those files and ensure they are ingested correctly.

Overall Workflow
----------------

The main process of driving content management, based on what's on HDFS, starts with the tasks in
[tasks.ingest.listings](http://ukwa-manage.readthedocs.io/en/latest/source/tasks.ingest.html#module-tasks.ingest.listings). This generates various summaries and reports of the content of HDFS that other tasks use
to updated downstream systems.

As elsewhere, 'listing' task also generated summary metrics that are pushed to Prometheus. These can be used to set up alerts
so that we will notice the tasks are not being run, or if the metrics are not trending as we expect.

Checking & Reporting on Crawled Content
---------------------------------------

The whole process is driven by the [ListAllFilesOnHDFSToLocalFile](http://ukwa-manage.readthedocs.io/en/latest/source/tasks.ingest.html#tasks.ingest.listings.ListAllFilesOnHDFSToLocalFile) task, which runs `hadoop fs -lsr /` and captures the output for the 
other tasks to use.

Notably, [CopyFileListToHDFS](http://ukwa-manage.readthedocs.io/en/latest/source/tasks.ingest.html#tasks.ingest.listings.CopyFileListToHDFS) places a compressed copy of this file on HDFS where [access tasks](./access-indexing.md) can use it to update indexes etc.

Other tasks run checks on subsets of the crawl and verify that the crawl content is laid out as expected, leading to the generation of crawl summaries for every crawl job and launch.

These are then used to populate a simple reporting system, [ukwa-reports](https://github.com/ukwa/ukwa-reports), based on the [Hugo](https://gohugo.io/) static site generator.


Ingest to DLS
-------------

### Hashing and ARK minting

Hashes, possibly lodged by move-to-hdfs?

### Pre-packaging preparation
