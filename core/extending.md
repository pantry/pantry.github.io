---
layout: default
title: Pantry Core
subtitle: Extending Pantry
github: pantry
summary: Async Client/Server Remote Execution Framework
---

# Extending Pantry

Adding functionality to Pantry via plugins starts with Commands. Commands can be configured to run on the Server or on Clients, but not both. Commands have full access to almost the entire Pantry system and as such are very powerful tools, yet built on simple building blocks. This ensures simple Commands are easy to write, and complicated ones are possible.

## Writing a Command

All Commands must adhere to the following rules:

* A Command must be a `Pantry::Command`
* A Command must implement `#perform(message)`
* A Command may implement `#prepare_message(options)`
* A Command may implement `#receive_response(response)`
* A Command may define a `command` block for [CLI](/cli.html) support
* The Command must be registered with Pantry as a Client or Server command

The details and use of each of these API endpoints will be explained below.

A well written Command is a self-contained entity. This means that reading the code of a Command from top-to-bottom should also follow the execution flow of a Command through a Pantry network. Specifically, the Command's required sections should appear in the following order:

{% highlight ruby %}
class MyCommand < Pantry::Command
  command "mycommand" { }

  def prepare_message(options); ...; end

  def perform(message); ...; end

  def receive_response(response); ...; end
end
{% endhighlight %}

With that, lets build a simple Command that echo's the message requested by the CLI.

### Simple Server Command

We'll start out with a quick skeleton and register the Command with Pantry as a Server Command.

{% highlight ruby %}
class TheSimplestEcho < Pantry::Command
  def perform(message)
    42
  end
end

Pantry.add_server_command(TheSimplestEcho)
{% endhighlight %}

This is the simplest Command one can write for Pantry. It implements the only required method, `#perform`, and returns a static value.

All values returned by `#perform` are packaged up into a response Message and sent back to the requester. We'll see this in action soon.

Even though this Command is registered as a Server Command, it's for the most part unusable. We want to be able to trigger this Command from the CLI and pass in a message to echo, so lets do that now.

### Configure the CLI

Commands are exposed to the CLI via the `command` configuration, which takes the name of the command. This name is what users type in to invoke this Command. This method takes a block which is sent down to Ruby's [OptionParser](http://ruby-doc.org/stdlib-2.1.0/libdoc/optparse/rdoc/OptionParser.html), so all options available there are valid in this block. We'll see an example of this later. For now, lets just configure the command itself with a parameter.

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

This command is now available via the CLI as `pantry echo "The Message"`. Much like how OptionParser parses option definitions, the string given to `command` can include any parameters for documentation purposes. These extra parameters are then passed into the Command's constructor.

One important oddity to note about `Command#initialize`. Commands are also instantiated on the receiver side without any parameters, as everything a Command requires should be in the Message's metadata or body. As such, every Command initializer must be callable with and without parameters, if parameters are expected from the CLI.

There's a new method here too, `#to_message`. All Commands are responsible for creating the Pantry::Message that will trigger the current Command on the receiving side. Messages are linked to a Command via `Message#type`, which by default is the class name of the Command. In order to prevent Command name collisions, this value can be overridden via the `Command.message_type` class method.

Lastly the Command now defines a custom `#receive_response`. By default a Command will assume that it is finished as soon as a single response comes back. To act on that response, override this method. It will be given the response Message who's body contains the returned values of `perform`. This method must either call `super` or `Command#finished` to mark the Command as complete. If you forget these calls, `pantry` will hang.

#### Adding CLI Options

Lets improve our Echo command by adding an option to multiply the message when it's echoed. We want to update the CLI invocation to look like the following:

    $> pantry echo -t 5 "Echo"
    EchoEchoEchoEchoEcho
    $>

We'll need to update the `command` definition to include the "-t" option, and while we're at it will add some more description to the "echo" command itself.

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
    message.body[0] * message.body[1].to_i
  end

  def receive_response(response)
    Pantry.ui.say(message.body[0])
    super
  end
end
{% endhighlight %}

There's a bit of new code here so we'll go over the changes from the top.

A `command` block can be given a `description` string. Description strings are processed to remove excessive whitespace, so feel free to make that a multi-line, clean block of text. The first line will be the summary and the rest will only show up on the command's own help text. The `option` line is passed as-is to OptionParser.

All commands are given a default set of `-h`/`--help` options.

Next, `#to_message` has been replaced with `#prepare_message`. This method takes the options parsed from the command line (including any global options, such as `--application`, `--environment`, and `--roles`).  Use this method to verify options and to take care of any pre-flight steps before the Message goes out. By default `#prepare_message` just calls `#to_message`. This method must return the Message that will be sent over the network or raise an error. Raised errors are caught and displayed cleanly to the user that something was wrong.

One `command` option not shown here is the `group` method. This method lets plugin authors group together their plugins in a common block in the `--help` output of `pantry`.

{% highlight ruby %}
class MyPluginCommand < Pantry::Command
  command "plugin" do
    group "My Plugin"
  end
end
{% endhighlight %}

### Writing a Client Command

Converting a Server Command to a Client Command requires only one change to `#receive_response`. Due to the fully async nature of Pantry, the CLI cannot know ahead of time how many Clients will receive a given Message. The Server, through which all Messages pass, will know, and thus the first response Message will contain in its body the list of client identities who will, or should, respond. The Command will need to handle this message as well as wait for the messages from each of the mentioned Clients before finishing. The pattern at its simplest looks as follows:

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
      Pantry.ui.say("#{message.from} echos #{message.body[0]}")
    end

    if !@expected_clients.empty? &&
        @received.length >= @expected_clients.length
      super
    end
  end
