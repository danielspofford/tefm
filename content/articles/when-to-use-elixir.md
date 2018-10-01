# When to use Elixir

Elixir is a powerful and productive language that produces testable, scalable,
and robust systems.

## Language strengths

- Functional language with modern syntax

  - Better local reasoning of code

  - Concise and composable functions that aid code readability and DRYness

  - Enhanced testability

  - Fast and confident refactoring

- Process based language

  - Powerful native statements to spawn processes, monitor them, pass messages
  between them, collect their results, and destroy them

- Supervisors, isolated processes, and immutability

  - Makes it trivial to write fault tolerant code that limits the impact of
   degredation and recovers automatically

  - Makes it trivial to write systems in such a way that they can partially
  degrade rather than taking the entire system down

  - Makes working with asynchronous code simpler since data cannot be mutated
  unexpectedly

- Lightweight processes

  - Do not have to worry about the memory footprint of processes, millions of
  processes is ok

- Preemptive scheduler

  - Consistent low latency in the face of millions of processes whether they are
  short or long lived

  - Predictable system degredation when faced with overwhelming traffic rather
  than tipping the system over

- Built on top of the Erlang ecosystem which has been battle tested over 30+
years

## Say that in dollars and in one sentence

Elixir lets you quickly write high quality code that will scale easily, fail
minimally, and recover automatically.

## Use cases

Elixir is a great choice for any backend system that is not CPU bound by an
inherently sequential task. If on top of that the system is high traffic, has
many connections, or is otherwise IO bound, Elixir is an exceptional choice.
Between these two points are most systems, for example:

- Chat backends

- Message queues and brokers

- Servers

- APIs

- Web applications (server and client side rendering)

- Service glue and coordination

- Control planes

- Data pipelines

- Embedded firmware for IoT devices
