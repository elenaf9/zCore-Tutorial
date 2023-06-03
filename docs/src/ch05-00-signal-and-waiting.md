# Signaling and waiting

## Signals

Objects may have up to 32 signals (represented by the zx_signals *t type and the ZX* **SIGNAL** definition), which represent a piece of information about their current state. For example, channels and sockets may be READABLE or WRITABLE. Processes or threads may be terminated. And so on.

A thread may wait for a signal to become active on one or more objects.

## Waiting

A thread may be used to [`zx_object_wait_one()`](https://fuchsia.dev/docs/reference/syscalls/object_wait_one) wait for a signal on a single handle to become active or [`zx_object_wait_many() `](https://fuchsia.dev/docs/reference/syscalls/object_wait_many) waits for signals on multiple handles. Both calls allow timeouts, and they return even if no signals are pending.

The timeout may deviate from the specified deadline, depending on the timer margin.

If the thread has to wait for a large number of handles, it is more efficient to use a port, which is an object to which other objects may be bound, and when the signal is asserted on them, the port receives a packet containing information about the pending signal.

## Events and Event Pairs

An Event (Event) is the simplest object, with no state other than its collection of active signals.

An Event Pair is one of a pair of events that can signal each other. A useful property of an Event Pair is that when one side of the pair disappears (all its handles are closed), the PEER_CLOSED signal is asserted on the other side.

See: [`zx_event_create()`](https://fuchsia.dev/docs/reference/syscalls/event_create), and [`zx_eventpair_create()`](https://fuchsia.dev/) docs/reference/syscalls/eventpair_create).
