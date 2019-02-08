# Clustered Elixir on AWS ECS with CircleCI

This guide will walk you through setting up an [Elixir](elixir) application on
top of AWS infrastructure resulting in:

- Language version management via `asdf-vm/asdf`
- Error reporting via Sentry
- Containerized OTP releases via Docker, [distillery](distillery), and a
  Makefile
- Automated checks and deploys via CircleCI
  - credo
  - dialyzer
  - tests
  - formatting
- Easy boot time configuration via [distillery](distillery) config providers
- Automated clustering via [libcluster](libcluster)

## Prerequisites

An AWS account and credentials.

- [Docker](docker)
- [asdf](asdf)
- [asdf-elixir](asdf-elixir)
- [asdf-erlang](asdf-erlang)
- [asdf-nodejs](asdf-nodejs)

## Create a Phoenix Project

```console
$ mix archive.install hex phx_new 1.4.0
$ mix phx.new my_app --umbrella
Fetch and install dependencies? [Yn]
$ y
...
$ cd my_app_umbrella
```

## Language Management via asdf

Create `my_app_umbrella/.tool-versions`:

```
erlang 21.2.2
elixir 1.7.4-otp-21
nodejs 11.6.0
```

Then run:

```console
$ asdf install
```

## Docker for Local Postgres

Create `my_app_umbrella/docker-compose.yml`:

```
version: '2'
services:
  db:
    image: postgres:11.1
    ports:
     - "5432:5432"
```

Start postgres and create the database:

```console
$ docker-compose up -d
$ mix ecto.create
```

## Dynamic Boot Time Configuration

Reference: Distillery's [Working with Docker](working_with_docker)

Create `my_app_umbrella/rel/commands/migrate.sh`:

```sh
#!/bin/sh

release_ctl eval --mfa "Carson.ReleaseTasks.Migrator.migrate/0"
```

In `my_app_umbrella/apps/my_app/mix.exs` add:

```elixir
{:sentry, "~> 7.0"}
```

Create `my_app_umbrella/apps/my_app/lib/my_app/release_tasks/migrator.ex`:

```elixir
defmodule MyApp.ReleaseTasks.Migrator do
  @moduledoc """
  Module for migrating the database.
  """

  require Logger

  @start_apps [
    :logger,
    :sentry,
    :crypto,
    :ssl,
    :postgrex,
    :ecto
  ]

  @repos [MyApp.Repo]

  def migrate() do
    start_services()

    _ =
      try do
        run_migrations()
        Logger.info("Success!")
      rescue
        exc ->
          {:ok, task} = Sentry.capture_exception(exc, stacktrace: __STACKTRACE__)
          {:ok, _event_id} = Task.await(task)

          exc
          |> inspect
          |> Logger.error()
      end

    stop_services()
  end

  defp start_services do
    _ = Logger.info("Starting dependencies..")
    # Start apps necessary for executing migrations
    Enum.each(@start_apps, &Application.ensure_all_started/1)

    # Start the Repo(s) for app
    _ = Logger.info("Starting repos..")
    Enum.each(@repos, & &1.start_link(pool_size: 1))
  end

  defp stop_services do
    :init.stop()
  end

  defp run_migrations do
    Enum.each(@repos, &run_migrations_for/1)
  end

  defp run_migrations_for(repo) do
    app = Keyword.get(repo.config, :otp_app)
    _ = Logger.info("Running migrations for #{app}")
    migrations_path = priv_path_for(repo, "migrations")
    Ecto.Migrator.run(repo, migrations_path, :up, all: true)
  end

  defp priv_path_for(repo, filename) do
    app = Keyword.get(repo.config, :otp_app)

    repo_underscore =
      repo
      |> Module.split()
      |> List.last()
      |> Macro.underscore()

    priv_dir = "#{:code.priv_dir(app)}"

    Path.join([priv_dir, repo_underscore, filename])
  end
end
```

Edit `my_app_umbrella/rel/config.exs` adding this snippet to the bottom of the
release block:

```elixir
set config_providers: [
  {Mix.Releases.Config.Providers.Elixir, ["${RELEASE_ROOT_DIR}/etc/config.exs"]}
]

set overlays: [
  {:copy, "rel/config/config.exs", "etc/config.exs"}
]

set(commands: [migrate: "rel/commands/migrate.sh"])
```

Create `rel/config/config.exs` containing:

```elixir
use Mix.Config

config :my_app, MyApp.Repo, url: System.get_env("DATABASE_URL")

config :my_app_web, MyAppWeb.Endpoint,
  secret_key_base: System.get_env("SECRET_KEY_BASE")
```

## OTP Releases via Distillery

Reference: Distillery's [Phoenix walkthrough](phx_walkthrough)

In `my_app_umbrella/mix.exs` add:

```elixir
{:distillery, "~> 2.0"}
```

In `my_app_umbrella/apps/my_app/mix.exs` add:

```elixir
{:dialyxir, "~> 1.0.0-rc.4"},
{:credo, "~> 1.0"}
```

From the repository's root:

```console
$ mix do deps.get, compile, release.init
```

