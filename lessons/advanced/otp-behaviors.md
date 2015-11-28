---
layout: page
title: OTP Behaviors
category: advanced
order: 3
---

We've looked at the Elixir abstractions for concurrency but sometimes we need greater control and for that we turn to the OTP behaviors that Elixir built on.

In this lesson we'll focus on three important pieces: GenServers, GenEvents, and Supervisors.

## Table of Contents

- [GenServer](#genserver)
  - [Synchronous Functions](#synchronous_function)
  - [Asynchronous Functions](#asynchronous_function)
- [GenEvent](#genevent)
- [Supervisors](#supervisors)
  - [Strategies](#strategies)

## GenServer

An OTP server is a module with the GenServer behavior that implements a set of callbacks.  At it's most basic level a GenServer is a loop that handles one request per iteration, passing along an updated state.

To demonstrate the GenServer API we'll implement a basic queue to store and retrieve data or work.

To begin our GenServer we need to start it and handle initialization.  In our case we want to link processes so we'll use `GenServer.start_link/3` which takes three parameters: the GenServer we're starting, arguments, and a set of options.  The arguments are be passed to `GenServer.init/1` which sets the initial state through it's return value.  In our example we'll use the arguments to set our initial state:

```elixir
defmodule SimpleQueue do
  use GenServer

  @doc """
  Start our queue and link it.  This is a helper method
  """
  def start_link(state \\ []) do
    GenServer.start_link(__MODULE__, state, name: __MODULE__)
  end

  @doc """
  GenServer.init/1 callback
  """
  def init(state), do: {:ok, state}
end
```

### Synchronous Functions

It's often necessary to interact with GenServers in a synchronous way, calling a function and waiting for it's response.  To handle these request we need to implement the `GenServer.handle_call/3` callback, the three parameters are: a request, the caller's PID, and existing state.  Through the use of pattern matching we can handle many different requests and states.  The expected return value for our callback is a tuple: `{:reply, response, state}`.  A complete list of accepted return values can be found in the [`GenServer.handle_call/3`](http://elixir-lang.org/docs/v1.1/elixir/GenServer.html#c:handle_call/3) docs.

To demonstrate synchronous requests let's add dequeue functionality to our queue:

```elixir
defmodule SimpleQueue do
  use GenServer

  ### GenServer API

  @doc """
  GenServer.init/1 callback
  """
  def init(state), do: {:ok, state}

  @doc """
  GenServer.handle_call/3 callback
  """
  def handle_call(:dequeue, _from, [value|state]) do
    {:reply, value, state}
  end
  def handle_call(:dequeue, _from, []), do: {:reply, nil, []}

  ### Client API / Helper methods

  def start_link(state \\ []) do
    GenServer.start_link(__MODULE__, state, name: __MODULE__)
  end

  def dequeue, do: GenServer.call(__MODULE__, :dequeue)
end

```

Let's start our SimpleQueue and test out our new dequeue functionality:

```elixir
iex> SimpleQueue.start_link([1, 2, 3])
{:ok, #PID<0.90.0>}
iex> SimpleQueue.dequeue
1
iex> SimpleQueue.dequeue
2
```

### Asynchronous Functions

Asynchronous requests are handled with the `handle_cast/2` callback.  This works much like `handle_cast/3` but does not receive the caller and is not expected to reply.

We'll implement our enqueue functionality to be asynchronous, updating the queue but not blocking our current execution:

```elixir
defmodule SimpleQueue do
  use GenServer

  ### GenServer API

  @doc """
  GenServer.init/1 callback
  """
  def init(state), do: {:ok, state}

  @doc """
  GenServer.handle_call/3 callback
  """
  def handle_call(:dequeue, _from, [value|state]) do
    {:reply, value, state}
  end
  def handle_call(:dequeue, _from, []), do: {:reply, nil, []}

  def handle_call(:queue, _from, state), do: {:reply, state, state}

  @doc """
  GenServer.handle_cast/2 callback
  """
  def handle_cast({:enqueue, value}, state) do
    {:noreply, state ++ [value]}
  end

  ### Client API / Helper methods

  def start_link(state \\ []) do
    GenServer.start_link(__MODULE__, state, name: __MODULE__)
  end
  def queue, do: GenServer.call(__MODULE__, :queue)
  def enqueue(value), do: GenServer.cast(__MODULE__, {:enqueue, value})
  def dequeue, do: GenServer.call(__MODULE__, :dequeue)
end
```

Let's put our new functionality to use:

```elixir
iex> SimpleQueue.start_link([1, 2, 3])
{:ok, #PID<0.100.0>}
iex> SimpleQueue.queue
[1, 2, 3]
iex> SimpleQueue.enqueue(20)
:ok
iex> SimpleQueue.queue
[1, 2, 3, 20]
```

For more information check out the offical [GenServer](http://elixir-lang.org/docs/v1.1/elixir/GenServer.html#content) documentation.

## GenEvent

We learned that GenServers are processes that can maintain state and request to synchronous and asynchronous requests so what is a GenEvent?  GenEvents are generic event managers that receive incoming events and notify subscribed consumers.

The most important callback in GenEvents as you can imagine is `handle_event/2`.  This receives the event and the handler's current state and is expected to return a tuple: `{:ok, state}`.  In addition to `handle_event/2`, GenEvents also support `handle_call/2` among other callbacs.

To demonstrate the GenEvent functionality let's start by creating two handlers, one to keep a log of messages and the other to persist them:

```elixir
defmodule LoggerHandler do
  use GenEvent

  def handle_event({:msg, msg}, messages) do
    IO.puts "Logging new message: #{msg}"
    {:ok, [msg|messages]}
  end

  def handle_call(:messages, messages) do
    {:ok, Enum.reverse(messages), messages}
  end
end

defmodule PersistenceHandler do
  use GenEvent

  def handle_event({:msg, msg}, state) do
    IO.puts "Persisting log message: #{msg}"

    # Save message

    {:ok, state}
  end
end
```

Once we're ready to use our handlers we need to familiarize ourselves with a few of GenEvent's functions.  The most commonly used functions are `add_handler/3`, `notify/2`, and `call/4`, these allow us to add new handlers, notify handlers of new events, and trigger specific handler functions.

Let's see our handlers in action:

```elixir
iex> {:ok, pid} = GenEvent.start_link([])
iex> GenEvent.add_handler(pid, LoggerHandler, [])
iex> GenEvent.add_handler(pid, PersistenceHandler, [])

iex> GenEvent.notify(pid, {:msg, "Hello World"})
Logging new message: Hello World
Persisting log message: Hello World

iex> GenEvent.call(pid, LoggerHandler, :messages)
["Hello World"]
```

See the offical [GenServer](http://elixir-lang.org/docs/v1.1/elixir/GenEvent.html#content) documentation for a complete list of callbacks and GenEvent functionality.

## Supervisors

Supervisors are specialized processes with one purpose: monitoring other processes. These supervisors enable us to create fault-tolerant applications by automatically restarting child processes when they fail.

The magic to Supervisors is in the `Supervisor.start_link/2` function.  In addition to starting our supervisor and children, it allows us to define the strategy our supervisor will use for managing its child processes.

Let's use our GenServer from the beginning of the lesson:

```elixir
import Supervisor.Spec

children = [
  worker(SimpleQueue, [[], [name: SimpleQueue]])
]

{:ok, pid} = Supervisor.start_link(children, strategy: :one_for_one)
```

If our process were to crash or be terminated our Supervisor would automatically restart it as if nothing had happened.

### Strategies

There are currently four different restart strategies available to supervisors:

+ `:one_for_one` - Only restart the failed child process.

+ `:one_for_all` - Restart all child processes in the event of a failure.

+ `:rest_for_one` - Restart the failed process and any process started after it.

+ `:simple_one_for_one` - Best for dynamically attached children. Supervisor is required to contain only one child.
