Indexing for Access
===================

<!-- MarkdownTOC depth=2 autolink=true bracket=round lowercase_only_ascii=true -->

- [Listing WARCs by date](#listing-warcs-by-date)
- [CDX Indexing \(with verification\)](#cdx-indexing-with-verification)
- [Solr Indexing \(with verification\)](#solr-indexing-with-verification)

<!-- /MarkdownTOC -->

Listing WARCs by date
---------------------

The HDFS listing process generates a summary of every WARC file, grouped by the day the file was created. As subsequent
scans can add more files (due to the time it can take to copy content up from the crawlers), we need to make sure the
file manifests are update in a way that ensures this gets noticed. We do this by including the number of files in the
file name itself.

The general naming scheme is:

    /var/state/warcs/<YYYY>-<MM>/<YYYY>-<MM>-<DD>-warcs-<NNNN>-warc-files-for-date.txt

e.g.

    /var/state/warcs/2018-02/2018-02-20-warcs-57-warc-files-for-date.txt

Which lists the 57 WARCs known to be associated with that date at the time the list was created. If subsequent runs
discover new WARCs, new files will be created...

    /var/state/warcs/2018-02/2018-02-20-warcs-68-warc-files-for-date.txt

n.b. using just the file number is simple, but assumes WARCs will not be removed from the cluster. It may be
possible to avoid this by using e.g. the truncated hash of all the warc file names


The downstream processing treats each of the above inventory files as ExternalFiles. For every date, it attempts to
ensure that every file set is processed, and that the presence of each WARC file's content in the CDX index is verified.

CDX Indexing (with verification)
--------------------------------

CDX indexing is run as follows:

```bash
# Run daily task, back 10 days:
END_DATE=`date "+%Y-%m-%d"`
luigi --module tasks.access.index RangeDailyBase --of CdxIndexAndVerify --days-back 10 --stop $END_DATE --reverse --workers 20
```

This uses a [built-in feature of Luigi intended to simplify running recurring tasks](http://luigi.readthedocs.io/en/stable/luigi_patterns.html#triggering-recurring-tasks),
where the `RangeDailyBase` helper task checks that our `CdxIndexAndVerify`
task has been run every day for the preceding ten days.
Note that the `--stop` end date is *exclusive*, so this will run
yesterday's task but not today's. We also specify the allowed number of
parallel worker processes allowed -- in this case, 20.

The [access.index.CdxIndexAndVerify](http://ukwa-manage.readthedocs.io/en/latest/source/tasks.access.html#tasks.access.index.CdxIndexAndVerify)
task chain is driven by a set of files that enumerate the WARC files
associated with each day of crawling. These files are re-generated
daily based on the contents of HDFS, and the output filenames are
designed so that they change when new content appears for a given day.

For every day with new content, the `CdxIndexAndVerify` task will launch a
[CdxIndexer](http://ukwa-manage.readthedocs.io/en/latest/source/tasks.access.html#tasks.access.index.CdxIndexer)
Hadoop Map-Reduce job that will parse the WARCs and submit the URLs to
our OutbackCDX service. But, before marking the task as complete, the `CdxIndexAndVerify`
task launches a [CheckCdxIndex](http://ukwa-manage.readthedocs.io/en/latest/source/tasks.access.html#tasks.access.index.CheckCdxIndex)
task that extracts a sample of URLs from that days WARCs, and checks they
are all present in OutbackCDX.

Once indexed and verified, the indexing process for this number of WARCs and
for this day is marked ask done.

Upon completion, where relevant, the tasks register execution metrics
with the [monitoring system](../Monitoring-Services.md) so we can verify
that these tasks are being run.

Solr Indexing (with verification)
--------------------------------

*TBA*


