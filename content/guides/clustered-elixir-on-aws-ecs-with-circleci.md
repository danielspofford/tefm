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
- Docker

## Create a Phoenix Project

```
$ mix phx.new lime --umbrella
Fetch and install dependencies? [Yn]
$ y
```

## Docker for Local Postgres

Create a `docker-compose.yml` file in the repo's root with the follow contents:

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

Add `{:distillery, "~> 2.0"}` to the top level `mix.exs` and then:

*Note*: This next dep change is only needed because of
(this issue)[https://github.com/phoenixframework/phoenix/issues/3119] and will
go away when Phoenix 1.4 is released and this guide is updated.

In `apps/lime_web/mix.exs`, replace `{:cowboy, "~> 1.0"}` with
`{:plug_cowboy, "~> 1.0"}`.

From repo's root:

```
$ mix do deps.get, compile, release.init
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

Within `rel/config.exs` add this snippet to the bottom of your application's
release definition, that is the section that starts with
`release :lime_umbrella do`.

```elixir
set config_providers: [
  {Mix.Releases.Config.Providers.Elixir, ["${RELEASE_ROOT_DIR}/etc/config.exs"]}
]

set overlays: [
  {:copy, "rel/config/config.exs", "etc/config.exs"}
]

set(vm_args: "rel/vm.args.eex")
```


Create `rel/config/config.exs` containing:

```elixir
use Mix.Config

config :lime_web, LimeWeb.Endpoint,
  secret_key_base: System.get_env("SECRET_KEY_BASE")
```

Create `rel/vm.args.eex` containing:

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

Create a new `Dockerfile` in your repo's root with the contents of the
[Alpine Elixir Release](https://github.com/danielspofford/alpine-elixir-release)
Dockerfile.

This is based off of the Dockerfile described in Distillery's 2.0 documentation.

Create a `.dockerignore` in your repo's root:

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

Create a `Makefile` in your repo's root:

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

Make sure everything you have done so far is commited then ensure help outputs
something similar then build the image:

```
$ make help
lime_umbrella:0.1.0-de2e903
build                          Build the Docker image
$ make build
```

Notice that the the last few lines of output:

```
Successfully built 6d494c748e0d
Successfully tagged lime_umbrella:0.1.0-de2e903
Successfully tagged lime_umbrella:latest
```

## Setup AWS infrastructure

Install the `draft` mix archive if you haven't already, this facilitates
command-line execution of arbitrary EEx templates and directories:

```
$ mix archive.install github danielspofford/draft
```

In the repo's root:

```
$ mix draft.github danielspofford/draft_aws_ecs --app-name=limeumbrella
$ cd infrastructure
```

*Disclaimer*: Review the Makefile at your leisure but know that these next few
steps are going to create infrastructure on AWS that will cost you money. Not
much, but it will.  There is also an easy command to tear it all down.

Run `make shell`.

The next step will require the following env vars to be set:

- `SECRET_KEY_BASE={some_long_string}`
- `AWS_DEFUALT_REGION=us-east-2`
- `AWS_ACCESS_KEY_ID={some_key}`
- `AWS_SECRET_ACCESS_KEY={some_secret}`

Notice that last var starts with your `--app-name` upcased. Run:

```
STACK=dev make build
```

This will start the process of standing up infrastructure. Once you see that the
ECR has completed, go ahead and CTRL-C to quit the running command, and type
`exit` to exit the container. If you got an error about LIMEUMBRELLA_IMAGE not
being set don't worry about it yet we will handle that soon. Run the following
command in the infrastructure folder:

```
$ aws ecr get-login --no-include-email
```

The output will be a command you should copy and paste and execute, the last
line of output after executing the command should be affirmative, like:
`Login Succeeded`. Notice at the end of that command you entered there is the
URI of your ECR repository, copy that and run:

```
$ docker tag lime_umbrella:latest {the URI}:latest
$ docker push {the URI}:latest
```

This will push that image we just built to your ECR repository. Now we want to
set an additional environment variable:

- `LIMEUMBRELLA_IMAGE={the URI}:latest`

Then continue standing up infrastructure:

```
$ make shell
$ STACK=dev make build
```

Notice whatever was stood up before will be noticed and skipped and it will pick
up where it left off.

## CI/CD via CircleCI

TODO: write me

## Tear down AWS infrastructure

Run `make shell` from within the infrastructure folder and then run `STACK=ds
make destroy ARGS=--force`.

[phx_walkthrough]: https://hexdocs.pm/distillery/guides/phoenix_walkthrough.html
[working_with_docker]: https://hexdocs.pm/distillery/guides/working_with_docker.html
