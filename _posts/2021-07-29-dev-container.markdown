---
layout: post
title: "Set up Dev Container for an Elixir Phoenix project"
date: 2021-07-29 21:42:01 +0800
---

## A brief on Dev Container

[Dev Container](https://code.visualstudio.com/docs/remote/create-dev-container) is a mechanism to set up a full-featured local development environment. By using VSCode as the editor, remote connecting to a Docker container (which running a fully functional development environment), developers can enjoy a great local development experience, while taking full advantage of the container technology.

## Prerequisites

To make the Dev Container works, the following software are required:

- [Docker Desktop](https://www.docker.com/products/docker-desktop)
- [VSCode](https://code.visualstudio.com) with [Remote-Container extension](https://github.com/Microsoft/vscode-remote-release) installed.

## A step-by-step guide

### A minimal Dev Container definition

Now, let's take the [Transhook](https://github.com/linjunpop/transhook) project as an example, set up the Dev Container step by step.

The first step is to create a new directory `.devcontainer` in your project's root directory, this directory will be the single place for the Dev Container definitions and some related files.

Now let's add the definition file `.devcontainer/devcontainer.json` for this project:

```json
{
  "image": "hexpm/elixir:1.12.2-erlang-24.0.4-ubuntu-focal-20210325",
  "name": "transhook-devcontainer",
  "onCreateCommand": "elixir --version",
  "forwardPorts": [4000]
}
```

Then try "Open Folder in Container..."

![Open folder in container menu](/assets/images/2021/07/29/dev-container-1.png)

The VSCode window will restart and connect to the container. After the container successfully running, you can see the result of `onCreateCommand` which show the elixir version here:

![The result of onCreateCommand](/assets/images/2021/07/29/dev-container-2.png)

```bash
root@aa36190d0d2f:/workspaces/transhook# elixir --version
Erlang/OTP 24 [erts-12.0.3] [source] [64-bit] [smp:4:4] [ds:4:4:10] [async-threads:1]

Elixir 1.12.2 (compiled with Erlang/OTP 24)
```

Then let's open the `mix.exs` file:

![Elixir file without syntax highlights](/assets/images/2021/07/29/dev-container-3.png)

As you can see, there are no syntax highlights for the Elixir files. Let move to the next step to make the syntax highlighting work.

### VSCode extensions

Dev Container also can bundle the VSCode extensions for the project. Let's go ahead and add the `extensions` section to the `.devcontainer/devcontainer.json` file:

```json
{
  "image": "hexpm/elixir:1.12.2-erlang-24.0.4-ubuntu-focal-20210325",
  "name": "transhook-devcontainer",
  "onCreateCommand": "elixir --version",
  "extensions": ["jakebecker.elixir-ls", "eamodio.gitlens"],
  "forwardPorts": [4000]
}
```

When you reopen the project in Container, VSCode will detect there's a change, then ask you to rebuild the container.

![VSCode detect changes and ask for a rebuild](/assets/images/2021/07/29/dev-container-4.png)

After the rebuild is finished, VSCode will get the ElixirLS and GitLens extension installed. Now the Elixir files got syntax highlighting:

![Elixir file with syntax highlights](/assets/images/2021/07/29/dev-container-5.png)

But as you can see, VSCode complains that the Git is missing, that's because there's the only Elixir installed in the image: `hexpm/elixir:1.12.2-erlang-24.0.4-ubuntu-focal-20210325`. But Git is required for our development, can we find a way to add it to the Dev Container? Let's move on.

### Using the Docker Compose approach

How can we add extra software and configuration to an existing Docker image? Use [Dockerfile](https://docs.docker.com/engine/reference/builder/)!

The Dev Container also supports building from a `Dockerfile` or even [Docker Compose](https://docs.docker.com/compose/), here I will take the second one, I'll show you why in the following content.

In the [Transhook](https://github.com/linjunpop/transhook) project, I'm using [asdf](https://asdf-vm.com) to manage tool versions, so Instead of build upon the `hexpm/elixir`, I will use the `ubuntu` as the base image, then add essential tools (Elixir, Erlang, Node.js) to the dev environment.

What we need to do is to modify the `.devcontainer/devcontainer.json` file, and add two extra new files: `.devcontainer/Dockerfile` and `.devcontainer/docker-compose.yml`.

In the `Dockerfile`, we installed [asdf](https://asdf-vm.com), and Elixir, Erlang, Nodejs based on the versions defined in the project's `.tool-versions` file:

```dockerfile
FROM ubuntu as dev

RUN apt-get update -qq && \
  apt-get install -qq -y \
  curl \
  git \
  dirmngr \
  gpg \
  gawk \
  unzip \
  build-essential \
  autoconf \
  libssl-dev \
  libncurses5-dev \
  m4 \
  libssh-dev

RUN useradd -ms $(which bash) asdf

USER asdf

RUN git clone https://github.com/asdf-vm/asdf.git $HOME/.asdf --branch v0.8.1 && \
  echo '. $HOME/.asdf/asdf.sh' >> $HOME/.bashrc && \
  echo '. $HOME/.asdf/asdf.sh' >> $HOME/.profile

ENV PATH /home/asdf/.asdf/bin:/home/asdf/.asdf/shims:$PATH

RUN /bin/bash -c "\
  asdf plugin-add elixir && \
  asdf plugin-add erlang && \
  asdf plugin-add nodejs \
  "

WORKDIR /app

COPY .tool-versions /app

RUN /bin/bash -c "ls -la && asdf install"

ENV LANG C.UTF-8

WORKDIR /workspace
```

In the `docker-compose.yml` file, the service `app_dev` will be defined to build the image and mount the project into the container's `/workspace/transhook` directory:

```yaml
version: "3.6"

services:
  app_dev:
    build:
      # Set the context to the parent directory, so we can add `.tool-versions` to the container
      context: ../
      dockerfile: .devcontainer/Dockerfile
    environment:
      MIX_ENV: dev
    volumes:
      - ../:/workspace/transhook

    # Overrides default command so things don't shut down after the process ends.
    command: bash -c "sleep infinity"
```

Then we will modify the `devcontainer.json` to tell VSCode we need to build the Dev Container from a Docker Compose file:

```json
{
  "dockerComposeFile": ["docker-compose.yml"],
  "workspaceFolder": "/workspace/transhook",
  "service": "app_dev",
  "extensions": [
    "jakebecker.elixir-ls",
    "eamodio.gitlens",
    "streetsidesoftware.code-spell-checker"
  ],
  "forwardPorts": [4000]
}
```

After a rebuild, everything should up and running:

```bash
asdf@bcdc19f598ee:/workspace/transhook$ cat .tool-versions
erlang 24.0.2
elixir 1.12.1-otp-24
nodejs 16.5.0
asdf@bcdc19f598ee:/workspace/transhook$ elixir --version
warning: the VM is running with native name encoding of latin1 which may cause Elixir to malfunction as it expects utf8. Please ensure your locale is set to UTF-8 (which can be verified by running "locale" in your shell)
Erlang/OTP 24 [erts-12.0.2] [source] [64-bit] [smp:4:4] [ds:4:4:10] [async-threads:1]

Elixir 1.12.1 (compiled with Erlang/OTP 24)
asdf@bcdc19f598ee:/workspace/transhook$ node --version
v16.5.0
```

Till now, we've set up a Dev Container with a fully functional Elixir development environment, we can start coding.

![Successfully setup the Elixir environment](/assets/images/2021/07/29/dev-container-6.png)

Can we?

```
asdf@cde62f300dea:/workspace/transhook$ iex -S mix phx.server
Erlang/OTP 24 [erts-12.0.2] [source] [64-bit] [smp:4:4] [ds:4:4:10] [async-threads:1]

Compiling 33 files (.ex)

Generated transhook app
[error] Postgrex.Protocol (#PID<0.623.0>) failed to connect: ** (DBConnection.ConnectionError) tcp connect (localhost:5432): connection refused - :econnrefused
...
[info] Running TranshookWeb.Endpoint with cowboy 2.9.0 at 0.0.0.0:4000 (http)
[info] Access TranshookWeb.Endpoint at http://localhost:4000
Interactive Elixir (1.12.1) - press Ctrl+C to exit (type h() ENTER for help)
iex(1)>
```

The webserver failed to start because the application is failed to connect to the Postgres database. This is the last issue we need to resolve.

### Add Postgres as a part of the Dev Container

In the previous steps, we did successfully set up the Elixir development environment. But in practice, a complicated production application will rely on many third-party tools, can we bundle them into the Dev Container system too? The answer is definitely yes. Take Transhook as an example, it's built with the Phoenix framework, so Postgres will be an underlying dependency as the data storage system.

Now let's add Postgres to the `docker-compose.yml` as the `db` service:

```
version: "3.6"

services:
  app_dev:
    build:
      # Set the context to the parent directory, so we can add `.tool-versions` to the container
      context: ../
      dockerfile: .devcontainer/Dockerfile
    environment:
      MIX_ENV: dev
    volumes:
      - ../:/workspace/transhook

    # Overrides default command so things don't shut down after the process ends.
    command: sleep infinity
    depends_on:
      - db
  db:
    environment:
      PGDATA: /var/lib/postgresql/data/pgdata
      POSTGRES_PASSWORD: postgres
      POSTGRES_USER: postgres
      POSTGRES_HOST_AUTH_METHOD: trust
    image: 'postgres:11-alpine'
    restart: always
    volumes:
      - 'pgdata:/var/lib/postgresql/data'
volumes:
  pgdata:
```

Now change the database hostname to `db` in `config/dev.exs`:

```elixir
# Configure your database
config :transhook, Transhook.Repo,
  username: "postgres",
  password: "postgres",
  database: "transhook_dev",
  hostname: "db",
  show_sensitive_data_on_connection_error: true,
  pool_size: 10
```

Then another rebuild, Postgres will be up and running, and we can successfully connect to it and bootstrap the database.

```bash
asdf@61d774e9052e:/workspace/transhook$ mix ecto.setup
The database for Transhook.Repo has been created

14:56:08.298 [info]  == Running 20200429042810 Transhook.Repo.Migrations.CreateHooks.change/0 forward

14:56:08.342 [info]  create table hooks

14:56:08.443 [info]  == Migrated 20200429042810 in 0.0s

14:56:08.601 [info]  == Running 20210401074039 Transhook.Repo.Migrations.AddFiltersToHooks.change/0 forward

14:56:08.601 [info]  alter table hooks

...
```

And the web server can be successfully started now:

```bash
asdf@61d774e9052e:/workspace/transhook$ iex -S mix phx.server
Erlang/OTP 24 [erts-12.0.2] [source] [64-bit] [smp:4:4] [ds:4:4:10] [async-threads:1]

[info] Running TranshookWeb.Endpoint with cowboy 2.9.0 at 0.0.0.0:4000 (http)
[info] Access TranshookWeb.Endpoint at http://localhost:4000
Interactive Elixir (1.12.1) - press Ctrl+C to exit (type h() ENTER for help)
iex(1)>
```

You can visit the server from the host machine, on the forwarded port `4000`:

![The index page for the Transhook app](/assets/images/2021/07/29/dev-container-7.png)

### We did it!

With the Dev Container mechanism, we've turned VSCode into a full-featured development environment.

You can refer to this Pull Request [Add Dev Container support](https://github.com/linjunpop/transhook/pull/1) for a reference of files described in this blog post.

## Why

If you read through the above content, you might be wondering why would we spend time to set up such an environment, here I'll put in my two cents, and you are welcome to share yours.

### Pros

- The Dev Container can provide a clean dev environment, which can keep consistant with the production runtime environment.
- New contributors can easily set up the dev environment, especially when the project is complex and relying on many 3rd party tools/services.
- VSCode extensions can be also included in the Dev Container definition, so developers can enjoy the same editing environment. Also, with a `$ git pull`, new extensions can be pull and installed automatically.
- As [GitHub Codespaces](https://github.com/features/codespaces) support Dev Container too, so a project hosted on GitHub might provide a cloud editing environment, which could allow you to edit and ship projects on an iPad. (I had try this for the Transhook project, and it works very well if the network is stable.)

### Cons

- As the Dev Container relies on Docker, it might consume more resources than a well-setup local dev environment, It's a trade-off. On my MacBook Air with 8G memory, sometimes I'll receive memory out warnings, the virtual machine uses around 5G)
