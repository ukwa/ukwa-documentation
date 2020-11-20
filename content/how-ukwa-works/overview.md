# Technical Overview

To get an idea of the overall technical architecture, we won't start with an attempt to enumerate every component involved. First, we'll break it down as a set of larger technical areas, using the names we tend to use when discussing them.

## High-level Architecture

The web archive is composed of a number of distinct groups of services, with clear data standards and APIs between them. This allows each group of services to evolve independently of the others.

Here we present them as a [Wardley map](https://learnwardleymapping.com/home/introduction/), starting with users and the required user capabilities at the top, and moving down the chain of services that fulfil those needs. The horizontal position of each service group gives a rough indication of the maturity and degree of UKWA-specific customisation involved for each group of services.

```{figure} ../maps/ukwa-arch-overview-Overview.svg
---
width: 100%
align: center
name: ukwa-arch-overview
---
A high-level [Wardley map](https://learnwardleymapping.com/home/introduction/) of the main UK services.
```

The dotted lines indicated data standards that connect services via regular updates, and the solid lines indicated live API connections. For example, the crawl services can proceed even if all other services are offline, but need to recieve updated crawl job specifications in order to change what and how we crawl web sites. In contrast, if our storage service is not available, all services that depend on viewing stored data will also break.

Management and Monitoring services are not directly connected to the overall WARC lifecycle, but orchestrate and verify all activities via their external interfaces.

## Information Flows

The _ingest_ information flow, for _Archivists_ and _Curators_, is as follows:

1. The _Archivists_ and _Curators_ use the _Ingest Services_ to configure the crawl parameters that say which sites to crawl, when, and how.
2. The Frequent Crawl system takes the crawl definitions from the _Ingest Services_ and attempts to crawl the specified sites.
3. The Domain Crawl runs once a year and attempts to crawl all UK sites.
4. Both crawler services capture the results as [WARC](https://iipc.github.io/warc-specifications/specifications/warc-format/warc-1.1/) files and log files which are tranferred to our [Hadoop](https://en.wikipedia.org/wiki/Apache_Hadoop) cluster.
5. Indexing tasks run over those WARC files, creating indexes that allow us to find archived URLs.
6. The _Ingest Services_ make all web pages available for inspection via a dedicated playback service known as _QA Wayback_ (using the indexes and the WARC files to replay the archived web pages).
7. The _Curators_ add or update the metadata that describes the content that has been collected. This includes license agreements with website owners to facilitate capture and/or open access to the archived material.

For _Readers_, the overall _access_ flow is:

1. Our _Readers_ come to the _Website_, either directly or via links or metadata in other discovery systems.
2. They can lookup URLs of interest, and play them back via the _Wayback Service_. Note that in library reading rooms they will have full access to the content, whereas the public _Website_ only gives access suitably-licensed material.
3. They can browse the collections of archived pages via the _'Topics & Themes'_ section of the _Website_.
4. They can use full-text faceted search to find URLs of interest based on their content.
5. They can use trend analysis or download datasets in order to understand and analyse the archived web pages.

## Components, Customisation & Maturity

As indicated by the position of the services on the diagram above, the Hadoop cluster is entirely generic, and the WARC files are specific to web archiving, but generic across web archives.

The Wayback service and the indexes are also fairly generic across web archives, although not quite as widely used as the WARC format.

The crawl systems are based on the Internet Archive's [Heritrix3](https://github.com/internetarchive/heritrix3) crawler, but a number of additional modules added to support our specific needs and remit.

The end-user systems are the most heavily custom-built components, with the website itself being the most recent and least mature.

## Development & Deployment

...

The individual components are deployed so that _access_, _ingest_ and _management_ services all run independently.

...
