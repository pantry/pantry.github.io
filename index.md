---
layout: page
title: Pantry
github: pantry
nav_order: 0
summary: Modern DevOps Automation
---

# Pantry: Taking the Tedium out of DevOps

The DevOps movement is alive and well and moving quickly, with a slew of configuration management tools such as [Chef](http://www.getchef.com/chef/), [Puppet](http://puppetlabs.com/), [Ansible](http://www.ansible.com/home), and [Salt Stack](http://www.saltstack.com/) along with provisioning tools like [Packer](http://www.packer.io/) and [Docker](https://www.docker.io/) to make server infrastructures ever easier to manage and maintain.

However, we're still missing an important piece of the puzzle: the pipeline. In most teams, "DevOps" is still a dedicated team building and maintaining the provisioning and deploy pipeline, often built around a single tool. This requires significant custom development work to answer several important questions, including:

* Where do I store my {cookbooks,manifests,packer configs,...}?
* How do I provision new servers?
* How do I configure these new servers?
* How do I use these tools together?
* How can I share configurations across applications?
* How do I keep all of this secure?

Pantry answers these questions for you. Use your provisioning and configuration tool of choice and let Pantry do the rest.

## So, What is it?

Technically, Pantry is a Client/Server asynchronous remote execution framework.

Non-technically, Pantry gives you a structure for setting up your DevOps pipeline without imposing itself on the process. Want to use Chef? Puppet? Packer and Chef? Pantry's plugin architecture and agnostic core make all of this possible!

## Is it Secure?

As server provisioning often includes sensitive information such as passwords and account tokens, security is of the utmost concern in Pantry. To protect Client and Server communication, Pantry employs [ZeroMQ's](http://zeromq.org/) [Curve encryption](http://curvezmq.org/) protocol. For more information please visit [Security](/core/security.html).


## How Do I Use It?

Go check out [Getting Started](/getting_started.html)!

## What's Available Now?

[Pantry Core](/core) :: Pantry's core messaging and execution system.

[Pantry Chef](/chef) :: Turn Pantry into a simpler Chef Server.

## How do I Write My Own Plugins?

[Extending Pantry](/core/extending.html) has all the information you'll need.

## What Does "Pantry" Mean?

At [Collective Idea](http://collectiveidea.com) we use Chef heavily to manage multiple Customer websites as well as a few of our own. We eventually found ourselves managing multiple Chef repositories, all with copied cookbooks, many of them slightly different from each other. After failing to find an existing solution (Chef Server does not have a good solution) we began building our own solution. After building the core of the Client/Server communication we realized we could use this for tools other than Chef, such as Packer. The name "Pantry" comes from Chef's "kitchen" metaphor. Where a pantry holds the food and ingredients for baking, Pantry holds your ingredients for server configuration and provisioning.

## Contributing

Pantry is Open Source and available on [Github](https://github.com/pantry).
