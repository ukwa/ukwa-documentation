# Management Services

## Preserve

We keep the WARCs and log files from all the crawlers, in a folder structure that reflects the crawl process. We keep content obtained under different terms in separate folders. For example, openly available Non-Print Legal Deposit material is separate from material that we obtained by permission, or from behind a paywall.


# Maintaining service through system migrations

A common use case is wanting to be able to maintain access to a service while upgrading or replacing it.  This document covers some suggestions of how to do this, based on our experience running UKWA.

## Sync & Switch

The idea here is to run a new service in parallel to the old one, but to rapidly synchronise the data so while the older service 'leads' the new one is following close behind. Because the two are synchronised, it is possible to _briefly_ pause any clients and update them to refer to the new service.

A good example of this is internal CDX service, which runs on OutbackCDX, which is used to enable playback of web pages and so is a critical part of our service. But upgrades can be slow because of the large amount of data in it. Fortunately, the on-disk format of the system's database is a fairly simple set of files that are created and appended by the service, but not otherwise modified or edited. Better still, the design of OutbackCDX means that these files could be updated while the system was running, and as long as there's only one process updating the database, it works fine. This means we were able to use `rsync` to keep updating the data from the older service and make it available in the newer service, and run _read only_ checks on the newer service to check it was working.

The final switch-over process looked like this:

- Block/shutdown all clients that write to the (old) CDX service.
- Run one final `rsync` and check it worked.
- Switch read-only clients over to the new service. Check they are working okay.
- Switch write clienrs over to the new service. Check they are working okay.
- Decommission the old service.

Because most of the data had already been transferred over in previous synchronisations, the final data synchronisation is nice and quick. This means the eventual service downtime is only a few minutes, and if anything goes wrong, it's always possible to revert to the old system.

## Access Facade

For services where synchronisation is slow or risky, we can manage migration using a more sophisticated approach:

1. Separate the  _write_ and _read_ functionality so that they can be operated independently.
2. Implement the _read_ functionality so the _read_ service can pull data from more than one _write_ service.

The reason for treating _write_ and _read_ functionality differently is partially because writes are usually easier to _pause_ than reads. But mostly because _ensuring consistenty of data in multi-component systems is really really hard!_. There's even a [theorem](https://en.wikipedia.org/wiki/CAP_theorem) that proves precisely how hard it is!

## Fast Switching

Service aliases and proxies.