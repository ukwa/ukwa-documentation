Roadmap
=======

Ingest NG Phase 2
------------------

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
