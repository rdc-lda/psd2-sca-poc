# Ory Hydra setup

This guide will:

1. Install the local hydra tooling
1. Download and run a PostgreSQL container in Docker.
1. Download and run ORY Hydra in Docker.
1. Download and run our reference User Login & Consent Provider.
1. Create an OAuth 2.0 Client to perform the OAuth 2.0 1. Authorize Code Flow.
1. Perform the OAuth 2.0 Authorize Code flow.

Before starting with this guide, please install the most recent version of Docker Desktop for Mac. It is advised to start with a "fresh" configuration -- hence apply the factory defaults is a good idea.

## Install Hydra utilities for OSX

In order to use the native hydra CLI on your workstation, install via brew:

~~~bash
# Add the brew tap
$ brew tap ory/hydra

# Install
$ brew install ory/hydra/hydra

# Test
$ hydra version
Version:    v1.0.0-rc.14+oryOS.12
Git Hash:   84d9cdf79c7962f40440b946fd233ec2b858bf81
Build Time: 2019-05-19T10:32:54Z
~~~

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

### Generate system secret

* The system secret can only be set against a fresh database. Key rotation is currently not supported. 
* This secret is used to encrypt the database and needs to be set to the same value every time the process (re-)starts.
* You can use `/dev/urandom` to generate a secret. But make sure that the secret must be the same anytime you define it.

~~~bash
# You could, for example, store the value somewhere.
$ export SECRETS_SYSTEM=$(export LC_CTYPE=C; \
   cat /dev/urandom | \
   tr -dc 'a-zA-Z0-9' | \
   fold -w 32 | \
   head -n 1)

# Alternatively you can obviously just set a secret:
$ export SECRETS_SYSTEM='S@m3th1n9L0n9AndH@rd2Gu355!'
~~~

### Configure database

Here we will configure the connection (URL) and the database schema.

The database url BELOW points at the Postgres instance. This could also be an ephemeral in-memory database (`export DSN=memory`) for the embedded (MySQL based) URI.

~~~bash
# Set DSN
$ export DSN=postgres://hydra:secret@ory-hydra-poc--postgres:5432/hydra-poc?sslmode=disable
~~~

Next steps we are pushing the database DDL to postgres; ORY Hydra does not do magic, it requires conscious decisions, for example running SQL migrations which is required when installing a new version of ORY Hydra, or upgrading an existing installation.

While the command below is run within the Docker (network), this is the equivalent to running from the CLI: `hydra migrate sql --yes postgres://hydra:secret@ory-hydra-poc--postgres:5432/hydra-poc?sslmode=disable`

~~~bash
# Start pushing the schema...
$ docker run -it --rm \
  --network hydrapoc \
  oryd/hydra:v1.0.0 \
  migrate sql --yes $DSN

Applying `client` SQL migrations...
[...]
Successfully applied 38 SQL migrations!
~~~

### Startup server

In the next step, we start the server -- you might want to check out the supported options using `hydra help serve`, read carefully!

~~~bash
# Start the Hydra server
$ docker run -d \
  --name ory-hydra-poc--hydra \
  --network hydrapoc \
  -p 9000:4444 \
  -p 9001:4445 \
  -e SECRETS_SYSTEM=$SECRETS_SYSTEM \
  -e DSN=$DSN \
  -e URLS_SELF_ISSUER=https://localhost:9000/ \
  -e URLS_CONSENT=http://localhost:9020/consent \
  -e URLS_LOGIN=http://localhost:9020/login \
  oryd/hydra:v1.0.0 serve all
~~~

You can follow the progress on the startup process using:

~~~bash
# Tail the Docker logs dor this container
$ docker logs -f ory-hydra-poc--hydra

[...]
time="2019-07-05T13:03:34Z" level=warning msg="JSON Web Key Set \"hydra.https-tls\" does not exist yet, generating new key pair..."
time="2019-07-05T13:03:38Z" level=info msg="Setting up http server on :4444"
time="2019-07-05T13:03:38Z" level=info msg="Setting up http server on :4445"
~~~

Let's dive into the various settings:

* `--network hydrapoc` connects this instance to the network and makes it possible to connect to the PostgreSQL database.
* `-p 9000:4444` exposes ORY Hydra's public API on https://localhost:9000/.
* `-p 9001:4445` exposes ORY Hydra's administrative API on https://localhost:9001/.
* `-e SECRETS_SYSTEM=$SECRETS_SYSTEM` sets the system secret environment variable (required).
* `-e DSN=$DSN` sets the database url environment variable (required).
* `-e URLS_SELF_ISSUER=https://localhost:9000/` this value must be set to the publicly available URL of ORY Hydra (required).
* `-e URLS_CONSENT=http://localhost:9020/consent` this sets the URL of the consent provider (required). We will set up the service that handles requests at that URL in the next sections.
* `-e URLS_LOGIN=http://localhost:9020/login` this sets the URL of the login provider (required). We will set up the service that handles requests at that URL in the next sections.

**Note**: _In this example we did not define a value for the optional setting `OAUTH2_ERROR_URL`. This URL can be used to provide an endpoint which will receive error messages from ORY Hydra that should be displayed to the end user. The URL receives `error` and `error_description` parameters. If this value is not set, Hydra uses the fallback endpoint `/oauth2/fallbacks/error` and displays a default error message. In order to obtain a uniform UI, you might want to include such an endpoint in your login or consent provider._

To confirm that the instance is running properly, [open the health check](https://localhost:9001/health/ready). If asked, accept the self signed certificate in your browser. You should see `{"status":"ok"}`.

