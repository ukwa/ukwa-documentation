Fine, but _how_ does it _work_?
-------------------------------

We manage the Capture, Preserve and Access phases as separate, loosely-coupled processes with clear interfaces.

### Capture

#### Curation 

Our curators work with our [W3ACT](https://github.com/ukwa/w3act) (World Wide Web Annotation and Curation Tool), which provide a user interface for configuring the crawl. This interface is very specific to the UK Web Archive, including our Non-Print Legal Deposit constraints and our additional licensing workflow. This results in a lists of crawl jobs to be executed, which look like this:

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

Crucially, while W3ACT implements a lot of processes specific to the UK Web Archive, this only affects the contents of the crawl job specification. The specification format itself is more generic, although the configuration options to reflect expectations of the capabilities of the crawler itself.

#### The Frequent Crawler

The 'frequent crawler' implements the curated crawl jobs, and depends only on the crawl job specification shown above. A set of Python scripts are used to manage a set of crawl jobs configured to this specification. The crawling process results in a set of WARC files that capture the data and logs that capture some details about what we did and didn't crawl (e.g. blocked by robots.txt, or by a crawl quota).

#### The Domain Crawler

The annual domain crawl is operated separately, and launched separately using lists of known URLs from various sources (including W3ACT records marked 'Domain Crawl Only').  If outputs WARCs and logs in just the same way as the Frequent Crawler.

### Preserve

We keep the WARCs and log files from all the crawlers, in a folder structure that reflects the crawl process. We keep content obtained under different terms in separate folders. For example, openly available Non-Print Legal Deposit material is separate from material that we obtained by permission, or from behind a paywall.

### Access

Combining the WARCs and logs with metadata from curators, we offer various types of access.

#### Playback

The primary access mode is via a Playback service. This relies on readers knowing which URLs they wish to access, and a Content Index (or CDX) service is used to look up which WARC records hold copies of each URL, and from what times.

#### Browse & Search

We take a range of steps to provide useful access to our collections for users who do not know which specific URLs they need. We use W3ACT to gather archived material into a set of browsable collections, and create a full-text index so the contents of the archive can be searched. This index can also be used to explore trends and perform some basic analyses.

#### Datasets, Statistical Reports & APIs

We also work with researchers to provide data sets that summarise some factual aspects of the archived resources. For example, in the past we have processed our WARCs to analyse links between web sites, and how they change over time. The same infrastructure can be used to generate statistical reports.

Where appropriate, we also offer API access to information that it better queried than downloaded. For example our Playback service offers the Memento API, which allows others to query our holdings without downloading some massive dataset.


