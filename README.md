UKWA Documentation
==================

Technical documentation for the UK Web Archive. We start with a very high-level summary, then 'zoom in'.

<!-- MarkdownTOC autolink=true bracket=round -->

- [What is the UK Web Archive?](#what-is-the-uk-web-archive)
	- [How does it work?](#how-does-it-work)
	- [Yes, but how does it _work_?](#yes-but-how-does-it-work)
	- [Fine, but _how_ does it work?](#fine-but-how-does-it-work)
		- [Capture](#capture)
			- [Curation](#curation)
			- [The Frequent Crawler](#the-frequent-crawler)
			- [The Domain Crawler](#the-domain-crawler)
		- [Preserve](#preserve)
		- [Access](#access)

<!-- /MarkdownTOC -->

<!--
See the [ukwa-documentation](https://github.com/ukwa/ukwa-documentation#ukwa-documentation) for an overview.
-->


What is the UK Web Archive?
===========================


The UK Web Archive (UKWA) collects millions of websites each year, preserving them for future generations.


How does it work?
-----------------

We enable curators and collaborators to define what we should collect, and how often. We attempt to visit every UK website at least once a year, and for the sites the curators have identified we can visit much more frequently. For example, we collect news sites at least once per day.

We capture the websites using web crawling software, and converting the live sites into static records we can preserve for the future.

We use these records to reconstruct an imperfect facsimile of the original live site, allowing our readers and researchers to browse the UK web as it was in the past. We can also analyse these historical records as data, to extract historical trends, or to build access tools.


Yes, but how does it _work_?
----------------------------

* Capture:
    * Curators provide us with the URLs of interest, and instructions on the scope and limits of the crawl.
    * A web crawler takes the list of curated crawl jobs and executes them.
    * All know UK URLs are also passed to a large domain crawl job once per year.
    * The crawlers produce standardised WARC files and logs that capture what happened.
* Preserve:
    * The WARC files, logs and metadata are kept safe.
* Access:
    * The WARC files are processed to generate suitable indexes so users and tools can locate the information they need.
    * A playback service allows individual pages to be viewed, if the URL is known.
    * A search service allows trends to be explored, and pages to be discovered, based on their content.
    * Datasets summarising facts drawn from the content are made available too.


Fine, but _how_ does it work?
-----------------------------

### Capture

#### Curation 

Our curators work with our [W3ACT](https://github.com/ukwa/w3act) (World Wide Web Annotation and Curation Tool), which provide a user interface for configuring the crawl. This interface is very specific to the UK Web Archive, including our Non-Print Legal Deposit constraints and our additional licensing workflow. This results in a list of crawl jobs to be executed, which look like this:

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


#### The Frequent Crawler

The 'frequent crawler' implements the curated crawl jobs, and depends only on the crawl job specification shown above. A set of Python scripts are used to manage a set of crawl jobs configured to this specification.


#### The Domain Crawler

### Preserve

### Access




