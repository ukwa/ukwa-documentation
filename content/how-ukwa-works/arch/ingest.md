# Ingest Services



## W3ACT

Our curators work with our [W3ACT](https://github.com/ukwa/w3act) (World Wide Web Annotation and Curation Tool), which provide a user interface for:

- Configuring the crawls, including honouring our Non-Print Legal Deposit constraints on crawl scope. 
- A license workflow to ask site owners for an appropriate license to permit us to crawl (if not in scope for Legal Deposit) and for open access (if the site is not already suitable licensed).
- Collecting metadata describing the targeted sites.
- Collecting metadata describing book or journal publications found by the crawl (also known as the Document Harvester) to be submitted to the library's main catalogue systems.

Currently, user accounts in W3ACT are managed separately from any other user account system. It is particularly important to know if users are from a Legal Deposit Library or not, as staff of those organisations are able to review NPLD material.

Some configuration is sensitive, like whether to allow the crawler to ignore robots.txt, so there are different user roles, some of which have more control than others.

## Crawl Feed API

To drive the crawl, we generate lists of crawl jobs to be executed, which look like this:

```
[
    {
        "title": "gov.uk Publications",
        "seeds": [
            "https://www.gov.uk/government/publications"
        ],
        "schedules": [
            {
                "startDate": 1438246800000,
                "frequency": "MONTHLY",
                "endDate": null
            }
        ],
        "ignoreRobotsTxt": false,
        "depth": "DEEP",
        "scope": "root",
        "watched": false,
        "loginPageUrl": null,
        "secretId": null,
        "documentUrlScheme": null,
        "logoutUrl": null,
        "id": 1
    },
    ...
 ```

Crucially, while W3ACT implements a lot of processes specific to the UK Web Archive, the crawler system only needs the data in the crawl feed. Although the configuration options do reflect expectations of the capabilities of the crawler itself, the use of this simple data format interface means that the two components can deployed and updated independently.

## The Frequent Crawler

The 'frequent crawler' implements the curated crawl jobs, and depends only on the crawl job specification shown above. A set of Python scripts are used to manage a set of crawl jobs configured to this specification. The crawling process results in a set of WARC files that capture the data and logs that capture some details about what we did and didn't crawl (e.g. blocked by robots.txt, or by a crawl quota).

## The Domain Crawler

The annual domain crawl is operated separately, and launched separately using lists of known URLs from various sources (including W3ACT records marked 'Domain Crawl Only').  If outputs WARCs and logs in just the same way as the Frequent Crawler.

## The Document Harvester

This additional post-processed step scans the log of what was crawled and looks for things that appear to be publications on sites that W3ACT users have indicated should be scanned for documents.  This post-processor checks the items are available for viewing and sends records to W3ACT for all the documents.  The users then use W3ACT to check the metadata and submit it to downstream British Library systems. The metadata entry system uses pdftohtmlEX to present the PDF to the user while allowing them to select text from the PDF to quickly populate the metadata form.

## W3ACT QA Access

