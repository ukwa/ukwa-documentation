

<!-- MarkdownTOC depth=2 autolink=true bracket=round lowercase_only_ascii=true -->

- [HDFS](#hdfs)
- [Harvest & Ingest](#harvest--ingest)
	- [Moving content to HDFS](#moving-content-to-hdfs)

<!-- /MarkdownTOC -->

HDFS
====

We use folder layout conventions and [user & group permissions](https://hadoop.apache.org/docs/r2.7.1/hadoop-project-dist/hadoop-hdfs/HdfsPermissionsGuide.html) on HDFS to attempt to ensure the following:

 * Original content (like harvested WARCs) that cannot be re-generated is held safely and is very difficult to delete.
 * That non-original but 'expensive to recreate' data (like task state or derived data) is kept in a separate part of the file system from the original content. This data should also be difficult to delete, but doing so is sometimes necessary, so we should ensure the data is managed in a way that prevents any such deletion event delete original content.
 * Transient and temporary end user data should not be easy to delete except for the owning user.

e.g. make an `archive` user that owns any original content, and make groups for the access/license terms? e.g. `npld` group content, etc. Even the user would not have write permission, and there would be no actual 'archive' user, meaning that an explicit `chown` by the hdfs superuser should be required first.

The username that runs the NameNode, 'hdfs', is the superuser for HDFS is the only one that can perform some critical functions, and override all over permissions.

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