In `my_app_umbrella/apps/my_app_web/config/prod.exs` replace the
`MyAppWeb.Endpoint` config  with:

```elixir
config :my_app_web, MyAppWeb.Endpoint,
  http: [port: {:system, "PORT"}],
  url: [host: "localhost", port: {:system, "PORT"}],
  cache_static_manifest: "priv/static/cache_manifest.json",
  server: true,
  root: ".",
  version: Application.spec(:my_app_umbrella, :vsn)
```

In `my_app_umbrella/apps/my_app_web/assets/`:

```console
$ node node_modules/webpack/bin/webpack.js --mode production
$ cd ..
$ mix phx.digest
```

Modify `my_app_umbrella/apps/my_app/config/config.exs` adding:

```elixir
config :my_app, MyApp.Repo,
  adapter: Ecto.Adapters.Postgres,
  pool_size: 10
```

*Note*: Ensure this is added above the `import_config` line.

Replace the repo configuration in `my_app_umbrella/apps/my_app/config/dev.exs`
with:

```elixir
config :my_app, MyApp.Repo,
  url: "postgres://postgres:postgres@localhost/my_app_dev"
```

Replace the repo configuration in `my_app_umbrella/apps/my_app/config/test.exs`
with:

```elixir
config :my_app, MyApp.Repo,
  url: "postgres://postgres:postgres@localhost/my_app_test",
  pool: Ecto.Adapters.SQL.Sandbox
```

Modify `my_app_umbrella/apps/my_app/config/prod.exs` and
`my_app_umbrella/apps/my_app_web/config/prod.exs` removing this line from both:

```elixir
import_config "prod.secret.exs"
```

Delete the `my_app_umbrella/apps/my_app/config/prod.secret.exs` and
`my_app_umbrella/apps/my_app_web/config/prod.secret.exs` files.

Build your release and ensure it starts:

```console
$ MIX_ENV=prod mix release
$ PORT=4001 \
    DATABASE_URL="postgres://postgres:postgres@localhost/my_app_dev" \
    _build/prod/rel/my_app_umbrella/bin/my_app_umbrella foreground
```

## Containerized Releases

Reference: Distillery's [Working with Docker](working_with_docker)

Create a new `my_app_umbrella/Dockerfile` with the contents of the [Alpine
Elixir Release](aer).

This is based off of the Dockerfile described in Distillery's 2.0 documentation.

Where the `PHOENIX_SUBDIR` is declared, set its value to `apps/my_app_web`.

Create a `my_app_umbrella/.dockerignore`:

```
_build/
deps/
.git/
.gitignore
Dockerfile
README*
test/
priv/static/
```

Create `my_app_umbrella/scripts/build.sh`:

```sh
#!/bin/sh

set -e

APP_NAME=my_app_umbrella
APP_VSN=0.1.0
BUILD="$(git rev-parse --short HEAD)"
SKIP_PHOENIX=false
PHOENIX_SUBDIR=./apps/my_app_web

docker build \
  --build-arg APP_NAME=$APP_NAME \
  --build-arg APP_VSN=$APP_VSN \
  --build-arg SKIP_PHOENIX=$SKIP_PHOENIX \
  --build-arg PHOENIX_SUBDIR=$PHOENIX_SUBDIR \
  -t $APP_NAME:$APP_VSN-$BUILD \
  -t $APP_NAME:latest .
```

Run the script:

*Note*: If you haven't committed anything yet, this step will require a commit
to have been made.

```console
$ ./scripts/build.sh
```

Notice that the the last few lines of output:

```
Successfully built 6d494c748e0d
Successfully tagged my_app_umbrella:0.1.0-de2e903
Successfully tagged my_app_umbrella:latest
```

## Create a healthcheck endpoint

Edit
`my_app_umbrella/apps/my_app_web/lib/my_app_web/controllers/page_controller.ex`
adding:

```elixir
def healthcheck_show(conn, _params) do
  render(conn, "healthcheck_show.json")
  conn
  |> send_resp(200, "OK")
  |> halt()
end
```

Edit `my_app_umbrella/apps/my_app_web/lib/my_app_web/router.ex` uncommenting the
`/api` scope and adding:

```elixir
get("/healthcheck", PageController, :healthcheck_show)
```

Edit `my_app_umbrella/apps/my_app_web/lib/my_app_web/views/page_view.ex` adding:

```elixir
def render("healthcheck_show.json", _) do
  %{status: :healthy}
end
```

## Setup AWS infrastructure

Install the `draft` mix archive if you haven't already, this facilitates
command-line execution of arbitrary EEx templates and directories:

```console
$ mix archive.install github danielspofford/draft
```

In the repo's root:

```console
$ mix draft.github danielspofford/draft_aws_ecs --app-name=myappumbrella
$ cd infrastructure
```

Edit the `stacker.yaml` file adding `PORT: "5000"` to the `ContainerEnvironment`
section under the `my_appumbrella-web-stack` section.

*Disclaimer*: Review the Makefile at your leisure but know that these next few
steps are going to create infrastructure on AWS that will cost you money. Not
much, but it will.  There is also an easy command to tear it all down.

