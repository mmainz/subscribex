# Subscribex

A lightweight wrapper around [pma's Elixir AMQP library](https://github.com/pma/amqp).
Compared to the AMQP library, this package adds:

  1. Auto-start a single global connection to your RabbitMQ server on app startup
  1. Auto-reconnect to your RabbitMQ server (currently at 30 second intervals)
  1. A configurable subscriber abstraction
  1. Simplified channel creation, with the ability to automatically link or monitor the channel process.

## Installation

  1. Add `subscribex` to your list of dependencies in `mix.exs`:

    ```elixir
    def deps do
      [{:subscribex, "~> 0.4.0"}]
    end
    ```

  2. Ensure `subscribex` is started before your application:

    ```elixir
    def application do
      [applications: [:subscribex]]
    end
    ```

## Usage

### Simplest example
`Subscribex.Subscriber` is a behavior which requires several callbacks:

```elixir
  @type body          :: String.t
  @type channel       :: %AMQP.Channel{}
  @type delivery_tag  :: any
  @type ignored       :: term
  @type payload       :: term

  @callback deserialize(body) :: {:ok, payload} | {:error, term}
  @callback exchange()                       :: String.t
  @callback queue()                          :: String.t
  @callback routing_key()                    :: String.t | [String.t]
  @callback provide_channel?                 :: boolean
  @callback auto_ack?                        :: boolean
  @callback handle_payload(payload)          :: ignored
  @callback handle_payload(payload, channel) :: ignored
  @callback handle_payload(payload, channel, delivery_tag)
    :: {:ok, :ack} | {:ok, :manual}
```

Assume a queue called "my_app.publisher.activity.created" is bound to an
"events" exchange using the routing key "publisher.activity.created".

An example of a subscriber that pops items off this queue and simply logs the
payloads using Logger is as follows:

```elixir
defmodule MyApp.Subscribers.ActivityCreated do
  @behaviour Subscribex.Subscriber

  require Logger

  def start_link do
    Subscribex.Subscriber.start_link(__MODULE__)
  end

  def queue, do: "my_app.publisher.activity.created"
  def exchange, do: "events"
  def routing_key, do: "publisher.activity.created"

  def auto_ack?, do: true
  def provide_channel?, do: false
  def durable?, do: false

  def deserialize(body), do: {:ok, body} # A simple passthrough

  def handle_payload(payload), do: Logger.info(payload)

  def handle_payload(_payload, _channel) do
    raise "Unexpected use of handle_payload/2 - should route to handle_payload/1"
  end

  def handle_payload(_payload, _channel, _delivery_tag) do
    raise "Unexpected use of handle_payload/3 - should route to handle_payload/1"
  end
end
```

This module's start_link/0 function delegates to `Subscribex.Subscriber.start_link/1`.

This module is pretty verbose. A lot of (overrideable) sensible defaults can be chosen. As such, we can trim down the amount we have to do by including `use Subscribex.Subscriber` instead of manually specifying everything.

```elixir
defmodule MyApp.Subscribers.ActivityCreated do
  use Subscribex.Subscriber
  require Logger

  def start_link, do: Subscribex.Subscriber.start_link(__MODULE__)

  def queue, do: "my_app.publisher.activity.created"
  def exchange, do: "events"
  def routing_key, do: "publisher.activity.created"

  def handle_payload(payload), do: Logger.info(payload)
end
```

`auto_ack?/0` defaults to true, `provide_channel?/0` defaults to false, and we
have default implementations of `handle_payload/1-3` which raise an exception.
`durable?/0` is false by default.
We specify the queue, exchange, and routing_key, and override the default
implementation of `handle_payload/1` to simply log the payload.

### Publishing and Deserializing

Instead of just logging the payload, let's deserialize it from a JSON string,
transform the result, and publish it to a different queue.

Let's say the payload is of the format:
```json
{
  "email": "abc@gmail.com",
  "username": "abc123"
}
```

Let's say our goal is to send the user a welcome email and publish to the
"my_app.new_user.emailed" routing key a result of the format:
```json
{
  "email": "abc@gmail.com",
  "username": "abc123",
  "welcome_email_delivered": true
}
```

```elixir
defmodule MyApp.Subscribers.UserRegistered do
  use Subscribex.Subscriber
  require Logger

  def start_link, do: Subscribex.Subscriber.start_link(__MODULE__)

  def queue, do: "my_app.publisher.user.registered"
  def exchange, do: "events"
  def routing_key, do: "publisher.user.registered"

  def provide_channel?, do: true

  def deserialize(body), do: Poison.decode(body)

  def handle_payload(%{"email": email, "username": username} = payload, channel) do
    :ok = MyApp.Email.send_welcome_email(email, username)

    {:ok, publishing_payload} =
      payload
      |> Map.put_new("welcome_email_delivered", true)
      |> Poison.decode

    routing_key = "my_app.new_user.emailed"

    Subscribex.publish(channel, exchange(), routing_key, publishing_payload)
  end
end
```

This time, we overrode the `provide_channel?/0` function to return true. As a
result, a channel was passed in as the second argument to our `handle_payload/2`
function which we can use when publishing to RabbitMQ using `Subscribex.publish/4`.

We also overrode the default `deserialize/1` function, using Poison to decode
the JSON string into an elixir map.

### Optional Macros

In that last case, our module started getting pretty noisy again. We can further
simplify it using a few macros built-in to Subscribex. These macros are
automatically required and imported when you do `use Subscribex.Subscriber`, but
you can require and import manually by doing:

```elixir
defmodule MyApp.Subscribers.UserRegistered do
  require Subscribex.Subscriber.Macros
  import  Subscribex.Subscriber.Macros
end
```

However you choose to make them available, we can rewrite the `UserRegistered`
module like so:

```elixir
defmodule MyApp.Subscribers.UserRegistered do
  use Subscribex.Subscriber
  require Logger

  queue "my_app.publisher.user.registered"
  exchange "events"
  routing_key "publisher.user.registered"

  provide_channel!

  def start_link, do: Subscribex.Subscriber.start_link(__MODULE__)

  def deserialize(body), do: Poison.decode(body)

  def handle_payload(%{"email": email, "username": username} = payload, channel) do
    :ok = MyApp.Email.send_welcome_email(email, username)

    {:ok, publishing_payload} =
      payload
      |> Map.put_new("welcome_email_delivered", true)
      |> Poison.decode

    routing_key = "my_app.new_user.emailed"

    Subscribex.publish(channel, exchange(), routing_key, publishing_payload)
  end
end
```

These macros create a clear separation between settings and behavior. They
compile to *exactly* the same code as above, but are clearer to read. They are
completely optional, so do whatever is most comfortable for you and your team.

### Manual Acking

Let's take the email example and modify it a bit. It still has the same
requirements, but the email sending is handled by another process, and we don't
want to block the subscriber while it's working.

```elixir
defmodule MyApp.Subscribers.UserRegistered do
  use Subscribex.Subscriber
  require Logger

  queue "my_app.publisher.user.registered"
  exchange "events"
  routing_key "publisher.user.registered"

  manual_ack!

  def start_link, do: Subscribex.Subscriber.start_link(__MODULE__)

  def deserialize(body), do: Poison.decode(body)

  def handle_payload(%{"email": email, "username": username}, _channel, delivery_tag) do
    :ok = MyApp.Email.send_welcome_email(email, username, delivery_tag)
    {:ok, :manual}
  end
end
```

This time, we've used the `manual_ack!` flag, which is equivalent to having:
```elixir
defmodule MyApp.Subscribers.UserRegistered do
  def auto_ack?, do: false
end
```

Because we've told the subscriber we intend to manually ack, we need the
delivery_tag for the payload, which is provided by RabbitMQ. Now the process in
charge of sending the email is responsible for acking the message to RabbitMQ
when it's done processing the message.

```elixir
defmodule MyApp.Email do
  # Code for delivering the email here

  def handle_email_sent(delivery_tag) do
    channel = Subscribex.channel(:no_link) # creates a new channel
    Subscribex.publish(channel, ...other args)
    Subscribex.ack(channel, delivery_tag)
    Subscribex.close(channel)
  end
end
```

In this case, we only passed the delivery tag to the mailer, not the channel we
were provided. As such, we need a way to get a channel from anywhere. This is
the purpose of the `Subscribex.channel/1` function, which is spec-ed as:

```elixir
  @spec channel(:link | :no_link | :monitor | function | (channel -> term))
  :: %AMQP.Channel{} | {%AMQP.Channel{}, monitor} | :ok
```

It expects you to pass in :link, :no_link, or :monitor. There is no default
value - you must explicitly pass in the appropriate monitoring scheme for your
use case.

We can then use this channel to publish and ack. When we use
`Subscribex.channel/1`, we _must_ remember to `Subscribex.close/1` the channel
when we're done with it - otherwise, we'll experience a memory leak as channel
processes are created and never stopped. 

If the channel is needed long-term, it's best to either link or monitor it, and
handle the case where a connection failure occurs.

In this case, we only need the process for a short duration. This is why
`Subscribex.channel/1` also accepts a function which takes the channel as an
argument. By doing this, we can change the above function to:

```elixir
defmodule MyApp.Email do
  # Code for delivering the email here

  def handle_email_sent(delivery_tag) do
    Subscribex.channel(fn(channel) ->
      Subscribex.publish(channel, ...other args)
      Subscribex.ack(channel, delivery_tag)
    end)
  end
end
```

Now the process of channel creation and closing are managed for us, and we only
have to code what we need to do with the channel.
