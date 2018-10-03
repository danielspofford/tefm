# Clustered Elixir on AWS ECS with CircleCI

This guide will walk you through setting up an Elixir application on existing
AWS infrastructure like that setup in the
[Standing up an ECS Web Stack with Stacker](content/guides/standing-up-an-ecs-web-stack-with-stacker.md)
guide.

## Prerequisites

An AWS account and credentials.

- Erlang 21.1
- Elixir 1.7.3
- Phoenix 1.3.0

## Create a Phoenix Project

```
$ mix phx.new lime --umbrella
Fetch and install dependencies? [Yn]
$ y
$ cd lime_umbrella
```

## Docker for Local Development Environment

Create a `docker-compose.yml` file in the project root with the follow contents:

```
version: '2'
services:
  db:
    image: postgres:10.5
    ports:
     - "5432:5432"
```

Start postgres and create your database:

```
$ docker-compose up -d
$ mix ecto.create
```

## OTP Releases via Distillery

Reference: Distillery's [Phoenix walkthrough](phx_walkthrough)

Add `{:distillery, "~> 2.0"}` to the root `mix.exs` and then:

```
$ mix do deps.get, compile
$ mix release.init
```

Update the prod endpoint configuration to resemble this:

```
config :lime_umbrella, LimeWeb.Endpoint,
  http: [port: {:system, "PORT"}],
  url: [host: "localhost", port: {:system, "PORT"}],
  cache_static_manifest: "priv/static/cache_manifest.json",
  server: true,
  root: ".",
  version: Application.spec(:lime_umbrella, :vsn)
```

```
$ cd apps/lime_web/assets/
$ node node_modules/brunch/bin/brunch build --production
$ cd ..
$ mix phx.digest
```

Our Ecto configuration will rely entirely on a `DATABASE_URL` env var. Lets
straighten out our repo configurations a bit to reflect that.  Navigate to
`apps/lime/config` and modify `config.exs` adding:

```
config :lime, Lime.Repo,
  adapter: Ecto.Adapters.Postgres,
  pool_size: 10
```

Drop all repo configuration from `dev.exs` and do the same to `test.exs` except
leave the `:pool` configuration for the repo. Next, delete `import_config
"prod.secret.exs"` from `prod.exs` and delete the `prod.secret.exs` file as
well. Keep in mind now that for all mix envs starting your application requires
the `DATABASE_URL` env var to be set. A tool like direnv can minimize the pain
of managing your environment.

Finally, build your release and ensure it starts:

```
$ MIX_ENV=prod mix release
$ PORT=4001 \
    DATABASE_URL="postgres://postgres:postgres@localhost/lime_dev" \
    _build/prod/rel/lime_umbrella/bin/lime_umbrella foreground
```

## Containerize your Releases

Reference: Distillery's [Working with Docker](working_with_docker)

Create a new `Dockerfile` in your project root:

```
# The version of Alpine to use for the final image
# This should match the version of Alpine that the `elixir:1.7.2-alpine` image uses
ARG ALPINE_VERSION=3.8

FROM elixir:1.7.2-alpine AS builder

# The following are build arguments used to change variable parts of the image.
# The name of your application/release (required)
ARG APP_NAME
# The version of the application we are building (required)
ARG APP_VSN
# The environment to build with
ARG MIX_ENV=prod
# Set this to true if this release is not a Phoenix app
ARG SKIP_PHOENIX=false
# If you are using an umbrella project, you can change this
# argument to the directory the Phoenix app is in so that the assets
# can be built
ARG PHOENIX_SUBDIR=./apps/lime_web

ENV SKIP_PHOENIX=${SKIP_PHOENIX} \
    APP_NAME=${APP_NAME} \
    APP_VSN=${APP_VSN} \
    MIX_ENV=${MIX_ENV}

# By convention, /opt is typically used for applications
WORKDIR /opt/app

# This step installs all the build tools we'll need
RUN apk update && \
  apk upgrade --no-cache && \
  apk add --no-cache \
    nodejs \
    yarn \
    git \
    build-base && \
  mix local.rebar --force && \
  mix local.hex --force

# This copies our app source code into the build container
COPY . .

RUN mix do deps.get, deps.compile, compile

# This step builds assets for the Phoenix app (if there is one)
# If you aren't building a Phoenix app, pass `--build-arg SKIP_PHOENIX=true`
# This is mostly here for demonstration purposes
RUN if [ ! "$SKIP_PHOENIX" = "true" ]; then \
  cd ${PHOENIX_SUBDIR}/assets && \
  yarn install && \
  yarn deploy && \
  cd .. && \
  mix phx.digest; \
fi

RUN \
  mkdir -p /opt/built && \
  mix release --verbose && \
  cp _build/${MIX_ENV}/rel/${APP_NAME}/releases/${APP_VSN}/${APP_NAME}.tar.gz /opt/built && \
  cd /opt/built && \
  tar -xzf ${APP_NAME}.tar.gz && \
  rm ${APP_NAME}.tar.gz

# From this line onwards, we're in a new image, which will be the image used in production
FROM alpine:${ALPINE_VERSION}

# The name of your application/release (required)
ARG APP_NAME

RUN apk update && \
    apk add --no-cache \
      bash \
      openssl-dev

ENV REPLACE_OS_VARS=true \
    APP_NAME=${APP_NAME}

WORKDIR /opt/app

COPY --from=builder /opt/built .

CMD trap 'exit' INT; /opt/app/bin/${APP_NAME} foreground
```

Create a `.dockerignore` in your project root:

```
_build/
deps/
.git/
.gitignore
Dockerfile
Makefile
README*
test/
priv/static/
```

Create a `Makefile` in your project root:

```
.PHONY: help

APP_NAME ?= lime_umbrella
APP_VSN ?= 0.1.0
BUILD ?= `git rev-parse --short HEAD`

help:
    @echo "$(APP_NAME):$(APP_VSN)-$(BUILD)"
    @perl -nle'print $& if m{^[a-zA-Z_-]+:.*?## .*$$}' $(MAKEFILE_LIST) | sort | awk 'BEGIN {FS = ":.*?## "}; {printf "\033[36m%-30s\033[0m %s\n", $$1, $$2}'

build: ## Build the Docker image
  docker build --build-arg APP_NAME=$(APP_NAME) \
    --build-arg APP_VSN=$(APP_VSN) \
    -t $(APP_NAME):$(APP_VSN)-$(BUILD) \
    -t $(APP_NAME):latest .

run: ## Run the app in Docker
  docker run --env-file config/docker.env \
    --expose 4000 -p 4000:4000 \
    --rm -it $(APP_NAME):latest
```

Ensure help outputs something similar then build the image:

```
$ make help
lime_umbrella:0.1.0-83e046d
build                          Build the Docker image
run                            Run the app in Docker
$ make build
```

[phx_walkthrough]: https://hexdocs.pm/distillery/guides/phoenix_walkthrough.html
"Distillery's Phoenix walkthrough"
[working_with_docker]: https://hexdocs.pm/distillery/guides/working_with_docker.html
"Working With Docker"
