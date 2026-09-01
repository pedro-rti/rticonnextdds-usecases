# `connextsrc` Post-DDS Queue Behavior

## Data flow

A complete video path contains two independent buffering layers:

```text
GStreamer producer
  -> connextsink
  -> DDS DataWriter
  -> DDS DataReader history
  -> connextsrc on_data_available() / take()
  -> connextsrc internal GAsyncQueue
  -> GStreamer consumer
```

DDS reliability and history control delivery only until `connextsrc` calls
`take()`. After a sample is taken, its frame is copied into a `GstBuffer` owned
by the plugin's internal queue. DDS no longer controls that buffer.

Consequently, DDS `KEEP_LAST(1)` does not bound the plugin queue. It can keep the
DataReader history small, but a slower GStreamer consumer can still accumulate
buffers after the listener has taken samples. Use `queue-mode` to select the
post-DDS behavior explicitly; it is never inferred from DDS reliability or
history.

## Queue modes

`queue-mode=unbounded`

Preserves the original behavior. Every valid, non-empty frame is queued, with no
plugin-side limit. This avoids intentional post-DDS loss, but a slow consumer can
cause increasing latency and memory use.

`queue-mode=drop-oldest`

Limits pending buffers to `max-queued-frames`. At capacity, the oldest pending
buffer is unreferenced before the newest one is queued. This keeps live output
current at the cost of frame loss. With a capacity of `1`, only the newest valid,
non-empty sample from a multi-sample DDS take is allocated and retained.

`queue-mode=blocking`

Limits pending buffers to `max-queued-frames` and waits for the GStreamer
consumer to make room. No frame is intentionally discarded during normal
streaming. Backpressure occurs in the DDS listener after `take()`, so select DDS
history and resource limits appropriate for the publisher rate and workload.
Blocked callbacks are woken during flushing and shutdown.

`max-queued-frames` defaults to `1`, must be greater than zero, and is ignored in
`unbounded` mode.

All modes work independently with DDS `BEST_EFFORT` and `RELIABLE`. In
particular, `drop-oldest` may discard a frame even when DDS reliably delivered
it: reliable delivery ended at `take()`, and the discard occurs afterward in the
plugin queue.

## Recommended settings

| Workload | Recommended properties | Rationale |
|---|---|---|
| Live inference | `queue-mode=drop-oldest max-queued-frames=1` | Process the newest available frame and minimize latency. |
| Live display | `queue-mode=drop-oldest max-queued-frames=1` or a small capacity | Avoid stale catch-up playback. |
| Reliable recording | `queue-mode=blocking max-queued-frames=30` | Preserve frames while allowing a bounded processing burst. Size for the workload. |
| Complete playback | `queue-mode=blocking max-queued-frames=30` | Preserve ordering and frames with bounded memory. Size for the workload. |

`unbounded` remains available when compatibility with the original behavior is
more important than bounding latency or memory.

## Examples

The DDS QoS profiles below are examples and can be changed without changing the
queue policy.

Unbounded compatibility mode:

```sh
gst-launch-1.0 connextsrc domain=0 topic=Video key=cam1 queue-mode=unbounded dp-qos-profile="TransportLibrary::SHMEM" dr-qos-profile="DataFlowLibrary::Reliable" ! h264parse ! avdec_h264 ! videoconvert ! fpsdisplaysink
```

Drop stale frames for live processing:

```sh
gst-launch-1.0 connextsrc domain=0 topic=Video key=cam1 queue-mode=drop-oldest max-queued-frames=1 dp-qos-profile="TransportLibrary::SHMEM" dr-qos-profile="DataFlowLibrary::Reliable" ! h264parse ! avdec_h264 ! videoconvert ! fpsdisplaysink
```

Block for bounded, complete playback:

```sh
gst-launch-1.0 connextsrc domain=0 topic=Video key=cam1 queue-mode=blocking max-queued-frames=30 dp-qos-profile="TransportLibrary::SHMEM" dr-qos-profile="DataFlowLibrary::Reliable" ! h264parse ! avdec_h264 ! videoconvert ! fpsdisplaysink
```

## Ownership and shutdown

Each copied frame has exactly one owner. A queued buffer is either returned to
GStreamer, unreferenced when `drop-oldest` evicts it, or unreferenced while the
queue is drained. Queue access, depth accounting, and shutdown state are
serialized. Stop and flush operations wake blocked producers and consumers,
and queued buffers are drained before finalization.

A participant created by `connextsrc` is deleted by the plugin. An externally
provided participant is never deleted; only the subscriber, reader, topic, and
content-filter entities created by this element are removed from it.

Because the IDL has no sender timestamp, `connextsrc` assigns receiver-side PTS
and DTS. It uses GStreamer clock running time when a clock is available and a
per-start monotonic running time otherwise. Sender timestamps would require a
future wire-type change.

## Diagnostics

At `GST_DEBUG` level, `connextsrc` reports a queue summary every 300 valid DDS
samples and when streaming stops. Enable it with, for example:

```sh
GST_DEBUG=connextsrc:5 gst-launch-1.0 connextsrc domain=0 topic=Video key=cam1 queue-mode=drop-oldest max-queued-frames=1 ! h264parse ! avdec_h264 ! fakesink
```

The summary fields are:

- `DDS samples`: valid samples received by the listener.
- `delivered`: buffers returned to GStreamer.
- `dropped`: pending buffers evicted by `drop-oldest`.
- `depth`: buffers currently pending.
- `maximum depth`: highest pending depth observed since the latest start.

In `drop-oldest` mode, a rising dropped count is expected when downstream is
slower. In `blocking` mode, dropped should remain zero and maximum depth should
not exceed the configured capacity. In `unbounded` mode, sustained depth growth
indicates that downstream cannot keep up.

## Migration

Existing pipelines require no changes: the default is `queue-mode=unbounded`,
which preserves the previous post-DDS queue behavior. Pipelines that prioritize
freshness should opt into `drop-oldest`; pipelines that require every frame but
need bounded memory should opt into `blocking` and choose a capacity suitable for
the expected processing stalls. Changing DDS reliability or history does not
change the selected plugin queue mode.