Run `make shell`.

The next step will require the following env vars to be set:

- `SECRET_KEY_BASE={some_long_string}`
- `AWS_DEFUALT_REGION=us-east-2`
- `AWS_ACCESS_KEY_ID={some_key}`
- `AWS_SECRET_ACCESS_KEY={some_secret}`

*Note*: Those AWS credentials will be persisted into the ECS container
environment and used by `libcluster_ecs` to interact with the AWS API.

```console
$ STACK=dev make build
```

This will start the process of standing up infrastructure. Once you see that the
ECR has completed, go ahead and CTRL-C to quit the running command, and type
`exit` to exit the container. If you got an error about my_appUMBRELLA_IMAGE not
being set don't worry about it yet we will handle that soon. Run the following
command in the infrastructure folder:

```console
$ aws ecr get-login --no-include-email
```

The output will be a command you should copy and paste and execute, the last
line of output after executing the command should be affirmative, like:
`Login Succeeded`. Notice at the end of that command you entered there is the
URI of your ECR repository, copy that and run:

```console
$ docker tag my_app_umbrella:latest {the URI}:latest
$ docker push {the URI}:latest
```

This will push that image we just built to your ECR repository. Now we want to
set an additional environment variable:

- `my_appUMBRELLA_IMAGE={the URI}:latest`

Then continue standing up infrastructure:

```console
$ make shell
$ STACK=dev make build
```

Notice whatever was stood up before will be noticed and skipped and it will pick
up where it left off. Your app is now up and running and can be viewed by
navigating the the associated ELBs DNS name. This can be found in the AWS
console by going to EC2 then Load Balancers and viewing the associated ELBs
Basic Configuration.

### Tear down AWS infrastructure

If you plan on continuing through this guide then skip this section for now.
Run `make shell` from within the infrastructure folder and then run
`STACK=ds make destroy ARGS=--force`.

*Note*: If your ECR has images in it you will need to delete those yourselves
before the above command will be able to destroy it.

## CI/CD via CircleCI

In the repo's root:

```console
$ mix draft.github danielspofford/draft_aws_ecs_circleci_elixir \
  --app-name=my_app_umbrella \
  --app-version=0.1.0 \
  --production-image={the URI} \
  --cluster=your_cluster_name \
  --service=your_service_name \
  --task=your_task_definition_name \
  --slack-build-channel=your_slack_channel \
  --slack-hook=your_slack_hook
```

If you want you can set the slack keys to gibberish and then modify the
generated `.circleci/config.yml` removing any sections specific to them.

Edit `.circleci/config.yml` and find the `docker build` command and before the
`-t` add `--build-arg SKIP_PHOENIX=false`.

Commit everything and push it up to master then login to CircleCI with your
GitHub account and follow / start building this repository.

Sign up for a [Sentry] account if you don't have one already. You will need to
create an auth token with these scopes `project:releases`, `org:write`. Also,
add your github repository to your sentry account.

In the CircleCI settings for your project add these environment variables:

- `DATABASE_URL=postgres://postgres:postgres@localhost/db`
- `AWS_ACCESS_KEY_ID=your_key`
- `AWS_SECRET_ACCESS_KEY=your_secret`
- `SENTRY_AUTH_TOKEN=your_token`
- `SENTRY_DSN=your_dsn`
- `SENTRY_ORG=your_org`
- `SENTRY_PROJECT=your_project`
- `AWS_DEFAULT_REGION=us-east-2`

## Clustering

In `apps/my_app/mix.exs`, add:

```elixir
{:libcluster, "~> 3.0"},
{:libcluster_ecs, github: "verypossible/libcluster_ecs"}
```

Then run:

```console
$ mix deps.get
```

Within the `start/2` function of `apps/my_app/lib/my_app/application.ex` define
this:

```elixir
topologies = [example: [strategy: ClusterECS.Strategy.Metadata]]
```

Then add this to the list of children to be started:

```elixir
{Cluster.Supervisor, [topologies, [name: MyApp.ClusterSupervisor]]}
```

Create a file `rel/config/vm.args.eex` with the follow contents:

```
-name app@${HOSTNAME}

-setcookie ${COOKIE}
```

Commit and push all your changes on master to trigger a deploy.


[aer]: https://github.com/danielspofford/alpine-elixir-release/blob/v1.0.0/Dockerfile
[asdf-elixir]: https://github.com/asdf-vm/asdf-elixir
[asdf-erlang]: https://github.com/asdf-vm/asdf-erlang
[asdf-nodejs]: https://github.com/asdf-vm/asdf-nodejs
[asdf]: https://github.com/asdf-vm/asdf
[distillery]: https://github.com/bitwalker/distillery
[docker]: https://www.docker.com/
[libcluster]: https://github.com/bitwalker/libcluster
[phx_walkthrough]: https://hexdocs.pm/distillery/guides/phoenix_walkthrough.html
[working_with_docker]: https://hexdocs.pm/distillery/guides/working_with_docker.html
