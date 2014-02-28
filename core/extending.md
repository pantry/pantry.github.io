---
layout: default
title: Pantry Core
subtitle: Extending
github: pantry
summary: Async Client/Server Remote Execution Framework
sub_nav: 1
---

# Extending Pantry

Adding functionality to Pantry via plugins starts with Commands. Commands can be configured to run on the Server or on Clients, but not both. Commands have full access to almost the entire Pantry system and as such are powerful tools, yet built on simple building blocks. This ensures simple Commands are easy to write, and complicated ones are possible.

## Writing a Command

All Commands must adhere to the following rules:

* A Command must be a `Pantry::Command`
* A Command must implement `#perform(message)`
* A Command may implement `#prepare_message(options)`
* A Command may implement `#receive_response(response)`
* A Command may define a `command` block for [CLI](/core/cli.html) support
* The Command must be registered with Pantry as a Client or Server command

The details and use of each of these API endpoints are explained below.

A well written Command is a self-contained entity. This means that reading the code of a Command from top-to-bottom should also follow the execution flow of a Command through a Pantry network. Specifically, the Command's various sections should appear in the following order:

{% highlight ruby %}
class MyCommand < Pantry::Command
  command "mycommand" { }

  def prepare_message(options); ...; end

  def perform(message); ...; end

  def receive_response(response); ...; end
end
{% endhighlight %}

With that, let's build a simple Command that echo's a string entered via the command line interface (CLI) back to the user from the Server.

### Simple Server Command

We'll start out with the basics and register the Command with Pantry as a Server Command.

{% highlight ruby %}
class TheSimplestEcho < Pantry::Command
  def perform(message)
    42
  end
end

Pantry.add_server_command(TheSimplestEcho)
{% endhighlight %}

This is the simplest Command possible. It implements the only required method -- `#perform` -- and returns a static value.

All values returned by `#perform` are packaged up into a response Message and sent back to the requester. We'll see this in action soon.

Even though this Command is registered as a Server Command, it is for the most part unusable. We want to be able to trigger this Command from the CLI and pass in a message to echo, so lets do that now.

### Configure the CLI

Commands are exposed to the CLI via the `command` method, which takes the name of the command and an optional block. This name is what users type to invoke the Command.

{% highlight ruby %}
class TheSimplestEcho < Pantry::Command
  command "echo MESSAGE"

  def initialize(to_echo = nil)
    @to_echo = to_echo
  end

  def to_message
    super.tap do |msg|
      msg << @to_echo
    end
  end

  def perform(message)
    message.body[0]
  end

  def receive_response(response)
    Pantry.ui.say(message.body[0])
    super
  end
end
{% endhighlight %}

This command is now available via the CLI as `pantry echo "The Message"`. You'll notice that there's more in this string than just "echo". To facilitate documentation via the `--help` option, arguments can be specified in this string. The first word of the string is pulled out to be the invocation target. Any other parameters parsed from the command line are then passed into the Command's initializer.

One important oddity to note about `Command#initialize`. Commands are also instantiated on the receiver side without any parameters, as everything a Command requires should be in the Message's metadata or body. As such, every Command initializer must be callable without parameters, thus the `to_echo = nil`.

There's a new method here too, `#to_message`. All Commands are responsible for creating the Message that will trigger the Command on the receiving side. Messages are linked to a Command via `Message#type`, which by default is the class name of the Command. In order to prevent Command name collisions, this value can be overridden via the `Command.message_type` class method.

Lastly the Command now also defines `#receive_response`. By default a Command will assume that it is finished as soon as a single response comes back. To act on that response, override this method. `#receive_response` is called for each and every Message response as long as the process runs (normally, as a CLI receiving responses from the Server). This method must either call `super` or `Command#finished` to mark the Command as complete, otherwise the CLI will hang, waiting for more responses that will probably never come.

#### Adding CLI Options

Lets improve our Echo command by adding an option to multiply the message when echoed. We want to update the CLI invocation to look like the following:

    $> pantry echo -t 5 "Echo"
    EchoEchoEchoEchoEcho
    $>

