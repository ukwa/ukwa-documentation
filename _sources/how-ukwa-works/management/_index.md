# Management

The UK Web Archive is a fixed-resource cross-organisational service delivery team, funded by the UK Legal Deposit Libraries. We are responsible for both the day-to-day operation of the service, and it's continuous improvement over time.

We are 

## Priorities

When there is so much to be done, and especially dealing with unexpected and urgent issues, it is important that we all understand the overall priorities of the different types of work we undertake.

Generally, operational issues take precedence over continuous improvement. Within that overall framework, our services are prioritised as follows:

1. __Crawling:__ We cannot preserve what we fail to capture in the first place. Ensuring our regular and domain crawls are running as expected is our highest priority end-user service.
   - Frequent Crawls
   - Domain Crawl (if running)
   - Supporting services like Gluster.
   - Automation platform, and automated tasks like launches, tidying, etc.
   - Monitoring
2. __Long-Term Storage:__ Crawled material needs to be transferred to HDFS promptly for safe keeping.
   - HDFS itself and capacity management.
   - Operation and monitoring of HDFS and `move-to-hdfs` processes.
   - Backup of live service databases to HDFS.
   - Replication of HDFS to a separate location (NLS).
   - TrackDB (which is used to track indexing status etc.) ???
3. __Ingest Services:__ So curators, archivists and parters can define what gets crawled and how it's described.
   - W3ACT
   - QA Wayback
   - Document Harvester
4. __Access Services:__ So readers and researchers can access and use our collections.
   - The website
   - Public Wayback
   - Full-text Search
5. __Improving Crawl Quality:__ Areas for improvement include:
   - Improved automation
   - Improved reporting 
   - Automated verification & QA
6. __Improving Ingest Services:__ Areas for improvement include:
   - W3ACT fixes and features
   - Document Harvester improvements
   - Integrated reporting and analysis
7. __Improving Access Services:__ Areas for improvement include:
   - New website features
   - Updated version of Wayback
   - APIs and datasets

This is a service-oriented perspective. Of course, each service area has its own underlying infrastructure, like servers and networks.

## Procedures 

```{tableofcontents}
```