end

Pantry.add_client_command(TheSimplestEcho)
{% endhighlight %}

The `echo` Command is built into Pantry and can be viewed at [Pantry::Commands::Echo](https://github.com/pantry/pantry/blob/master/lib/pantry/commands/echo.rb).

### MultiCommand

Following principles of good Object Oriented Design, good Commands should do one thing and do it well, but server administration is rarely a simple process. For tasks that require multiple tasks, instead of writing more complicated Commands Pantry exposes a special kind of Command, the `Pantry::MultiCommand`, which can call multiple Commands in order, gathering up each Command's response and passing the whole set back to the requester. Commands are ordered with the `MultiCommand.performs` method:

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

It is possible that a Command may need further information from the Server or Clients before it can continue it's job. For such situations, Commands have the `#send_request` and `#send_request!` methods. They do the same thing but `#send_request` returns a `Celluloid::Future` that needs to be queried for the response, while `#send_request!` will block until a response is ready and will return the response Message.

These methods take a Message, which should be created from the Command that needs to run on the receiving side. For example, say there was a Client Command who need to know how many Clients were connected and available.

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

### Sending and Receiving Files

There are two ways in Pantry to pass files between the Server and its Clients/CLI. The first is simple, `File.read` and put the contents of the file in the Message, and `File.write` on the receiving side. This is the recommended way for any small or plain text file. In fact this is so common that Pantry provides an abstract Command encapsulating this logic: `Pantry::Commands::UploadFile`. Subclasses then only need to specify where the file will be written out on the receiving end (e.g. [Pantry::Chef::UploadRole](https://github.com/pantry/pantry-chef/blob/master/lib/pantry/chef/upload_role.rb)).

Larger files however should use Pantry's file service. This service implements reliable, chunk-based file transfer following the [ZeroMQ Transferring Files](http://zguide.zeromq.org/page:all#Transferring-Files) guide and requires a few steps to use correctly. This service is exposed via two methods: `server_or_client#send_file` and `server_or_client#receive_file`, but first a primer on how Pantry's file service works.

The most important aspect of the file service is that the **Receiving end initiates the transfer**. Because both sides can send and receive files, and to support sending many files in parallel, the sender and receiver are linked together via two UUIDs: the receiver's UUID and the specific file transfer UUID. The Receiving end does not initially care about the name or location of the file in question, it just needs the file's size and a checksum to ensure the upload was successful and correct. The Receiver is started with `server_or_client#receive_file(file_size, checksum)` which returns an `UploadInfo` object containing `#receiver_uuid` and `#file_uuid` to send back to the Sender.

As the File Service really doesn't care where files go, it writes uploads to a temp file. To put that file into it's appropriate home once successfully uploaded, `UploadInfo` can be given an `#on_complete` block which will fire when the file has been fully received. `UploadInfo` then offers up the `#uploaded_path` method which will turn the path to the received file.

Back on the Sender's side, having received `receiver_uuid` and `file_uuid` from the Receiver, the Sender can now fire off their side of the process with `server_or_client#send_file(file, receiver_uuid, file_uuid)`. This method also returns an `UploadInfo` for one important use-case: waiting until the upload is finished. Because the File Service is a fully asynchronous process, `#send_file` immediately returns. If the Command was allowed to end at that point, the file may not get uploaded at all. To force a blocking wait on the current process until the file sending is complete, call `UploadInfo#wait_for_finish`.

Here's an example of a Command that uploads a file to the Server from the CLI.

{% highlight ruby %}
class UploadBigFile < Pantry::Command
  command "upload:big:file FILE"
  
  def initialize(file = nil)
    @file = file
  end
  
  def to_message
    super.tap do |message|
      message << File.basename(@file)
      message << File.size(@file)
      message << Digest::SHA256.file(@file).hexdigest
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
    
    upload_info = client.send_file(@file, receiver_uuid, file_uuid)
    upload_info.wait_for_finish
    super
  end
end
{% endhighlight %}

You'll notice a new invocation here, `Pantry.root`. We'll touch on that in just a sec. Also always remember that during the CLI invocation of a Command, `#initialize`, `#to_message`, `#prepare_message`, and `#receive_response` are all called on the same object on the CLI/Client side. `#perform` is executed on the Server side, so any data that `#perform` needs to know about must be sent through the Message built in `#to_message` or `#prepare_message`.

### Integrating Your Plugin

Last, but not least, any new Commands need to be loaded into Pantry proper. This is done via Rubygems. Package up your Plugin as a gem, and make sure it includes the file `pantry/init.rb` in its load path. Pantry, on boot, will look for all gems that include this file and will `require` each of them. If there's a problem loading a specific plugin, the error will be logged. For now the plugin gems need to be installed on every node (Server and Clients) that need it. There are plans on improving this flow.

### The File System

Pantry does not currently use a database of any sort; all persistent data is written out to the file system. Where Pantry writes these files is dictated by the [configuration](/core/configure.html) and is available in the code via `Pantry.root`. In order to stay self contained and good citizens of servers, no files should be written outside of the `Pantry.root` (except in the case of temp files). Furthermore, plugins should carve out their own location for their files to prevent accidental file clobbering. For example, [Pantry Chef](/chef) writes all global Chef data under `Pantry.root.join("chef")` and all Application specific data under `Pantry.root.join("application", app_name, "chef")`.