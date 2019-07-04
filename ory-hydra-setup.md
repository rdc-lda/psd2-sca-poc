# Ory Hydra setup

This guide will:

1. Download and run a PostgreSQL container in Docker.
1. Download and run ORY Hydra in Docker.
1. Download and run our reference User Login & Consent Provider.
1. Create an OAuth 2.0 Client to perform the OAuth 2.0 1. Authorize Code Flow.
1. Perform the OAuth 2.0 Authorize Code flow.

Before starting with this guide, please install the most recent version of Docker Desktop for Mac. It is advised to start with a "fresh" configuration -- hence apply the factory defaults is a good idea.

## Create a Network

Before we can start, a network must be created which we will attach all our Docker containers to. That way, the containers can talk to one another.

~~~bash
docker network create hydrapoc
~~~

## Deploy PostgreSQL

For the purpose of this poc, we will use PostgreSQL as a database.

~~~bash
docker run \
  --network hydrapoc \
  --name ory-hydra-poc--postgres \
  -e POSTGRES_USER=hydra \
  -e POSTGRES_PASSWORD=secret \
  -e POSTGRES_DB=hydra-poc \
  -d postgres:9.6
~~~

This command wil start a postgres instance with name `ory-hydra-poc--postgres`, set up a database called `hydra-poc` and create a user `hydra` with password `secret`.

## Deploy ORY Hydra

The vendor highly recommends using Docker to run Hydra, as installing, configuring and running Hydra is easiest with Docker. ORY Hydra is available on [Docker Hub](https://hub.docker.com/r/oryd/hydra/).

