
<!-- MarkdownTOC depth=2 autolink=true bracket=round lowercase_only_ascii=true -->

- [Next Steps in Development](#next-steps-in-development)
    - [Development Work Ideas](#development-work-ideas)
    - [QA](#qa)
    - [Managing Collections](#managing-collections)
    - [Embedding Archived Web Resources](#embedding-archived-web-resources)
    - [Workflows](#workflows)
    - [Wider Issues](#wider-issues)

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

See also [https://trello.com/b/nJCKUtYf/ukwa-next-generation-services](https://trello.com/b/nJCKUtYf/ukwa-next-generation-services)

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
        -   and/or Netcapsule old browsers over the web [http://oldweb.today/](http://oldweb.today/)
    -   Jupyter notebooks for general analysis.
    -   Secondary (especially non-consumptive) datasets.
    -   Full-text search across all holdings, combining data sources.

The ‘big’ parts are the externalised frontier and the timeline index.

QA
--

- Set up Archived Screenshot engine: Puppeteer to render pages against OpenWayback, masked by MITMProxy/SSLStrip. Also notes 404 embeds that are missing and logs these missed embedded URLs in a Kafka queue.
- Also collect NotInArchive logs from QA Wayback, feed to Logstash from Docker logs and push missed URLs onto a Kafka queue (not marked as embeds)
- Set up patch crawl stream to scope the incoming URLs and crawl them.
- Set up a task that periodically requests the screenshots for curated URLs.
    - If possible, measure and report improvements/reduction in 404s over time.
    - If possible, compare with original screenshot too.


Managing Collections
--------------------

Instead of managing curated metadata in our own custom tools, I'd like to use off-the-shelve collection management tools that can be extended to cover web archives.

There are three areas to consider, which are currently all handled in W3ACT, but may be better handled separately.

- Targets: Seeds and scopes and crawl authorisation/licencing.
- Collections: Groups of archived web resources, used for presentation
- Documents: Looking for documents among the archived resources, to add into the main catalogue.

This is not meant to imply that access and curation happen in the same system. The main idea is that we need an interface for curators to manage the data, which may well get published elsewhere. However, it may make sense to use the system for general access too, rather than having a separate website for that.

There is little point in shifting this data our of W3ACT unless it reduces the amount of code we need to maintain and the number of systems we need to manage. Therefore, we wish to consider moving this data into systems that can be supported by third-parties at modest cost, and that are based on open standard and open source so we can avoid vendor lock-in and shop around for support.

Possible engines include [Omeka](http://omeka.org/), Blacklight's [Spotlight](http://spotlight.projectblacklight.org/) and [Drupal](https://www.drupal.org/). There's also [Plone]() which is a but like a Python version of Drupal but requires more back-end coding work. Then there are other tools like Collective Access, OpenRefine, citation managers like Zotero and Bibsonomy, and bookmark services like Pinboard/Delicious and Diigo.

Drupal is very general, very powerful, and widely used. It is quite complex to manage and maintain, so third-party support would be needed, but there are lots of options. It's not 'pre-tuned' to our kind of use cases, so would require a good deal of customisation at first, but is very flexible and could cover a wide range of use cases (e.g. we could probably move Permissions + Targets into it without using custom code, just configuring supported modules). Cloud hosting and support appears to be widespread but may be fairly expensive, but this requires more investigation (e.g. [Pantheon](https://pantheon.io/) & possible [Pantheon EDU](https://pantheon.io/edu), [Acquia Cloud](https://www.acquia.com/gb/products-services/acquia-cloud), [platform.sh](https://platform.sh/)). Only Acquia Cloud includes automatic upgrades for security issues, although [in the past Drupal hosting services have acted quickly to respond to major issues](https://www.drupal.org/hosting-support/2015-04-11/hosts-that-perform-security-updates). At small scale, this is not terribly expensive (around £1,400/year for Acquia Cloud's smallest server, which for curators only is likely fine.)

[Omeka S](https://omeka.org/s/) is less widely used, but seems to match our use cases more closely. There are not as many support options as Drupal, but [hosting and support](http://www.omeka.net/) is available from the lead developers at affordable rates, and [other support options are available)(https://digirati.com/updates/insights/a-crowdsourcing-platform-for-wales/). It seems very well suited to handling modern metadata. e.g. I was able to import a prototype [RDF model for Memento](https://groups.google.com/d/msg/memento-dev/xNHUXDxPxaQ/XQ250Y3AAwAJ) into Omeka easily, and this could be used to clearly mark up our archived web pages and web sites. It can manage multi-lingual data, and also supports [things like FAST and Library of Congress Subject Headings](https://omeka.org/s/modules/ValueSuggest/) using standard modules. The main different with our current model is that Omeka Collections (Item Sets) are not  hierarchical and would be managed as a flat list, although the heirarchy could still be captured via `Has Part` or `Is Part Of` relationships, or by using an additional taxonomy to manage where items appear. If needed, it can be used via Docker too, as shown [here](https://github.com/klokantech/omeka-s-docker). 

To make using it easier, it would make sense to consider adding some kind of helper module to smooth the process of discovering archived resources for inclusion, using Wayback (and perhaps Solr) queries to help pick the timestamps of interest. This could be a kind of 'importer' that pre-populated any necessary fields, e.g. oEmbeds (see below). For example, the Document Harvester workflow could be re-built as a Omeka Importer that queried the Solr service to find documents relating to e.g. 'Watched Sites' Items owned by a particular user (say).

Spotlight is more geared towards highlighting items in a Solr index. This quite suites the Document use case, but the system is less strong on the kind of metadata management required in the other cases. There are also no hosted options and little formal third-party support, so we'd have to take this on in house. This would be worth it if we could much most internal Solr UI work onto Blacklight.

Note that the main challenge in using any of these for the main public website is how to handle the integration of the kind of search-oriented functionality we use in Shine. For example, it may be possible to use Omeka as a end-user collections browser, but then hand over to a dedicated search UI for complex Solr queries.

[Collective Access](http://www.collectiveaccess.org/getting-started) is not dissimilar, but appears to be too keenly focussed on a much broader set of use cases.

An alternative approach would be to drive the whole process from [Zotero](). We could build up shared bibliographies and co-author them using any Zotero-compatible tool. The crawlers could read [feeds like this one](https://api.zotero.org/groups/76372/items/top?start=0&limit=25&format=json&v=1) to ensure we rapidly processed candidate URLs. Managing multi-lingual data in Zotero would be difficult, but otherwise it's a good fit.

OpenRefine is a futher option that may make sense to consider for some use cases.

Similarly, we could integrate with [Bibsonomy](https://bibsonomy.bitbucket.io/), which works rather like Zotero although it's not able to build heirarchical collections. e.g. [my test account is here](https://www.bibsonomy.org/user/anj).

Embedding Archived Web Resources
--------------------------------

We would like to support common methods for embedding archived web pages in other web media, along the lines of that discussed in [this blog post about storytelling with web archives](http://ws-dl.blogspot.co.uk/2017/08/2017-08-11-where-can-we-post-stories.html). To this end, our Collections and our Archived Web Sites and our Archived Web Pages (OpenWayback) should ideally expose social card data. e.g. [Twitter Cards](https://developer.twitter.com/en/docs/tweets/optimize-with-cards/guides/getting-started), Open Graph etc. 

Beyond social media, there is a [oEmbed](https://oembed.com/) standard that makes it easier to embed content in blogs and other engines. We could expose this service, and also register it with e.g. [Embedly](http://embed.ly/providers/new) so more people can use it. The way Twitter itself support [embedded tweets](https://dev.twitter.com/web/embedded-tweets) and [oEmbed](https://developer.twitter.com/en/docs/tweets/post-and-engage/api-reference/get-statuses-oembed) may help us understand how to achieve the same for archived web pages.

For example, we could create a `rich` embed HTML snippet which includes basic information about an archived resource, overlayed on a screenshot (where possible). Clicking would take you through to the Wayback playback, much like any other media embed.

Both [Omeka](https://omeka.org/s/docs/user-manual/content/media/#add-media) and [Drupal](https://www.drupal.org/project/url_embed) support oEmbed, so this standard could also be used to simplify our own systems.

Workflows
---------

- Website & Web Page Collection Management (very good fit for Omeka, but Drupal could be extended to cover it)
- Document Harvesting, and related grey literature like company reports (could be an Omeka importer, or in Drupal, but 'MEX' functionality would need some work and it may be better as a standalone component?)
- Crawl Targets (seeds and scopes etc.) including permissions. (Clumsy in Omeka, better fit for Drupal -- like the original ACT!)
- Crawl Target Licensing Process (Could be done in Drupal, likely using standard modules.)
- Site Nomination (all automatically-in-scope URLs crawled immediately. All nominations recorded for review. Pretty straightforward in Drupal).

Done right, the collections manager could also replace the basic browsing part of the new website engine. We would switch over to the new engine for search, but collections browsing could be done elsewhere.


Wider Issues
------------

Security: A couple of proof-of-concept publications have shown that web archives can be 'edited' retrospectively, so we need to take that onboard.

How we are tackling bias is not expressed explicitly? Transparency, unthinking machines, complementing information professional approach.

User comments/feedback on collections useful for understanding how we need to document things.

BREXIT: 

NPLD: Agreed: Equality of accessibility, digital facsimile for newsprint magazine. Plus two documents of disagreement. They are suggesting a consulation process about the web archives. Publishers, percentage of content disputed.  Maybe sneak some less-controversial stuff through pre-Brexit, but proposal is a further consulation on access and/or changes on regulations which means post-Brexit.


Services to Research Community where is makes sense. 

Content Strategy: Led by Sally Halper. Document Harvester mentioned as expected. Web archive as source of intellegence on publishing in the UK. Alongside other contemporary British collections. Much digital content within there too. e.g. internally host as web sites we index? Or by web team. Problem is exposing them as web sites really. Sound Archive hardware repository. 

Preservation platform idea, news across different media, other areas.

Everything Available/Research Services: Building services for audience sectors. Web archiving OA comes up here. EThOS. Repository service.

Labs using Student Passes to allow researchers to VPN in to actual get stuff done.

Sources for growing UK domain. Data sets like open corporates, Wikipedia, companies house. 

Others wanting to wrap content we hold. Something like loaning a collection item. Embed specific archived targets in others sites. oEmbed.

Desire to have access to WARCs for a specific site. (Consider splitting upstream? Switch to mutltiple jobs?) Could be content owner or collection dataset.

End users: Create their own collection. Mix in Internet Archive material where we can't archive it. "Flickr for archive websites." Supply the narrative too.

Google/search engine indexing of Collection sites might help surface stuff.

More time improving search.

Wikipedia, WikiCite, WikiData.

Research user forum?





