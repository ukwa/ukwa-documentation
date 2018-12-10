Roadmap (reverse chronological milestones)
=======

Ingest v.4 Phase 2
------------------

### Ingest Orchestration 

#### Goals

- Clean-up Heritrix management tools, moving all to https://github.com/ukwa/hapy
- Tidy up what was python-shepherd, now called https://github.com/ukwa/ukwa-manage
- Move Docker Compose service deployment configurations into https://github.com/ukwa/ukwa-ingest-services
    - Consider deploying our [HttpFS varient](https://github.com/ukwa/httpfs) in this way too.

### Crawl Engine

See [ukwa-heritrix](https://github.com/ukwa/ukwa-heritrix).

#### Goals

- Improved robustness and monitorability. 
- Extraction of 'already seen' and 'persist log' state into an external service. 
- Ability to rebuild the frontier easily. 
- Dynamic crawl workflow where individual targets launch when needed.  
- Permit elastic Heritrix3 deployment.

#### Strategy:

Route scoped ('to crawl') URIs via external Kafka queue, which can also be used to launch crawls. 
This become a full log of requested crawls, which can be used to re-build the frontier by re-consuming the entire queue. 
To improve deduplicate handling and to control re-crawling frequency, we also use OutbackCDX as an external register of 
what has happened in the crawl so far. This is also used to discard fulfilled crawl requests when re-populating the frontier.

This one routes all in-scope URIs via a Kafka queue, which can be used to reconstruct the frontier in the event of failure.


#### To Be Considered:

- Consider adding a recently-crawled check before dispatching to the `to-crawl` queue, to avoid overstuffing queues.
- Sketch out who to stop/start many instances and launch/unpause/pause/terminate jobs, consider pulling Heritrix3 Python management library in to here as CLI utils to be called from outside. (see [the H3 API Guide](https://webarchive.jira.com/wiki/spaces/Heritrix/pages/5735014/Heritrix+3.x+API+Guide)). Move most H3 code out of ukwa-manage and into [our python-heritrix fork](https://github.com/ukwa/python-heritrix.git)
- this should include extracting status information for monitoring, running helper scripts, etc.
- Add sheets etc back in and ensure sheet associations and re-crawl frequencies can be set appropriately from the messages themselves. 
- look at using full scopes from W3ACT rather than 'growing' scopes.
- consider how best to generate the surt TTL mapping.
- consider separating discovered/out of scope/to crawl streams
- design streams for different npld/pw/by Content
- key on host, and in the Python client too (check crawl key is host for us, consider using that)
- document and provide examples of keyed messages https://varunksaini.com/blog/send-key-value-messages-to-kafka-from-console-producer/
- check setting stats to zero is working 
- refine OutbackCDX Recently Seen lookup to query for just the most recent matches rather than all, to avoid filtering. [DONE]
- think about how to plumb in crawler restart logic when pulling from Kafka. Should it be 'earliest' by default? How to reset when we need to recover fully? [DONE]
- write up spark/storm comparison e.g. https://mapr.com/blog/stream-processing-everywhere-what-use/ - move Storm POC work into a suitable place and document how we could shift some processing to Storm and some reporting to Spark
- document idea of switching from OutbackCDX to HBase as this may improve parallel performance and may make ad hoc queries easier (rather than relying on crawl logs)
- explore the idea of directly exposing the Kafka streams, using filtering to move backward and forwards through what a crawl did.
- Look at integrating OutbackCDX in warcprox, following this approach: https://github.com/internetarchive/warcprox/pull/37/files
- Look at publishing crawled URIs from warcprox, following this approach: https://github.com/internetarchive/warcprox/pull/32
- Instead of [Trifecta](https://github.com/ldaniels528/trifecta), use [this nice Kafka UI](https://github.com/Landoop/kafka-topics-ui) although this requires [Kafka REST](https://github.com/confluentinc/kafka-rest) as well (although there is a [Kafka REST docker image](https://hub.docker.com/r/confluentinc/cp-kafka-rest/))
- Clean up our ukwa-manage/LuigiD container along [these lines](https://github.com/pysysops/docker-luigid/blob/master/Dockerfile)

Ingest v.4 Phase 1
------------------

### Ingest Orchestration 

#### Goals

#### Strategy:


### Crawl Engine

#### Goals

#### Strategy:

Ingest v. 3
-----------
Heritrix (pulse)

Ingest v. 2
-----------
WCT + H3 (DC)

Ingest v. 1
-----------

WCT
