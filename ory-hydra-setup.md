# Ory Hydra setup

This guide will:

1. Install the local hydra tooling
1. Download and run a PostgreSQL container in Docker.
1. Download and run ORY Hydra in Docker.
1. Download and run our reference User Login & Consent Provider.
1. Create an OAuth 2.0 Client to perform the OAuth 2.0 1. Authorize Code Flow.
1. Perform the OAuth 2.0 Authorize Code flow.

Before starting with this guide, please install the most recent version of Docker Desktop for Mac. It is advised to start with a "fresh" configuration -- hence apply the factory defaults is a good idea.

## Install Hydra CLI for OSX

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

### Management of Hydra server

Here we discuss basic access to logfiles and management APIs.

On start up, ORY Hydra is initializing some values. Let's take a look at the logs:

~~~bash
# Get the logs...
$ docker logs ory-hydra-poc--hydra

[...]
Config file not found because "Config File ".hydra" Not Found in "[/]""
time="2019-07-05T13:03:30Z" level=info msg="No tracer configured - skipping tracing setup"
time="2019-07-05T13:03:30Z" level=info msg="Connecting with postgres://*:*@ory-hydra-poc--postgres:5432/hydra-poc?sslmode=disable"
time="2019-07-05T13:03:30Z" level=info msg="Connected to SQL!"

time="2019-07-05T13:03:30Z" level=warning msg="JSON Web Key Set \"hydra.openid.id-token\" does not exist yet, generating new key pair..."
time="2019-07-05T13:03:33Z" level=warning msg="JSON Web Key Set \"hydra.jwt.access-token\" does not exist yet, generating new key pair..."
time="2019-07-05T13:03:34Z" level=info msg="Software quality assurance features are enabled. Learn more at: https://www.ory.sh/docs/ecosystem/sqa"
time="2019-07-05T13:03:34Z" level=warning msg="JSON Web Key Set \"hydra.https-tls\" does not exist yet, generating new key pair..."

time="2019-07-05T13:03:38Z" level=info msg="Setting up http server on :4444"
time="2019-07-05T13:03:38Z" level=info msg="Setting up http server on :4445"
~~~

As you can see, the following steps are performed when running ORY Hydra against a fresh database:

1. If no system secret was given (in our case we provided one), a random one is generated and emitted to the logs. Note this down, otherwise you won't be able to restart Hydra.
1. Cryptographic keys are generated for the:
    1. OpenID Connect ID Token, 
    1. the consent challenge and response, 
    1. TLS encryption using a self-signed certificate,

...which is why we need to run all commands using `--skip-tls-verify`

ORY Hydra can be managed using the Hydra Command Line Interface (CLI), which is using ORY Hydra's REST APIs.

~~~bash
# Run via Docker
$ docker run --rm -it \
  -e HYDRA_ADMIN_URL=https://ory-hydra-poc--hydra:4445 \
  --network hydrapoc \
  oryd/hydra:v1.0.0 \
  [command] --skip-tls-verify [options]

Hydra is a cloud native high throughput OAuth2 and OpenID Connect provider

# Run from your MacBook
$ export HYDRA_ADMIN_URL=https://localhost:9001
$ hydra [command] \
   --skip-tls-verify \
   [options]

# ... for example:
$ hydra clients list --skip-tls-verify

| CLIENT ID | NAME | RESPONSE TYPES | SCOPE | REDIRECT URIS | GRANT TYPES | TOKEN ENDPOINT AUTH METHOD |
|-----------|------|----------------|-------|---------------|-------------|----------------------------|
~~~

## Deploy Login & Consent App

