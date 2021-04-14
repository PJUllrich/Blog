---
layout: post
title:  How to deploy a Phoenix application with Fly.io
description: Fly.io allows you to quickly deploy a Phoenix application in different regions in the world. Here is how to get started.
date:   2021-04-11 07:01:35 +0200
tags:   [development, elixir]
---

José Valim recently twittered about the [Liveview-Cluster](https://github.com/fly-apps/phoenix-liveview-cluster) project, which showcased how state can be shared among a global cluster of Elixir nodes.

{% twitter https://twitter.com/josevalim/status/1380524993885388804 %}

I found this project impressive, because the global state (a simple counter) was shared by 17 machines all around the globe. What I found even more impressive was [Fly.io](https://fly.io), their server provider. It allowed them to deploy machines in 17 different regions with relative ease.

I tried out Fly.io recently and would like to share how to get started with it. I will create a new [Phoenix](https://phoenixframework.org) project, upload it to fly.io as a Docker image, and provision the application with a Postgres instance. Let's get started!

## Creating the application

First, let's create a new Phoenix application with:
```bash
mix phx.new my_app --live
```

This will create a basic [Phoenix LiveView](https://github.com/phoenixframework/phoenix_live_view) project with a database connection using [Ecto](https://hexdocs.pm/ecto/Ecto.html). We want to run the database migration every time the application starts. Since we will create a [Mix Release](https://hexdocs.pm/mix/Mix.Tasks.Release.html) for the application, we cannot use [mix](https://hexdocs.pm/mix/Mix.html) for migrating the database. The recommended solution is to create a custom `Release` module, which takes care of migrating the repo. 

Create a `/lib/my_app/release.ex` file and copy the following code into it:

```elixir
defmodule MyApp.Release do
  @app :my_app

  def migrate do
    load_app()

    for repo <- repos() do
      {:ok, _, _} =
        Ecto.Migrator.with_repo(
          repo,
          &Ecto.Migrator.run(&1, :up, all: true)
        )
    end
  end

  def rollback(repo, version) do
    load_app()

    {:ok, _, _} =
      Ecto.Migrator.with_repo(
        repo,
        &Ecto.Migrator.run(&1, :down, to: version)
      )
  end

  defp repos do
    Application.fetch_env!(@app, :ecto_repos)
  end

  defp load_app do
    Application.load(@app)
  end
end
```

Now, whenever we want to migrate the database, we can call `MyApp.Release.migrate()` instead of `mix ecto.setup`. We will use this in the following section when we start our Docker image.

As the last step, we need to configure our application with a `/config/release.exs` file. This allows us to read the environment variables when the application is started (at runtime) instead of when the Docker image is built (at compile time). 

Create a `/config/release.exs` file and copy the following code into it:

```elixir
import Config

secret_key_base =
  System.get_env("SECRET_KEY_BASE") ||
    raise """
    environment variable SECRET_KEY_BASE is missing.
    You can generate one by calling: mix phx.gen.secret
    """

app_name =
  System.get_env("FLY_APP_NAME") ||
    raise "FLY_APP_NAME not available"

database_url =
  System.get_env("DATABASE_URL") ||
    raise """
    environment variable DATABASE_URL is missing.
    For example: ecto://USER:PASS@HOST/DATABASE
    """

config :my_app, MyApp.Repo,
  url: database_url,
  # DON'T FORGET THE FOLLOWING LINE
  socket_options: [:inet6],
  pool_size: String.to_integer(System.get_env("POOL_SIZE") || "10")

config :my_app, MyAppWeb.Endpoint,
  server: true,
  secret_key_base: secret_key_base,
  load_from_system_env: true,
  http: [port: {:system, "PORT"}],
  url: [scheme: "https", host: "#{app_name}.fly.dev", port: 443],
  force_ssl: [rewrite_on: [:x_forwarded_proto]],
  cache_static_manifest: "priv/static/cache_manifest.json"

# Do not include metadata nor timestamps in development logs
config :logger, :console, format: "[$level] $message\n"
```

Make sure that you defined the `socket_options: [:inet6]` option for your `Repo`. Otherwise, your repo won't be able to talk to the provisioned Postgres instance, which requires a connection via `IPv6`. This info was brought to you by [a very kind engineer](https://elixirforum.com/t/use-fly-io-internal-dns-for-resolving-database-url/38889) at fly.io.

## The Docker setup

Fly.io only supports deploying Docker images as of now. We need to create a Dockerfile to build and deploy a Docker image with our application inside of it. 

Create a `Dockerfile` and copy the following code into it:

```dockerfile
# Stage 1: Build a Mix.Release of the application (image size: ~750mb)
FROM bitwalker/alpine-elixir-phoenix:latest AS phx-builder

WORKDIR /app

# These two environment variables will be overwritten when the application is started.
# They are needed here to satisfy the env-variable checks in `prod.secret.exs`
ENV SECRET_KEY_BASE=nokey
ENV DATABASE_URL=nodb

# If you set the PORT to 4000, you need to change it in the fly.toml as well.
ENV PORT=4000
ENV MIX_ENV=prod

# Cache elixir deps
ADD mix.exs mix.lock ./
RUN mix do deps.get --only prod, deps.compile

# Cache npm deps
ADD assets/package.json assets/
RUN npm install --prefix assets

# Copy all local files to the build context
# Ignores the ones specified in .dockerignore
ADD . .

# Run frontend build, compile, and digest assets
RUN npm run --prefix assets deploy
RUN mix do compile, phx.digest

# Create a Mix.Release of the application
RUN mix release

# Stage 2: Create a smaller deployment image (image size: ~98mb)
FROM bitwalker/alpine-elixir:latest

# Make sure that this PORT is equal to the one above and to the one in fly.toml
ENV PORT=4000
ENV MIX_ENV=prod

WORKDIR /app

# Create a unprivileged user to run the app
#
# This is a common security practice to avoid
# giving root permissions to the application which attackers 
# could potentially abuse if they gain access to the application.
ENV USER="phoenix"
ENV HOME=/home/"${USER}"
ENV APP_DIR="${HOME}/app"
RUN \
  addgroup \
  -g 1000 \
  -S "${USER}" && \
  adduser \
  -s /bin/sh \
  -u 1000 \
  -G "${USER}" \
  -h "${HOME}" \
  -D "${USER}" && \
  su "${USER}" sh -c "mkdir ${APP_DIR}"

# Copy the files necessary to run the application
COPY --from=phx-builder --chown="${USER}":"${USER}" /app/_build/prod/rel/my_app ./
COPY --from=phx-builder --chown="${USER}":"${USER}" /app/entrypoint.sh ./

# Define the entrypoint and the command it should execute
ENTRYPOINT ["/app/entrypoint.sh"]
CMD ["bin/my_app", "start"]
```

This Dockerfile defines a [multi-stage build process](https://hexdocs.pm/phoenix/releases.html#containers) and creates a [Mix.Release](https://hexdocs.pm/mix/Mix.Tasks.Release.html) of the application. Alex Koutmos wrote about this topic in great depth. I recommend reading [his fabulous blog post](https://akoutmos.com/post/multipart-docker-and-elixir-1.9-releases/) to learn more about it.

To speed up the build process you can define the following rules in your `.dockerignore` file. They will prevent Docker from uploading the specified folders to the build context.

```
assets/node_modules
_build
deps
test
mix.lock
package-lock.json
```

The last step is to create an `entrypoint.sh` script, which migrates our database and starts our application.

Create a `entrypoint.sh` file and copy the following code into it:

```bash
#!/bin/bash
# Docker entrypoint script.

/app/bin/my_app eval "MyApp.Release.migrate"

exec $@
```

We use Mix.Release's [eval](https://hexdocs.pm/mix/Mix.Tasks.Release.html#module-one-off-commands-eval-and-rpc) command to run the `migrate` function of our `MyApp.Release` module. This will migrate the database for us. We don't need to create the database first since fly.io will take care of that for us.

## Deploying to fly.io

We created an application and configured our Docker build process. Now, it's time to let our application fly.io (Badumm, tsssss).

If you haven't already, please install the [flyctl](https://fly.io/docs/getting-started/installing-flyctl/) command line interface and sign up or log in to your fly.io account.

```bash
# To sign up
flyctl signup

# To log in
flyctl login
```

Now that you are logged in, let's first provision our app with a `postgres` instance. In order to create a new postgres cluster, run:

```bash
flyctl postgres create
```
This command will ask you a few questions. I added example answers below:
```bash
? App name: my-app-postgres
Automatically selected personal organization: Your Name
? Select region: fra (Frankfurt, Germany)
? Select VM size: shared-cpu-**1x** - 256
? Volume size (GB): 10
Creating postgres cluster my-app-postgres in organization personal
Postgres cluster my-app-postgres created
  Username:    postgres
  Password:    BASE64-PASSWORD-HERE
  Hostname:    my-app-postgres.internal
  Proxy Port:  5432
  PG Port: 5433
Save your credentials in a secure place, 
you will not be able to see them again!

Monitoring Deployment

2 desired, 2 placed, 2 healthy, 0 unhealthy 
[health checks: 6 total, 6 passing]
--> v0 deployed successfully

Connect to postgres
Any app within the personal organization can connect to postgres using the 
above credentials and the hostname "my-app-postgres.internal."
For example: 
postgres://postgres:BASE64-PASSWORD-HERE@my-app-postgres.internal:5432
```

We received the `DATABASE_URL` in the last section here. Normally, we would need to set it manually for our application, but don't worry since we will let fly.io set this environment variable automatically.

Next, let's create an `app` on fly.io.:

```bash
# This will generate a random app name
flyctl launch

# To set the app name yourself, run:
flyctl launch --name my-app
```

If you run the above command, you can choose a region for the app. Don't worry. You can always change and add regions afterward. For now, select a region close to you.

This will create the `fly.toml` file, which configures the deployment to fly.io.

```toml
# 1
app = "my-app"

kill_signal = "SIGINT"
kill_timeout = 5

# DON'T FORGET THE FOLLOWING 2 LINES
# OTHERWISE YOUR REPO WON'T CONNECT TO POSTGRES
[experimental]
  private_network=true

[[services]]
  internal_port = 4000
  protocol = "tcp"

  [services.concurrency]
    hard_limit = 25
    soft_limit = 20

  [[services.ports]]
    handlers = ["http"]
    port = "80"

  [[services.ports]]
    handlers = ["tls", "http"]
    port = "443"

  [[services.tcp_checks]]
    grace_period = "1s"
    interval = "15s"
    port = "4000"
    restart_limit = 6
    timeout = "2s"
```

Before we deploy the first version of our application, we must not forget to attach the Postgres database to the application:

```bash
flyctl postgres attach --postgres-app my-app-postgres
```

This will add the `DATABASE_URL` environment variable to your application.

Finally, let's deploy the application with:

```bash
flyctl deploy
```

This command will build, upload, and start the Docker container of your application. It might take a minute or two.

After the deployment process is done, you should be able to access your application at `my-app.fly.dev`. If not, please don't hesitate to contact the folks at fly.io either on their [community board](https://community.fly.io/), or me on [Twitter](https://twitter.com/pjullrich).