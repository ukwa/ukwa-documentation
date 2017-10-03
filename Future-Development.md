
<!-- MarkdownTOC depth=2 autolink=true bracket=round lowercase_only_ascii=true -->

- [Next Steps in Development](#next-steps-in-development)
    - [Development Work Ideas](#development-work-ideas)

<!-- /MarkdownTOC -->

Next Steps in Development
=========================

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
