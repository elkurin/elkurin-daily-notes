# Wayland messages and Flush

When we call wayland protocol, it's sometimes not passed to the other side immediately.  
Why does this happen? And how can we make sure to pass the message?

## How messages are stored
Each [wl_connection](https://source.chromium.org/chromium/chromium/src/+/refs/heads/main:third_party/wayland/src/src/connection.c;l=67;drc=1c5350e7f3c012f386e12a8167b2a3e28f57774a) has 4 ring buffers.
```cpp=
struct wl_connection {
  struct wl_ring_buffer in, out;
  struct wl_ring_buffer fds_in, fds_out;
  int fd;
  int want_flush;
};
```
`out` is for a message to send, and `in` is for a message to receive.

[wl_ring_buffer](https://source.chromium.org/chromium/chromium/src/+/refs/heads/main:third_party/wayland/src/src/connection.c;l=57;drc=1c5350e7f3c012f386e12a8167b2a3e28f57774a) is, as the name says, a ring buffer that holds messages as char.

```cpp=
struct wl_ring_buffer {
  char *data;
  size_t head, tail;
  uint32_t size_bits;
  uint32_t max_size_bits;
};
```
`data` is a stream of messages. As this is a ring buffer, the meaningful data does not neccessararily start from `*data` address. Instead, it's start from `tail` and ends at `head`.  
When new message has arrived, it's queued to `head`. If it gets larger than ring buffer size, it overrides the tail.

### Write
[wl_connection_write](https://source.chromium.org/chromium/chromium/src/+/refs/heads/main:third_party/wayland/src/src/connection.c;l=524;drc=1c5350e7f3c012f386e12a8167b2a3e28f57774a) is called when the new message is added.  
The message is held as `void *data`.  
This `data` is not immediately read by the other side, but instead it's queued by [wl_connection_queue](https://source.chromium.org/chromium/chromium/src/+/refs/heads/main:third_party/wayland/src/src/connection.c;l=536;drc=1c5350e7f3c012f386e12a8167b2a3e28f57774a).  
The queued message is put by [ring_buffer_put](https://source.chromium.org/chromium/chromium/src/+/refs/heads/main:third_party/wayland/src/src/connection.c;l=86;drc=1c5350e7f3c012f386e12a8167b2a3e28f57774a).  
The new `data` to be added is memcpy-ed to the head of the ring buffer like `memcpy(b->data + head, data, count)`. If the remaining size of the ring buffer is not enough to copy `data`, it fills in the remaining area and goes to the tail.
```cpp=
size = ring_buffer_space(b) - head;
memcpy(b->data + head, data, size);
memcpy(b->data, (const char *) data + size, count - size);
```

After this, `count`, ths size of the message, is added to `b->head` and completes the wl_connection_write flow.  
As you can see, the message is just added to queue, and no one dispatches the sending event for now.


[wl_connection_demarshal](https://source.chromium.org/chromium/chromium/src/+/refs/heads/main:third_party/wayland/src/src/connection.c;l=833;drc=1c5350e7f3c012f386e12a8167b2a3e28f57774a) creates a `closure` which will be added to wl_event_queue..

The first message in `data` in ring buffer is [wl_connection_copy](https://source.chromium.org/chromium/chromium/src/+/refs/heads/main:third_party/wayland/src/src/connection.c;l=350;drc=1c5350e7f3c012f386e12a8167b2a3e28f57774a)-ed.  
Its inner implementation is like `memcpy(data, b->data + tail, count)`, so it's copying the tail data.  
Then the message is parsed with [`arg.type`](https://source.chromium.org/chromium/chromium/src/+/refs/heads/main:third_party/wayland/src/src/connection.c;l=884;drc=1c5350e7f3c012f386e12a8167b2a3e28f57774a).  
After this, [wl_connection_consume](https://source.chromium.org/chromium/chromium/src/+/refs/heads/main:third_party/wayland/src/src/connection.c;l=356;drc=1c5350e7f3c012f386e12a8167b2a3e28f57774a) adds `tail` by the size of the message consumed, meaning the area is no longer meaningful and ignored from the ring buffer.  

Then such event queue is dispatched from [dispatch_event](https://source.chromium.org/chromium/chromium/src/+/refs/heads/main:third_party/wayland/src/src/wayland-client.c;l=1562;drc=1c5350e7f3c012f386e12a8167b2a3e28f57774a) for client and [wl_client_connection_data](https://source.chromium.org/chromium/chromium/src/+/refs/heads/main:third_party/wayland/src/src/wayland-server.c;l=330;drc=1c5350e7f3c012f386e12a8167b2a3e28f57774a) from server.


## flush
So, when the queued message is sent?

From client, [queue_event](https://source.chromium.org/chromium/chromium/src/+/refs/heads/main:third_party/wayland/src/src/wayland-client.c;l=1467;drc=1c5350e7f3c012f386e12a8167b2a3e28f57774a) seems to always trigger wl_connection_demarshal if there is no error.

[wl_connection_flush](https://source.chromium.org/chromium/chromium/src/+/refs/heads/main:third_party/wayland/src/src/connection.c;l=417;drc=1c5350e7f3c012f386e12a8167b2a3e28f57774a) is its inner implementation.  
[wl_client_flush](https://source.chromium.org/chromium/chromium/src/+/main:third_party/wayland/src/src/wayland-server.c;l=464;drc=55d044810ca32ae24499d2c6aee6084d7e31d576) from server side, and [wl_display_flush](https://source.chromium.org/chromium/chromium/src/+/main:third_party/wayland/src/src/wayland-client.c;l=1974;drc=55d044810ca32ae24499d2c6aee6084d7e31d576) from client side. [wl_client_connection_data](https://source.chromium.org/chromium/chromium/src/+/refs/heads/main:third_party/wayland/src/src/wayland-server.c;l=330;drc=1c5350e7f3c012f386e12a8167b2a3e28f57774a) may also flush.

If the message ring buffer becomes big enough, larger than [`WL_BUFFER_FLUSH_SIZE`](https://source.chromium.org/chromium/chromium/src/+/refs/heads/main:third_party/wayland/src/src/wayland-private.h;l=54;drc=1c5350e7f3c012f386e12a8167b2a3e28f57774a), it immediately flushes the current buffered messages. [here](https://source.chromium.org/chromium/chromium/src/+/refs/heads/main:third_party/wayland/src/src/connection.c;l=539-542;drc=1c5350e7f3c012f386e12a8167b2a3e28f57774a)  

Flush will `sendmsg` all the message we have in the ring buffer from tail to head.  
However, this is only valid when [`want_flush`](https://source.chromium.org/chromium/chromium/src/+/refs/heads/main:third_party/wayland/src/src/connection.c;l=71;drc=1c5350e7f3c012f386e12a8167b2a3e28f57774a) is true.  
This is set to 1 when a new data is queue via `wl_connection_write`, `wl_connection_queue` or `wl_connection_put_fd`.

## Note
OMG T1 JUST WON THE SERIES ALL AGAINST THE ODDS!!! !!! !!! !!! !!! !!! !!! !!! !!!  
LET'S GOOOOOOOOOOOOOOOOOOOOOOO