We will update the `command` definition to include the "-t" option, and while we're at it add a description to the "echo" command itself. These options are specified in the optional block to the `command` method. This block is executed in a light wrapper round Ruby's [OptionParser](http://ruby-doc.org/stdlib-2.1.0/libdoc/optparse/rdoc/OptionParser.html) library, so all options available there are valid here. The parsed options are then passed into `#prepare_message` as a Hash, using the longest given `#option` names as keys.

{% highlight ruby %}
class TheSimplestEcho < Pantry::Command
  command "echo MESSAGE" do
    description "
      Echo a message back from the server.
      Requires a message
    "
    option "-t", "--times TIMES",
      "Duplicate the message response by TIMES. Must be positive"
  end

  def initialize(to_echo = nil)
    @to_echo = to_echo
  end

  def prepare_message(options)
    repeat = options[:times] || 1

    if repeat <= 0
      raise "Option 'times' must be a positive non-zero number"
    end

    super.tap do |msg|
      msg << @to_echo
      msg << repeat
    end
  end

  def perform(message)
    to_echo = message.body[0]
    repeat_count = message.body[1].to_i
    to_echo * repeat_count
  end

  def receive_response(response)
    Pantry.ui.say(message.body[0])
    super
  end
end
{% endhighlight %}

There's a bit of new code here so we'll go over the changes from the top.

A `command` block can be given a `description` string. Description strings are processed to remove excessive whitespace, so feel free to write multi-line, clean blocks of text. The first line will be the Command's summary and the rest will only show up on the Command's own help text (e.g. `pantry echo --help`). Pantry gives all commands their own `-h`/`--help` options.

Next, `#to_message` has been replaced with `#prepare_message`. This method takes the options parsed from the command line (including any global options, such as `--application`, `--environment`, and `--roles`).  Use this method to verify options and to run any pre-flight steps before the Message is sent. By default `#prepare_message` just calls `#to_message`. This method must return the Message to send over the network, or raise an error. Raised errors are caught and displayed cleanly to the user along with the Command's `--help`.

One `command` option not shown here is the `#group` method. This method lets plugin authors group together their plugins in a common block in the CLI's top-level `--help`.

{% highlight ruby %}
class MyPluginCommand < Pantry::Command
  command "plugin" do
    group "My Plugin"
  end
end
{% endhighlight %}

### Writing a Client Command

Converting a Server Command to a Client Command requires one important change to `#receive_response`. Due to the asynchronous nature of Pantry, the CLI cannot know ahead of time how many Clients will receive a given Message. The Server, through which all Messages pass, will know, and first responds with a Message containing all relevant Client identities before forwarding on the Message to these Clients. The Command will need to handle this first response as well as wait for the responses from each of the mentioned Clients before finishing. The pattern looks like this:

{% highlight ruby %}
class TheSimplestEcho < Pantry::Command
  # Other code is the same as previous examples ...

  def receive_response(response)
    @received ||= []
    @expected_clients ||= []

    if message.from_server?
      @expected_clients = message.body
    else
      @received << message
    end

    if !@expected_clients.empty? &&
        @received.length >= @expected_clients.length
      super
    end
  end
end

Pantry.add_client_command(TheSimplestEcho)
{% endhighlight %}

That's everything you need to know to build simple Commands in Pantry. The Command built here is also available in [Pantry Core](/core) @ [Pantry::Commands::Echo](https://github.com/pantry/pantry/blob/master/lib/pantry/commands/echo.rb).

### MultiCommand

Following principles of good Object Oriented Design, good Commands should do one thing and do it well, but many server administration tasks can require multiple steps. For such tasks Pantry exposes a special kind of Command, the `Pantry::MultiCommand`, which will invoke multiple Commands in order, gathering up each Command's `#perform` response and passing the whole set back to the requester.

{% highlight ruby %}
class DoStuff < Pantry::MultiCommand
  command "do:stuff"

  performs [
    DownloadStuff,
    InstallStuff,
    RunStuff
  ]

  def receive_response(response)
    # ... handle as needed
  end
end
{% endhighlight %}

A good example of this in action is [Pantry::Chef::Run](https://github.com/pantry/pantry-chef/blob/master/lib/pantry/chef/run.rb).

### Using Commands in Commands

It is possible that a Command may need further information from the Server or Clients before it can finish. To make extra requests inside of a Command, use the `#send_request` and `#send_request!` methods. They do the same thing, send a Message, but `#send_request` returns a `Celluloid::Future` that needs to be queried for the response, while `#send_request!` will block until a response is ready and will return the response Message.

These methods take a Message, which should be created from the Command to be run on the receiving side. For example, say there was a Client Command who needs to know how many Clients are connected and available. We can write the following:

{% highlight ruby %}
class CountClients < Pantry::Command
  def perform(message)
    4
  end
end

class ClientCommand < Pantry::Command
  def perform(message)
    server_response = send_request!(CountClients.new.to_message)
    client_count = server_response.body[0].to_i
    # ...
  end
end

Pantry.add_server_command(CountClients)
Pantry.add_client_command(ClientCommand)
{% endhighlight %}

Remember, Clients never talk directly to each other. All communication goes through the Server. In most cases, `#send_request` will be a Client command asking the Server for information.

### Sending and Receiving Files

There are two ways in Pantry to pass files between the Server and its Clients/CLI. The first is simple, `File.read` the contents of the file into a Message, and `File.write` it out on the receiving side. This is recommended for any small file (binary included) or plain text file. In fact this is so common that Pantry provides an abstract Command encapsulating this logic: `Pantry::Commands::UploadFile`. Subclasses only need to specify where the file will be written out on the receiving end (e.g. [Pantry::Chef::UploadRole](https://github.com/pantry/pantry-chef/blob/master/lib/pantry/chef/upload_role.rb)).

Larger files should use Pantry's file service. This service implements a reliable, chunk-based file transfer following the [ZeroMQ Transferring Files](http://zguide.zeromq.org/page:all#Transferring-Files) guide and requires a few steps to use correctly. This service is exposed via two methods: `server_or_client#send_file` and `server_or_client#receive_file`, but first a primer on how Pantry's file service works.

The most important aspect of the file service is that the **Receiving end initiates the transfer**. Because both sides can send and receive files, and to support sending many files in parallel, the sender and receiver are linked together via two UUIDs: the receiver's UUID and a file transfer UUID. The Receiving end does not initially care about the name or location of the file in question, it only wants the file's size and a checksum (to ensure correct transfers). The Receiver is triggered with `server_or_client#receive_file(file_size, checksum)` which returns an `UploadInfo` object containing `#receiver_uuid` and `#file_uuid`. These values should be sent back to the Sender.

Back on the Sender's side, having received `receiver_uuid` and `file_uuid` from the Receiver, the Sender can now fire off their side of the process with `server_or_client#send_file(file, receiver_uuid, file_uuid)`. This method also returns an `UploadInfo` for one important reason: waiting until the upload is finished. Because the File Service is a fully asynchronous process, `#send_file` immediately returns. If the Command was allowed to end at that point, the file may not get uploaded at all. To force a blocking wait on the current process until the file upload is complete, call `UploadInfo#wait_for_finish`.

To finish a file transfer, the Receiving end needs to do one more thing: define an `#on_complete` block. This block will be called once the Receiver has confirmed a successful file upload. The uploaded file initially lives at `UploadInfo#uploaded_path` and it's up to the `#on_complete` block to move or otherwise process the file properly.

Here's an example of a Command that uploads a file to the Server from the CLI.

{% highlight ruby %}
class UploadBigFile < Pantry::Command
  command "upload:big:file FILE"

  def initialize(file_path = nil)
    @file_path = file_path
  end

  def to_message
    super.tap do |message|
      message << File.basename(@file_path)
      message << File.size(@file_path)
      message << Pantry.file_checksum(@file_path)
    end
  end

  def perform(message)
    file_name = message[0]
    file_size = message[1]
    file_checksum = message[2]

    upload_info = server.receive_file(file_size, file_checksum)
    upload_info.on_complete do
      FileUtils.mv(
        upload_info.uploaded_path,
        Pantry.root.join(file_name)
      )
    end

    [upload_info.receiver_uuid, upload_info.file_uuid]
  end

  def receive_response(response)
    receiver_uuid = response.body[0]
    file_uuid = response.body[1]

    upload_info = client.send_file(
      @file_path, receiver_uuid, file_uuid
    )
    upload_info.wait_for_finish
    super
  end
end
{% endhighlight %}

You'll notice a new invocation here, `Pantry.root`. We'll touch on that in just a sec. Also always remember that during the CLI invocation of a Command, `#initialize`, `#to_message`, `#prepare_message`, and `#receive_response` are all called on the same object on the CLI/Client side, and `#perform` is executed on the Server side. Any data that `#perform` needs to know must be included in the Message.

### Pantry.root and the File System

Pantry does not use a database. All persistent data is written out to the file system. Where Pantry writes these files is dictated by the [configuration](/core/configure.html) and is available in the code via `Pantry.root`. In order to stay self-contained and a good citizen of servers, no files should be written outside of the `Pantry.root`. Furthermore, plugins should carve out their own location for their files to prevent accidental file clobbering. For example, [Pantry Chef](/chef) writes all global Chef data under `Pantry.root/chef` and all Application specific data under `Pantry.root/application/[app_name]/chef`.

### Integrating Your Plugin

Last, but not least, any new Commands need to be loaded with Pantry Core. This is done via Rubygems. Package up your plugin as a gem and make sure it includes a file named `pantry/init.rb`. Pantry, on boot, will look for all gems that include this file and will `require` each of them. If there's a problem loading a specific plugin, the error will be logged. For now the plugin gems need to be installed on every node (Server and Clients) that need it. There are plans on improving this flow.

## Further Reading

From here, after checking out the rest of this site, it's probably best to head to the [Rubydocs](http://rubydoc.info/github/pantry/pantry/master/frames).
