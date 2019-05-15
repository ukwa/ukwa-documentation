Roadmap (reverse chronological milestones)
=======

Web Rendering & Automated QA
----------------------------

The idea is to execute this sequence:

- Set up proxy and headers for all requests.
- Go to the URL.
- When we hit onReady, capture the image etc.
- Execute any Memento Tracer-style navigation assistance scripts.
- Extract the navigation URLs and enqueue them for crawl.
- (etc.)

n.b. we also need to clear the cache.

We currently use PhantomJS via warcprox to capture seeds. We capture:

- onreadydom
- screenshot
- thumbnail
- imagemap
- har

We want to use these to auto-evaluate crawl quality, by also screenshotting the archived page in proxy mode. We also want to align this work with the Memento Tracer project, and also deal with the fact that PhantomJS is EOL. Ideally, we want the screenshot-of-archived-pages to be a generic service that is exposes an oEmbed API.  See e.g. http://micawber.readthedocs.io/en/latest/

One option, which would improve compatibility with the Memento Tracer project, would be to move all of this to the Selenium WebDriver API. The problem here is that both the crawl-time render and the playback-time render required us to set additional headers (`WARCProx-Meta` to name WARC files on capture, and `Accept-Datetime` to select Mementos during playback).  This is not directly supported by the WebDriver API, and appears to be considered out of scope, as they recommend using an additional proxy (specifically BrowserMob) to modify headers. This is very clumsy for us, as we wish to modify the headers per-render, and so each render process would need to run a separate proxy. This is not something SeleniumGrid appears to support.

It is doable if you use PhantomJS as the Selenium back-end, because PhantomJS has a 'custom headers' API. But as PhantomJS is EOL, we need to move to Chrome and/or Firefox, and getting them to work in the same way may require us developing a web browser extension that somehow exposed the ability to modify the headers.  It's not clear if this would work, and it's depends on techologies our team has little experience with.

However, this may not be necessary. The Google Chrome Puppeteer documentation indicates that they do provide an [API for adding extra HTTP headers](https://github.com/GoogleChrome/puppeteer/blob/master/docs/api.md#pagesetextrahttpheadersheaders). We could use Puppeteer (or it [Python alternative Pyppeteer](https://pypi.org/project/pyppeteer/)) and set the headers as needed.

So, the remaining question is, can we talk to a Chrome instance, hooked up via Selenium, so we can use the [Chrome DevTools Protocol](https://chromedevtools.github.io/devtools-protocol/) to [set headers](https://chromedevtools.github.io/devtools-protocol/tot/Network#method-setExtraHTTPHeaders) and then execute the Selenium script.

I guess this should work. If we instanciate the Chrome instance in pyppeteer, then we can presumable execute any specific setup at that point, and then pass the instance (i.e. the remote debugging port) over to Selenium. Some Selenium stuff might not work if we do this [because they rely on a custom module that ChromeDriver installs](https://sites.google.com/a/chromium.org/chromedriver/help/operation-not-supported-when-using-remote-debugging), but I expect that would probably not affect the kinds of navigation-assist logic that is the main focus of the Tracer work.

Here's an example from the [ChromeDriver docs](https://sites.google.com/a/chromium.org/chromedriver/getting-started):

```java
WebDriver driver = new RemoteWebDriver("http://127.0.0.1:9515", DesiredCapabilities.chrome());
driver.get("http://www.google.com");
```

or [for Python](https://selenium-python.readthedocs.io/getting-started.html#using-selenium-with-remote-webdriver):

```python
from selenium import webdriver
from selenium.webdriver.common.desired_capabilities import DesiredCapabilities

driver = webdriver.Remote(
   command_executor='http://127.0.0.1:4444/wd/hub',
   desired_capabilities=DesiredCapabilities.CHROME)
```

Hm, except I think this expects the remote to start a session, one that talks the WebDriver protocol rather than DevTools? Not sure. Actually,it looks like you can pass `debuggerAddress` to ChromeDriver, e.g. from [here](https://stackoverflow.com/questions/12698843/how-do-i-pass-options-to-the-selenium-chrome-driver-using-python#12698844):

```python
from selenium import webdriver
from selenium.webdriver.chrome.options import Options
chrome_options = Options()
chrome_options.add_argument("--disable-extensions")
driver = webdriver.Chrome(chrome_options=chrome_options)
```

So, starting it with pyppeteer and passing the `debuggerAddress` to ChromeDriver for the Selenium stage should work okay.



Ingest NG Phase 2
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
