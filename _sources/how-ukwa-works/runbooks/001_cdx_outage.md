CDX Outage
==========

Triggers
--------

- ALERT: the CDX API endpoint (http://cdx.api.wa.bl.uk) is down.
- ALERT: the live CDX server (e.g. one of cdx1/cdx2) is down.


Dependents
----------
~~~~
- Wayback access will also report as down in this situation (although that also happens for other reasons, see TBA). However, access services can be left running in this situation without causing harm.
- The Frequent Crawl uses a CDX to store crawl state, so will also by affected and should be paused during remediation.
- The CDX update process will fail while the CDX is down.  This should be paused during remediation, as this is the process that writes to the CDX and we need to make sure we are updating the index correctly.


Actions
-------


### Switch to failover read-only CDX server

We run the CDX service as a pair of servers, with one 'leader' that is usually in use for both reads and writes, and one 'follower' that has an up-to-date replica of the data needed for accessing the web archive.

Therefore, when 

/root/gitlab/outbackcdx

./start_outbackcdx.sh

[root@cdx1 outbackcdx]# docker service ps outbackcdx_outbackcdx
ID             NAME                      IMAGE                       NODE      DESIRED STATE   CURRENT STATE                ERROR     PORTS
rdt9ivi1sqlx   outbackcdx_outbackcdx.1   nlagovau/outbackcdx:0.7.0   cdx1      Running         Running about a minute ago


http://cdx1.n45.wa.bl.uk:8080/#/data-heritrix?url=https%3A%2F%2Fwww.bl.uk

Now switch over cdx.api.wa.bl.uk to point at cdx1 instead of cdx2.


./start_crawldb-fc.sh

```
[root@cdx1 outbackcdx]# docker service ps crawldb-fc_crawldb-fc
ID             NAME                      IMAGE                       NODE      DESIRED STATE   CURRENT STATE            ERROR     PORTS
xz1rxq1vzzp7   crawldb-fc_crawldb-fc.1   nlagovau/outbackcdx:0.7.0   cdx1      Running         Running 15 seconds ago
[root@cdx1 outbackcdx]# docker service ls
ID             NAME                    MODE         REPLICAS   IMAGE                       PORTS
enyvz8j2xohl   crawldb-fc_crawldb-fc   replicated   1/1        nlagovau/outbackcdx:0.7.0   *:8081->8081/tcp
u8eczz7fw3e2   outbackcdx_outbackcdx   replicated   1/1        nlagovau/outbackcdx:0.7.0   *:8080->8080/tcp
```



Crawler

 docker run -ti ukwa/hapy h3cc -H crawler06.bl.uk -P 8443 -u admin -p \*\*\*\*\*\*\*\*\*\* show-metadata

`kill-all-toethreads`

TODO Add command to run CP?

ocdxc = appCtx.getBean("outbackCDXClient")
rawOut.println(ocdxc.endpoint)
ocdxc.endpoint = "http://192.168.45.7:8081/fc"
rawOut.println(ocdxc.endpoint)

(Not a proper solution as warcprox needs the update too)



b = appCtx.getBean("checkpointService")
rawOut.println(b.lastCheckpointSnapshot)
b.lastCheckpointSnapshot = null
rawOut.println(b.lastCheckpointSnapshot)

touch /mnt/gluster/fc/heritrix/output/frequent-npld/20210519154706/logs/crawl.log

`Checkpoint cp00033-20210528102926 saved`



Checkpoint cp00049-20210610194553 saved
