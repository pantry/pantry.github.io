---
layout: default
title: Pantry Core
subtitle: CLI
github: pantry
summary: Async Client/Server Remote Execution Framework
sub_nav: 1
---

# Command Line Interface

The entry point into a Pantry network is the `pantry` command line interface (CLI). The CLI has a dynamically generated `--help` system, pulling together the CLI options of all registered Commands. All Commands also have their own `--help` output that displays how to use the Command.

The order of options given is important. All calls need to be of the form

```
pantry [options] [command] [command options] [command arguments]
```

## Global Options

To target a Command at a specific Client or a subset of Clients, the CLI provides the `--application`, `--environment`, and `--roles` global options. As explained in [Pantry Core](/core) every Client can be configured with any or all of these options.

## Per Command Options

A Command can also define its own options. See [Extending Pantry](/core/extending.html) for details on how to do this. Giving a Command the `--help` option will print out the Command's specific help page. For example, try `pantry echo --help`.

Commands have access to the parsed options as well as any global options via the `#prepare_message` method.

## Default Options

To reduce redundant typing, the CLI supports setting default options using a local `.pantry/config` file. This file, if it exists, should contain CLI options, one options per line. These options will be applied first, allowing overriding as necessary. An example might look like:

```
--application pantry
--host pantry-server.example.com
```

## Curve Security

When using `curve` security, use the `--curve-key-file` option to configure credentials for the CLI. This option requires the use of the `.pantry` directory. As described in [Security](/core/security.html), this option should be given the name of the file containing the Curve credentials for this CLI. This file then must live in `.pantry/`.
