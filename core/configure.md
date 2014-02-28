---
layout: default
title: Pantry Core
subtitle: Configuration
github: pantry
summary: Async Client/Server Remote Execution Framework
sub_nav: 1
---

# Configuring Pantry

Pantry works to follow convention over configuration, but understands that no server infrastructure is the same and as such supports custom configuration. Configuration files are YAML formatted and normally live in `/etc/pantry`. To use these config files, specify the `--config` option when starting `pantry-client` and `pantry-server`.

## Default Configs

The default configuration files are in the main [pantry/pantry](https://github.com/pantry/pantry) repository and include all available configuration options and their defaults.

[Default server.yml](https://github.com/pantry/pantry/blob/master/dist/server.yml)

[Default client.yml](https://github.com/pantry/pantry/blob/master/dist/client.yml)

## data_dir and Pantry.root

Both the Client and the Server store all of their information in the configured `data_dir`. This directory is available in the code as `Pantry.root`. In the situation where a Pantry Server and a Pantry Client are running on the same server, `data_dir` should be not be set the same. A good pattern is to set it one more directory deep, such as `/var/lib/pantry/server` and `/var/lib/pantry/client`. Otherwise you risk the Client and Server clobbering each other's files.

## Run Scripts

Pantry also provides [Upstart](http://upstart.ubuntu.com/) run scripts for both the Client and the Server. These scripts assume configuration file locations of `/etc/pantry/client.yml` and `/etc/pantry/server.yml`.

[Upstart Pantry Server](https://github.com/pantry/pantry/blob/master/dist/upstart/pantry-server.conf)

[Upstart Pantry Client](https://github.com/pantry/pantry/blob/master/dist/upstart/pantry-client.conf)
