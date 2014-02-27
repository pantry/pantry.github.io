---
layout: default
title: Pantry Core
subtitle: Security
github: pantry
summary: Async Client/Server Remote Execution Framework
sub_nav: 1
---

# Security

Pantry has two security modes: `null` and `curve`. The type of security used is set via the [networking/security setting](/core/configure.html).

## Null Security

Null Security, as can probably be guessed, is no security at all. There's no encryption and no client authentication. If you are uncomfortable with using `curve` security, we recommend setting up encrypted pipes, such as `stunnel`, to protect Pantry communications. See [configuration](/core/configure.html) for the ports Pantry uses.

Null Security is the default for reasons described below.

## ZeroMQ's Curve Security

One of the main reasons ZeroMQ was chosen as the communication layer for Pantry is the inclusion of Curve cryptography. ZeroMQ 4.x now has an implementation of [CurveCP](http://curvecp.org/) -- an encrypted communication replacement for TCP using the [Curve25519](http://cr.yp.to/ecdh.html) encryption algorithm. Instead of replacing TCP, though, [CurveZMQ](http://curvezmq.org/) (as it's formally known) adds Curve encryption to the already solid ZeroMQ messaging system, requiring minimal setup.

Why Curve cryptography over the tried-and-true SSL? Curve cryptography provides the following:

* Server and Client authentication
* Stops replay attacks
* Stops Man in the Middle attacks
* Perfect Forward Secrecy
* No need to worry about old and/or broken protocols

Why is this not the default for Pantry? CurveCP and specifically CurveZMQ are new to the cryptography scene. The protocols, and specifically ZeroMQ's implementation, have not been around long enough to be considered vetted by the crypto community. Also, with CurveZMQ we expect many potentially large changes as the protocol is used and matures.

As such, while we believe that Curve encryption is incredibly strong -- so far no attacks have been found against it -- it must be said that **Curve encryption is unproven, use at your own risk**.

### Using Curve

To use CurveZMQ in Pantry, you first need a build of ZeroMQ 4.x that is linked against [libsodium](https://github.com/jedisct1/libsodium). Pantry's Server and Client will both fail to start if security is set to `curve` but the local ZeroMQ library is missing support. Chances are this will require building ZeroMQ from source (configure with `--with-libsodium`).

Curve works on 32-bit keys, where each end has it's own public and private key. For readability ZeroMQ can encode these keys into [40 character z85 strings](http://rfc.zeromq.org/spec:32). Pantry uses the encoded version of these keys for creation and storage. There are two aspects to Pantry's Curve Security: Encryption and Authorization. Encryption we can mostly ignore, but Authentication needs some explanation.

#### Authentication

By default, the Pantry Server will only allow in a Client if that Client knows the Server's public key **and** if the Server already knows the Client's public key. The Client and Server store their respective keypairs in `Pantry.root/security/curve/client_keys.yml` and `Pantry.root/security/curve/server_keys.yml` respectively. New Client keys are created by the Server using the `pantry client:create` CLI command, which returns the keys in YAML format that's easy to drop into place for the new Client.

When creating a new Client keypair meant for a CLI client, `pantry` supports the `--curve-key-file FILE` option, which will look for the named `FILE` in the local `.pantry` directory and when found configures the CLI as a `curve` Client.

#### Getting Started

There is a problem though. With such strict authentication and requiring a Curve connection to the Server to create new Client keys, how do we set up our the first Client? To help with initial setup, when the Server doesn't know about any Clients it relaxes its authentication requirements. The first Client to connect into the Server knowing the Server's public key will be allowed in and marked as a known Client.

To make finding the new Server's public key easy, the Server logs its public key on startup. In most cases, a CLI Client will be the first to connect to a new Server. Once you have the Server's public key, set up a local file named `.pantry/keys.yml` with the content:

{% highlight yaml %}
---
server_public_key: the-servers-public-key
{% endhighlight %}

and run the with the option `--curve-key-file keys.yml`. This should connect and running a simple `status` or `echo` request should be sufficient to fill out the rest of the Client's keys, allowing any future requests from this CLI.

#### Debugging Issues with Curve

There is currently little feedback when Curve connections fail, but there is work on ZeroMQ to improve this. In the mean time, if a Client is failing to connect to a Server via Curve, set the Server's [log level](/core/configure.html) to debug. If the Client is not known by the Server, debug logs will mention as such. If the Client has the wrong Server public key, nothing will show up in the logs at all (as the initial encrypted request is unreadable by the Server).
