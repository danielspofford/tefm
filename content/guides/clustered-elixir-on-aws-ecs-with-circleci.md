# Clustered Elixir on AWS ECS with CircleCI

This guide will walk you through setting up an Elixir application on top of AWS
infrastructure resulting in:

- An Elixir cluster
- Convenient build via Docker and a Makefile
- Containerized OTP releases
- CI/CD via CircleCI
- Easy boot time configuration via Distillery's config providers

## Prerequisites

An AWS account and credentials.

- Erlang 21.1
- Elixir 1.7.3
- Phoenix 1.3.4

## Create a Phoenix Project

```
$ mix phx.new lime --umbrella
Fetch and install dependencies? [Yn]
$ y
$ cd lime_umbrella && mix format
```

## Docker for Local Postgres

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

Replace the `LimeWeb.Endpoint` config in `apps/lime_web/config/prod.exs` with:

```
config :lime_web, LimeWeb.Endpoint,
  http: [port: {:system, "PORT"}],
  url: [host: "localhost", port: {:system, "PORT"}],
  cache_static_manifest: "priv/static/cache_manifest.json",
  server: true,
  root: ".",
  version: Application.spec(:lime_umbrella, :vsn)
```

Navigate to `apps/lime_web/assets/` then:

```
$ node node_modules/brunch/bin/brunch build --production
$ cd ..
$ mix phx.digest
```

Modify `apps/lime/config/config.exs` adding:

```
config :lime, Lime.Repo,
  adapter: Ecto.Adapters.Postgres,
  pool_size: 10
```

Remove all repo configuration from `apps/lime/config/dev.exs`. Reduce the repo
configuration in `apps/lime/config/test.exs` to just:

```
config :lime, Lime.Repo,
  pool: Ecto.Adapters.SQL.Sandbox
```

Modify `apps/lime/config/prod.exs` removing this line:

```
import_config "prod.secret.exs"
```

Delete the `apps/lime/config/prod.secret.exs` file.

*Note*: In all Mix environments Ecto is configured via the `DATABASE_URL`
environment variable.

Build your release and ensure it starts:

```
$ MIX_ENV=prod mix release
$ PORT=4001 \
    DATABASE_URL="postgres://postgres:postgres@localhost/lime_dev" \
    _build/prod/rel/lime_umbrella/bin/lime_umbrella foreground
```

## Dynamic boot time configuration

Reference: Distillery's [Working with Docker](working_with_docker)

Within `rel/config.exs` add this snippet:

```elixir
set config_providers: [
  {Mix.Releases.Config.Providers.Elixir, ["${RELEASE_ROOT_DIR}/etc/config.exs"]}
]

set overlays: [
  {:copy, "rel/config/config.exs", "etc/config.exs"}
]

set(vm_args: "rel/vm.args.eex")
```

To the bottom of your application's release definition, that is the section that
starts with `release :lime_umbrella do`.

Create `rel/config/config.exs`:

```elixir
config :lime_web, LimeWeb.Endpoint,
  secret_key_base: System.get_env("SECRET_KEY_BASE")
```

Create `rel/vm.args.eex`:

```
-name <%= release_name %>@127.0.0.1
-setcookie ${COOKIE}
-smp auto
```

*Note*: The cookie value above will only be interpolated out of the system
environment at boot time if at that time `REPLACE_OS_VARS=true` is in the system
environment.

## Containerize your Releases

Reference: Distillery's [Working with Docker](working_with_docker)

Create a new `Dockerfile` in your project root with the contents of the
[Alpine Elixir Release](https://github.com/danielspofford/alpine-elixir-release)
Dockerfile.

This is based off of the Dockerfile described in Distillery's 2.0 documentation.

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
SKIP_PHOENIX ?= false
PHOENIX_SUBDIR ?= ./apps/lime_web

help:
    @echo "$(APP_NAME):$(APP_VSN)-$(BUILD)"
    @perl -nle'print $& if m{^[a-zA-Z_-]+:.*?## .*$$}' $(MAKEFILE_LIST) | \
      sort | \
      awk 'BEGIN {FS = ":.*?## "}; {printf "\033[36m%-30s\033[0m %s\n", $$1, $$2}'

build: ## Build the Docker image
  docker build --build-arg APP_NAME=$(APP_NAME) \
    --build-arg APP_VSN=$(APP_VSN) \
    --build-arg SKIP_PHOENIX=$(SKIP_PHOENIX) \
    --build-arg PHOENIX_SUBDIR=$(PHOENIX_SUBDIR) \
    -t $(APP_NAME):$(APP_VSN)-$(BUILD) \
    -t $(APP_NAME):latest .
```

*Note*: If make reports an error mentioning multiple target patterns, you need
to ensure the Makefile is formatted with tabs not spaces.

*Note*: Makefiles like this can be helpful to simplify the build process. If
they start to grow in size or complexity it becomes invaluable to have an
informative help command or documentation.

Ensure help outputs something similar then build the image:

```
$ make help
lime_umbrella:0.1.0-83e046d
build                          Build the Docker image
run                            Run the app in Docker
$ make build
```

TODO: We are going to need that image in a second.

## Generate Infrastructure Folder TODO: Better section name



[ecs_stacker]: content/guides/standing-up-an-ecs-web-stack-with-stacker.md
[phx_walkthrough]: https://hexdocs.pm/distillery/guides/phoenix_walkthrough.html
[working_with_docker]: https://hexdocs.pm/distillery/guides/working_with_docker.html
