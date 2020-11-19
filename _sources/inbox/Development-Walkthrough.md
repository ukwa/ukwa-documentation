Development Walkthrough
=======================

As an example, imagine we are working on a modification to the main [crawl engine](https://github.com/ukwa/ukwa-heritrix). If we want to check the crawler is working as expected, can use the supplied [*docker-compose.yml*](https://github.com/ukwa/ukwa-heritrix/blob/master/docker-compose.yml) to run a set of Docker containers that spin up the various services the crawl depends on.

    docker-compose up

Seeds can be injected into the crawl, and we can run end-to-end tests as we wish. When we are happy, these changes can be committed and pushed to GitHub.

Production containers are built on [*Docker Hub*](https://hub.docker.com/u/ukwa/), using automated builds based on branches and tagged versions of the relevant GitHub codebases, e.g. [*here’s the build history for W3ACT*](https://hub.docker.com/r/ukwa/w3act/builds/).

In general, each Dockerized service should have a single GitHub repository with a Dockerfile that defines how to build the Docker container. Using Docker Hub Automated Builds, each of these should be set up to build `latest` containers from the master branch, and build tagged containers when tags are pushed to GitHub. All configuration should be carried out via environment variables, and the same binaries should be deployed across all test/beta/production contexts.

The deployment of releases will usually involve the following steps:

1. Tag the version and push to GitHub.
2. Wait for the container to be built.
3. Update the Docker Compose file to use the new version.
4. Bring the Docker service up using the new version tag.
5. If something goes wrong, roll-back to the previous version.

The [*ukwa-ingest-services*](https://github.com/ukwa/ukwa-ingest-services) codebase pulls the different crawl components together using [*Docker*](https://www.docker.com/), and uses [*Docker Compose*](https://docs.docker.com/compose/) to define different deployments, from local development to full production.

Hence, when we're happy to make a formal release, we tag the component repository, push the tag, and once the new image is built, update the deployment orchestration to specify the new version tag, and roll it out, e.g.

    docker stack deploy -c ~/github/ukwa-ingest-services/deploy/crawl-engine/prod/docker-compose.yml crawler


