The goal is a stable and robust crawl lifecycle, automated end-to-end. The frequent crawl stream should run with no manual intervention, and while the domain crawl may be initiated manually, the ingest and processing of content collected by the crawl should use the same framework.

During long-running processes, failure is expected. Therefore, everything is built to be idempotent, as it may be retried many times. Transient failure are reported but not critical unless persistent.

Overall, we use a small number of larger, modular components (mostly written in Java), and then ‘glue’ them together using Python scripts. Where appropriate, these scripts use the [*Luigi*](https://github.com/spotify/luigi) task framework to help ensure the processes are robust and recoverable. All services are wrapped as Docker containers for deployment.


Upon completion, where relevant, the tasks register execution metrics
with the [monitoring system](Monitoring-Services.md) so we can verify
that these tasks are being run.

All events, across all production systems, are initiated by cron jobs on sh.wa.bl.uk. For example, a cron job on sh.wa.bl.uk may initiate a Luigi task on the the ingest server that scans crawl engines for new content to upload to HDFS.

Anything critical gets put on or backed-up on HDFS

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

