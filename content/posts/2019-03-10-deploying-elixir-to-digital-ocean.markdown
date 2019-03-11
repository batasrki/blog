---
title: "Deploying Elixir to Digital Ocean"
date: 2019-03-09T22:40:52-05:00
---

## Deploying an Elixir/Phoenix app to Digital Ocean

This is a quick post on some gotchas I ran into as I was trying to deploy to my DO droplet. At the bottom of the post are the Dockerfile and the build shell script I'm using, courtesy of [Distillery's documentation](https://hexdocs.pm/distillery/), with some modifications.

## Releases and architectures

To be fair, there are probably easier ways to get an Elixir app running. I could've pulled the code from Github and built on the same VM, much like any Ruby app. However, the BEAM ecosystem allows me to build a whole binary, Erlang runtime included, and just sync that up to the server. I don't need to have Erlang installed.

Trouble is, Elixir and Erlang releases must be built on the same computer architecture as the machine I'm deploying to. I, currently, do not own a Ubuntu system. Therefore, I had to figure out how to build a release on the Ubuntu architecture from my MacBook Pro. The obvious answer is obviously Docker. I am not as yet comfortable with Docker. I did find a [good tutorial](https://hexdocs.pm/distillery/guides/building_in_docker.html) on how to build a release using Docker. It is straightforward, but it will take me time to understand the provided shell script.

I did need to immediately amend the script and Dockerfile, since part of my app uses Phoenix. The Phoenix framework, much like Rails, needs to have its static assets compiled using NodeJS. The Dockerfile therefore has to include an installation NodeJS, and the shell script that's executed in the context of the Docker image needs to invoke the appropriate command.

## Errors on startup

Once I had a viable build and had it `scp`-ing to my DO droplet, I ran the provided start-up script file, `bin/<app name> start`, only to see the process fail. I still have to figure out how to examine Erlang crash dumps. There's a [recommended guide](http://erlang.org/doc/apps/erts/crash_dump.html), but I haven't gone through it yet. As such, I ran `bin/<app_name> console` in order to get a REPL going and debug the issue.

This is where I was a bit disappointed. The error, `shutdown: failed to start child: <App name>Web.Endpoint` on starting up isn't helpful at all. Is the error because I didn't start an application my app depends on? Is it a misconfiguration of some kind? Is it something else? I don't know. I would like to see a more descriptive message, so I can figure out what went wrong. This is especially so as I'm new to Elixir. Using the REPL helped out, as figuring out get a more readable output of the crash dump file.

### Reading the output

One of the really nice things in Erlang is the availability of the `observer` module. Invoking this module starts up a graphical interface and pulls in statistics from the running system. As a companion, there is a `crashdump_viewer` module, which allows me to load in the `erl_crash.dump` file from the filesystem and lets me inspect it through the visual means.

This gave me a clue that I could use to search the internets with. It turns out that I should be starting up applications my app depends on before trying to start mine. I feel that this is a common enough of a mistake that there should be an error message specific to it. Once I included all expected applications, things worked nicely.

## Thoughts and files

In the end, I'm happy that I can build a release on my OSX laptop and ship it to my Ubuntu DO droplet without having to install Erlang, Elixir, Hex on the production system.

Here are the `Dockerfile` and the build script. They're mostly straight from the Distillery's guides, but adjusted to also build static assets for Phoenix.

```Dockerfile
FROM ubuntu:16.04

ENV REFRESHED_AT=2019-03-10 \
    LANG=en_US.UTF-8 \
    HOME=/opt/build \
    TERM=xterm

WORKDIR /opt/build

RUN \
  apt-get update -y && \
  apt-get install -y git wget vim locales && \
  locale-gen en_US.UTF-8 && \
  wget https://packages.erlang-solutions.com/erlang-solutions_1.0_all.deb && \
  dpkg -i erlang-solutions_1.0_all.deb && \
  rm erlang-solutions_1.0_all.deb && \
  apt-get update -y && \
  apt-get install -y erlang elixir curl && \
  curl -sL https://deb.nodesource.com/setup_11.x | bash - && \
  apt-get install -y nodejs

CMD ["/bin/bash"]

```

```Build shell script
#!/usr/bin/env bash

set -e

cd /opt/build

APP_NAME="$(grep 'app:' mix.exs | sed -e 's/\[//g' -e 's/ //g' -e 's/app://' -e 's/[:,]//g')"
APP_VSN="$(grep 'version:' mix.exs | cut -d '"' -f2)"

mkdir -p /opt/build/rel/artifacts

# Install updated versions of hex/rebar
mix local.rebar --force
mix local.hex --if-missing --force

export MIX_ENV=prod

# Fetch deps and compile
mix deps.get --only prod
# Run an explicit clean to remove any build artifacts from the host
mix do clean, compile --force

# Build the digest
cd assets
node node_modules/brunch/bin/brunch build --production
cd ..
mix phx.digest

# Build the release
mix release --env=prod
# Copy tarball to output
cp "_build/prod/rel/$APP_NAME/releases/$APP_VSN/$APP_NAME.tar.gz" rel/artifacts/"$APP_NAME-$APP_VSN.tar.gz"

exit 0

```
