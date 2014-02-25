---
layout: page
title: Pantry Chef
github: pantry/pantry-chef
nav_order: 2
summary: A Smaller, Simpler Chef Server
---

# Pantry Chef

Pantry Chef hosts and serves up Chef data much like a Chef Server. It handles all the major components of Chef: cookbooks, environments, roles, and data bags while Clients download all required Chef data locally and run [chef-solo](http://docs.opscode.com/chef_solo.html).

Like Chef Server, Pantry Chef keeps track of roles, environments, and data bags in a per-application setup. Unlike Chef Server though, Pantry Chef keeps all cookbooks globally, allowing easy sharing for any application needing said cookbooks. One updated cookbook can be immediately applied to all Applications without any extra hassle!

## Getting Started

Install the Ruby gem on the Server, Clients and locally:

    gem install pantry-chef

Restart the Server and Clients. The new commands will show up in the [Pantry CLI](/cli.html)'s `--help` output.

## Internal Details

### Cookbooks

All uploaded cookbooks are stored in `Pantry.root/chef/cookbooks`. Directories are validated to be "cookbooks" if they contain a `metadata.rb` file. Pantry Chef does not care about nor check cookbook versions. Pantry Chef also doesn't currently calculate cookbook dependencies, instead sending all uploaded cookbooks when requested. This will be improved upon in the future.

### Environments, Roles, Data Bags

These files are all stored in their respective Application directory in `Pantry.root/application/{application}/chef/{type}`. These files are uploaded, stored, and sent to Clients without any processing. Pantry Chef doesn't care if the files are written with the Ruby DSL or as JSON.

### Nodes

Pantry Chef does not support node definitions.

### Search

There are plans to build a pantry cookbook that includes libraries and helpers to search for other nodes via Pantry. There are no plans to support the full Chef Search syntax.
