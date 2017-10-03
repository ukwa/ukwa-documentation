The [*pulse*](https://github.com/ukwa/pulse) codebase pulls the different crawl components together using [*Docker*](https://www.docker.com/), and uses [*Docker Compose*](https://docs.docker.com/compose/) to define different deployments, from local development to full production.

The intention is that the supplied [*docker-compose.yml*](https://github.com/ukwa/pulse/blob/master/docker-compose.yml) file can be use to run an end-to-end test of a crawl, all within a set of Docker containers running locally or as part of a Travis CI automated build.

Production containers are built on [*Docker Hub*](https://hub.docker.com/u/ukwa/), using automated builds based on tagged versions of the relevant GitHub codebases, e.g. [*here’s the build history for W3ACT*](https://hub.docker.com/r/ukwa/w3act/builds/).

In general, each Dockerized service should have a single GitHub repository with a Dockerfile that defines how to build the Docker container. Using Docker Hub Automated Builds, each of these should be set up to build `latest` containers from the master branch, and build tagged containers when tags are pushed to GitHub. All configuration should be carried out via environment variables, and the same binaries should be deployed across all test/beta/production contexts.

The deployment of releases will usually involve the following steps:

1. Tag the version and push to GitHub.
2. Wait for the container to be built.
3. Update the Docker Compose file to use the new version.
4. Bring the Docker service up using the new version tag.
5. If something goes wrong, roll-back to the previous version.
