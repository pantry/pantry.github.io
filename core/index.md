---
layout: page
title: Pantry Core
github: pantry/pantry
nav_order: 1
summary: Async Client/Server Remote Execution Framework
---

# Pantry Core

Pantry is designed and built around two fantastic libraries: [Celluloid](http://celluloid.io/) for asynchronous processing and [ZeroMQ](http://zeromq.org/) for messaging and network communication.

## Pantry Server

The Pantry Server is the communication hub and central brain of a Pantry network. Pantry Server handles Client authentication, file and information storage, and routing of Messages to their intended destination. The Pantry Server stores everything it needs to know in its `Pantry.root` (see [Configuration](/core/configure.html)), making it trivial to add and remove Pantry from a system, as well as keeping backups.

Pantry Clients, including CLI users, will always connect to and communicate through a Pantry Server.

## Pantry Client

The Pantry Client always connects to a single Pantry Server, through which it sends and receives Messages and executes Commands. Pantry Clients can be [configured](/core/configure.html#client) to be a member of an `application` in a specific `environment` and managing any number of `roles`. Through these values the Pantry Server is able to target specific Clients for Messages. More on this below.

Like the Pantry Server, Pantry Clients also store all of their information under `Pantry.root`.

## Pantry CLI

Most users will interact with a Pantry network via the CLI. You can read more about this at [Using the Pantry CLI](/core/cli.html).

## Aspects of Pantry

The two core tennants of Pantry are the Command and the Message. Everything about Pantry is built around sending Messages between Clients and their Server in order to execute the requested Command.

### Message

The design of the Message is heavily influenced by ZeroMQ's messaging structure where a single message can contain many parts, and the order of these parts are guaranteed across the network. Each Message has three main segments: the stream, the metadata, and the body.

#### Stream

A Message's stream, or destination, contains the identity or PUB/SUB stream of who should receive this Message. See PUB/SUB below for a more detailed description of how this works.

#### Metadata

A Message's metadata is a Hash containing information that for one reason or another does not belong in the Message's body. Internally Pantry uses the metadata to communicate some state and flow control between Server and Clients, but this is not an internal-only feature. Any Command can also add whatever it wants to this metadata. See [Pantry::Chef::UploadCookbook](https://github.com/pantry/pantry-chef/blob/master/lib/pantry/chef/upload_cookbook.rb) for an example.

#### Body

The body of a Message is an array. Each element of the array is sent down the network as a ZeroMQ message part, guarenteed to be recombined at the destination in the same order. Individual elements of the body can be themselves Arrays or Hashes. These elements will be serialized over the wire via JSON. Any other object will be turned into a String.

### Command

The Command is the workhorse of Pantry. All logic to be executed on the Client or Server must be implemented in a Command. Commands, as said above, are triggered via Messages over the network, and can be triggered by the Server, a Client, or the [CLI](/core/cli.html).

Commands are also the extension point where all Pantry plugins begin, so to further understand these objects it's best to read [Extending Pantry](/core/extending.html).

## Networking Topology

Pantry uses ZeroMQ for all network communication. Communication between the Server and its Clients happens via three ZMQ pairings: PUB/SUB for sending Messages to Clients, DEALER/ROUTER so Clients can talk back to Servers, and ROUTER/ROUTER so Clients and Servers can pass files between each other ([Read about ZeroMQ Pairings](http://zguide.zeromq.org/page:all#toc62)). The ports used for each of these pairings are also [configurable](/core/configure.html).

![Pantry Network Topology](/assets/images/network_topology.png)

### Server <-> Client

Communication between Server and Client is a two way street but with different requirements depending on the direction. Servers must be able to talk to any number of Clients at once, but a Client will only ever talk directly to its Server.

#### PUB/SUB

The PUB/SUB pair is how a Server communicates to Client(s). ZeroMQ [PUB/SUB](http://zguide.zeromq.org/page:all#toc49) works by taking the first part of a Message (the stream) and matching that stream with the set of "subscriptions" registered by a given Client. These subscriptions are built from the configured Client options mentioned above. Thus, if a Client is under the "pantry" application, the "test" environment, and the "web" role, the Client will subscribe to the following streams:

* pantry
* pantry.test
* pantry.test.web

Thus, when a Message is meant for a subset of Clients, the Message's stream will be set to match only the Clients who've subscribed. For example, to send a Message to all Clients in the "pantry" application and "production" environment, the Message would be set to deliver to "pantry.production".

#### DEALER/ROUTER

As PUB/SUB is a one way street, and Clients only ever talk to their Server, DEALER/ROUTER is used to faciliate this communication. Using DEALER/ROUTER over REQ/REP allows Clients to send requests without expecting responses, and to send multiple requests before a response comes back.

### File Service (ROUTER/ROUTER)

For files and content that's too big to be safely sent through a single ZeroMQ Message Pantry also exposes a File Service communication socket. The design of this service is heavily influenced by ZeroMQ's [Transferring Files](http://zguide.zeromq.org/page:all#Transferring-Files) guide with the added aspect that both sides need to send and receive files equally.

See [Extending Pantry](/core/extending.html#send-receive-files) for more information on this service and how to use it in Commands.

## Security

As Security is an important and complicated topic, there's a [whole page on it](/core/security.html)!