The Login Provider and Consent Provider can be two separate web services. We provide a [reference implementation](https://github.com/rdc-lda/hydra-login-consent-node) which combines both features in one app. Here, we will use deploy that app using Docker.

~~~bash
# Start the Login and Consent module
$ docker run -d \
  --name ory-hydra-poc--consent \
  -p 9020:3000 \
  --network hydrapoc \
  -e HYDRA_ADMIN_URL=https://ory-hydra-poc--hydra:4445 \
  -e NODE_TLS_REJECT_UNAUTHORIZED=0 \
  rdclda/hydra-login-consent-node:latest

# Let's check if it's running ok:
$ docker logs ory-hydra-poc--consent

> hydra-login-consent-logout@0.0.0 start /usr/src/app
> node ./bin/www
~~~

Let's take a look at the arguments:

* `-p 9020:3000` exposes this service at `port 9020`. If you remember, that's the port of the `URLS_CONSENT` and `URLS_LOGIN` value from the ORY Hydra docker container (`URLS_CONSENT=http://localhost:9020/consent`, `URLS_LOGIN=http://localhost:9020/login`).
* `HYDRA_ADMIN_URL=https://ory-hydra-poc--hydra:4445` point to the ORY Hydra Administrative API.
* `NODE_TLS_REJECT_UNAUTHORIZED=0` disables TLS verification, because we are using self-signed certificates.

## Define OAuth 2.0 Client

Great! Our infrastructure is all set up!

Next it's time to perform the OAuth 2.0 Authorize Code flow.

For that purpose, the ORY Hydra CLI has a feature that sets up an OAuth 2.0 Consumer and an OAuth 2.0 callback URL. Typically, this would be a third-party application that requests access to a user's resources on your servers - for example a FinTech App you wrote that analyses a  user's loan accounts and thus requires read access to the user's loan accounts.

Before we go ahead, the OAuth 2.0 Client that performs the request has to be set up. Let's call the client `loansharks-r-us`. We have to specify which OAuth 2.0 Grant Types, OAuth 2.0 Scope, OAuth 2.0 Response Types, and Callback URLs the client may request:

~~~bash
# Define the oAuth2.0 client 
#  -- make sure HYDRA_ADMIN_URL is set!
$ hydra clients create --skip-tls-verify \
    --id loansharks-r-us \
    --secret 'W3(L)urM0n3y!' \
    --grant-types authorization_code,refresh_token,client_credentials,implicit \
    --response-types token,code,id_token \
    --scope openid,offline,ReadAccountsDetail,ReadBalances,ReadTransactionsDetail \
    --callbacks http://127.0.0.1:9010/callback

OAuth 2.0 Client ID: loansharks-r-us
~~~

**Note** _You should not provide secrets using command line flags, the secret might leak to bash history and similar systems_

Let's dive into some of the arguments:

* `--skip-tls-verify` is supported by all management commands (create/delete/update/... OAuth 2.0 Client, JSON Web Key, ...) and tells the CLI to trust any certificate authority - even self-signed ones. We need this flag because the server uses a self-signed certificate. In production deployments, you would use a certificate signed by a trusted CA.
* `--grant-types authorize_code,refresh_token,client_credentials,implicit` we want to be able to perform all of these OAuth 2.0 flows.
* `--response-types` token,code,id_token allows us to receive authorize codes, access and refresh tokens, and OpenID Connect ID Tokens.
* `--scope openid,offline,ReadAccountsDetail,ReadBalances,ReadTransactionsDetail` allows the client to request various permissions:
  * `openid` allows the client to perform the OpenID Connect flow and request an OpenID Connect ID Token.
  * `offline` allows the client to request a refresh token. Because we want to continuously update the analysis, the app must be able to refresh expired access tokens. This scope allows that.
  * `ReadAccountsDetail` this is an imaginary scope that is not handled by ORY Hydra but serves the purpose of making it clear that we could request read access to account details.
  * `ReadBalances` - same as above
  * `ReadTransactionsDetail` - same as above
  * `--callbacks http://localhost:9010/callback` allows the client to request this redirect uri.

## Perform an example oAuth2 flow

Perfect, let's perform an exemplary OAuth 2.0 Authorize Code Flow! To make this easy, the ORY Hydra CLI provides a helper command called `hydra token user`. Just imagine this being, for example, `passport.js` that is generating an auth code url, redirecting the browser to it, and then exchanging the authorize code for an access token.

![OAuth 2.0 Authorize Code flow](./imgs/auth_code.png)

The same thing happens with this command:

~~~bash
# Start flow
$ hydra token user --skip-tls-verify \
    --port 9010 \
    --auth-url https://localhost:9000/oauth2/auth \
    --token-url https://localhost:9000/oauth2/token \
    --client-id loansharks-r-us \
    --client-secret 'W3(L)urM0n3y!' \
    --scope openid,offline,ReadAccountsDetail

Setting up home route on http://127.0.0.1:9010/
Setting up callback listener on http://127.0.0.1:9010/callback
Press ctrl + c on Linux / Windows or cmd + c on OSX to end the process.
If your browser does not open automatically, navigate to:

        http://127.0.0.1:9010/
~~~

Your browser opens, the welcome text is displayed and you can click on the login link -- provide credentials and the consent, then the token user app will return the tokens below:

~~~text
Access Token:
        Q5CSsn_BKknjGoga9Hi0sV88_znqeZ5ABk84jZE7o5I.Yr9dzhaFf4HjQ2YpTqMXg23aB7en_C75vYeMg4DyI2s
Refresh Token:
        BkBhdKs9LzyZKVpkFb7P1VeQUH0THgZCjHwYfmuVZ28.Z1ZugUrQjzFyHt-UiLatoWHc5avywizQhkuHZBY81oU

Expires in:
        2019-07-05 17:17:06.166635 +0100 WEST m=+3628.954574766

ID Token:
        eyJhbGciOiJSUzI1NiIsImtpZCI6InB1YmxpYzplMzJiN2ZlYi05NDM4LTQ4OGEtODkwNy0wYzRhNzRmNWI3MjkiLCJ0eXAiOiJKV1QifQ.eyJhdF9oYXNoIjoiRTIwcjVsQzRHTDZsMmhwQnU2NzFpQSIsImF1ZCI6WyJsb2Fuc2hhcmtzLXItdXMiXSwiYXV0aF90aW1lIjoxNTYyMzM5ODE1LCJleHAiOjE1NjIzNDM0MjcsImlhdCI6MTU2MjMzOTgyNywiaXNzIjoiaHR0cHM6Ly9sb2NhbGhvc3Q6OTAwMC8iLCJqdGkiOiI3MThmMmE1YS01MTI3LTQ1YjMtYjMwMS1kZDM4Y2RmZmIwYWIiLCJub25jZSI6InlnZ3Rxc3hqaWxmZGFiaW96Y3poamVyeCIsInJhdCI6MTU2MjMzOTgwNiwic2lkIjoiOTY0MDI0NDMtOTYzYS00MTk2LWEyNDQtYjk1YmNkNGRjNWJhIiwic3ViIjoiZm9vQGJhci5jb20ifQ.p2VOguZep8Dp3lvSZ3a_AL3uGo_NA5zvbx5GFXCKGwWlE1kxo4Wl5dS4Ds-2YIlFoqma42_00jg2Q87UrY8Z8hME1nZ1lPtkL8jqH8CrFSfF-n0xHw7BWTQ5qFiZvIY-4nYm_RIwvVKDzc1CO_uwwz2f2pi21FzKwUWF4S2LOyUEhQ_p0AmBDXgBfV-Mrp8Ut4G_fG17fZAEBFP1We8SdokBirXWggN1IpTptKqJKeu6PfeBgvdKhDQhA40CJFkC539dsNbj3LuAn5p0QuUngV3O9-fDSo_A3Kr-G-lexl09A6iLkB2iErCaw8wtUCTBqFZ7aZQmSRKSKxZTVrGAkgE_QJjrH2SX3O5TyXCeuxpqjpPt38EgSzGgUMxm838UrjerLdtC1Uch5tbj-K6TvdjqWzPaGY9MTcvV6WRrqk-tfpUDjju5SJVL_DjuSB9xswZXO1UN6RoqG-WKJ9Uhu1mjpSgnYjMc_wS_Fv_o9RdgM7zUJjAIOq5ex4YfGqvXKC6X8Jd81en3FgHZcJEmPneGmkyMssMC6nVJteCgnvJsqyrdhFjNuqNfYheVjWMQfv8o0DeaPVXYmPddZkubSkbHzBmKIrqvYaTfVrl2ZcXeZDQVpqXKOK6AzdbOJzHDvP756hTBakEUHLXNOcIpDdMnKr9MC38ODW6EC3BGfzM
~~~

## Validating OAuth 2.0 Access Tokens

When an OAuth 2.0 client makes a request to the resource server, the resource server needs some way to verify the access token. The OAuth 2.0 core spec doesn’t define a specific method of how the resource server should verify access tokens, just mentions that it requires coordination between the resource and authorization servers. In some cases, especially with small services, both endpoints are part of the same system, and can share token information internally such as in a database. In larger systems where the two endpoints are on different servers, this has led to proprietary and non-standard protocols for communicating between the two servers.

The OAuth 2.0 Token Introspection extension defines a protocol that returns information about an access token, intended to be used by resource servers or other internal servers.

![OAuth 2.0 flow](./imgs/basic-oauth2-system.png)

The best and easiest way to validate OAuth 2.0 Access Tokens is by performing OAuth 2.0 Token Introspection. You can do this with the CLI `hydra token introspect <token>`.

~~~bash
# Introspect the token via Hydra
$ hydra token introspect \
   --skip-tls-verify \
   'Q5CSsn_BKknjGoga9Hi0sV88_znqeZ5ABk84jZE7o5I.Yr9dzhaFf4HjQ2YpTqMXg23aB7en_C75vYeMg4DyI2s'
~~~

...and as a result...

~~~json
{
        "active": true,
        "aud": null,
        "client_id": "loansharks-r-us",
        "exp": 1562343427,
        "iat": 1562339827,
        "iss": "https://localhost:9000/",
        "scope": "openid offline ReadAccountsDetail",
        "sub": "foo@bar.com",
        "token_type": "access_token"
}
~~~

What happens under water... (showcase CURL) is:

~~~bash
$ curl -X POST --insecure \
    -d 'token=Q5CSsn_BKknjGoga9Hi0sV88_znqeZ5ABk84jZE7o5I.Yr9dzhaFf4HjQ2YpTqMXg23aB7en_C75vYeMg4DyI2s' \
    https://localhost:9001/oauth2/introspect

{"active":true,"scope":"openid offline ReadAccountsDetail","client_id":"loansharks-r-us","sub":"foo@bar.com","exp":1562343427,"iat":1562339827,"iss":"https://localhost:9000/","token_type":"access_token"}
~~~
