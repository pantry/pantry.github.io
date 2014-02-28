---
layout: default
title: Pantry
subtitle: Getting Started
github: pantry
summary: Modern DevOps Automation
sub_nav: 0
---

# Getting Started with Pantry

Pantry is written in [Ruby](https://www.ruby-lang.org/) and distributed via [Rubygems](http://rubygems.org/gems/pantry). Install the core library on every server you want to manage with Pantry:

```
gem install pantry
```

## Plugins

Adding functionality to Pantry is done via Plugins. Plugins should also be distributed as a Gem, e.g. to install [Pantry Chef](/chef) run:

```
gem install pantry-chef
```

Pantry and all plugins must be installed on all servers in a Pantry network. Information about writing plugins is available in [Extending Pantry](/core/extending.html).

## Dependencies

Pantry requires ZeroMQ 3.x or later. Most \*nix distributions have this in their respective package managers. However if you want to enable Pantry's `curve` security, ZeroMQ 4.x is required, which has not yet made it through all of the normal packaging channels. That said even if the package manager has ZeroMQ 4.x there is a good chance it still isn't ready, because `curve` security requires a ZeroMQ built and linked against the [libsodium](https://github.com/jedisct1/libsodium) library, so there's a good chance ZeroMQ will need to be built from source.

## Configuration and Run Scripts

Pantry provides configuration files and run scripts in the main repository's [dist](https://github.com/pantry/pantry/blob/master/dist/) directory. More information on configuring Pantry is available in [Configuration](/core/configure.html).

## Your First Commands

Once a Pantry Server is running, test that it is running properly with the `status` command from the CLI:

```
pantry --host [server host name] status
```

This command should return a single line saying something like "A minute ago $USER". This output confirms that the Server is running and is accessible. If you then start up a Client to connect to this Server and re-run `status` you should see two responses: your CLI like before and the new Client. If so, you're set up and ready to go!

## Configuring Security

By default Pantry does not enable any security. To configure Pantry with `curve` or other security setups, please see [Security](/core/security.html) for the details and setup instructions.
