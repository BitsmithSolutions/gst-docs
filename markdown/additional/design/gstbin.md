# GstBin

`GstBin` is a container element for other `GstElements`. It makes
possible to group elements together so that they can be treated as one
single `GstElement`. A `GstBin` provides a `GstBus` for the children and
collates messages from them.

## Adding/removing elements

The basic functionality of a bin is to add and remove `GstElement`
to/from it. `gst_bin_add()` and `gst_bin_remove()` perform these
operations respectively.

The bin maintains a parent-child relationship with its elements (see
[relations](additional/design/relations.md)).

## Retrieving elements

`GstBin` provides a number of functions to retrieve one or more children
from itself. A few examples of the provided functions:

* `gst_bin_get_by_name()` retrieves an element by name.
* `gst_bin_iterate_elements()` returns an iterator to all the children.

## Element management

The most important function of the `GstBin` is to distribute all
`GstElement` operations on itself to all of its children. These
operations include:

  - state changes

  - index get/set

  - clock get/set

The state change distribution is the most complex and is explained in
[states](additional/design/states.md).

## GstBus

The `GstBin` creates a `GstBus` for its children and distributes it when
child elements are added to the bin. The bin attaches a sync handler to
receive messages from children. The bus for receiving messages from
children is distinct from the bin’s own externally-visible `GstBus`.

Messages received from children are forwarded intact onto the bin’s
external message bus, except for EOS and `SEGMENT_START`/`DONE` which are
handled specially.

`ASYNC_START`/`ASYNC_STOP` messages received from the children are used to
trigger a recalculation of the current state of the bin, as described in
[states](additional/design/states.md).

The application can retrieve the external `GstBus` and integrate it in the
mainloop or it can just `pop()` messages off in its own thread.

When a bin goes to `READY` it will clear all cached messages.

## EOS

The sink elements will post an `EOS` message on the bus when they reach
`EOS`. This message is only posted to the bus when the sink element is
in `PLAYING`.

The bin collects all `EOS` messages and forwards it to the application as
soon as all the sinks have posted an `EOS`.

The list of queued `EOS` messages is cleared when the bin goes to `PAUSED`
again. This means that all elements should repost the `EOS` message when
going to `PLAYING` again.

## SEGMENT_START/DONE

A bin collects `SEGMENT_START` messages but does not post them to the
application. It counts the number of `SEGMENT_START` messages and posts a
`SEGMENT_STOP` message to the application when an equal number of
`SEGMENT_STOP` messages were received.

The cached `SEGMENT_START`/`STOP` messages are cleared when going to `READY`.

## DURATION

When a `DURATION` query is performed on a bin, it will forward the query
to all its sink elements. The bin will calculate the total duration as
the MAX of all returned durations and will then cache the result so that
any further queries can use the cached version. The reason for caching the
result is because the duration of a stream typically does not change
that often.

A `GST_MESSAGE_DURATION_CHANGED` posted by an element will clear the
cached duration value so that the bin will query the sinks again. This
message is typically posted by elements that calculate the duration of
the stream based on some average bitrate, which might change while
playing the stream. The `DURATION_CHANGED` message is posted to the
application, which can then fetch the updated `DURATION`.

## Subclassing

Subclasses of `GstBin` are free to implement their own add/remove
functions. It is a good idea to update the `GList` of children so
that the `_iterate()` functions can still be used if the custom bin
allows access to its children.

Any bin subclass can also implement a custom message handler by
overriding the default one.
