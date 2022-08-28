## Event

Events in Melon are not supported across threads for two reasons:

1. The conventional multi-threading model is generally dominated by thread pools. Under the thread pool model, the main thread triggers events and then the lower tasks, and the child threads process tasks in synchronous mode.
2. Under the conventional multi-process model, it is generally single-threaded, such as: Nginx

Therefore, the event structure can only be used under a single thread. But **allows** individual threads to create their own event structures.

The system calls used by events vary according to different operating system platforms, and now support:

- epoll
- kqueue
- select



### Header file

```c
#include "mln_event.h"
```



### Functions



#### mln_event_init

```c
mln_event_t *mln_event_init(mln_u32_t is_main);
```

Description: Create an event structure. `is_main` expresses the meaning: whether it is the main thread. The main thread initialization is `1`, otherwise it is `0`, which is the main thread in the case of a single thread.

Return value: return event structure pointer if successful, otherwise return `NULL`



#### mln_event_destroy

```c
void mln_event_destroy(mln_event_t *ev);
```

Description: Destroy the event structure.

Return value: none



#### mln_event_dispatch

```c
void mln_event_dispatch(mln_event_t *event);
```

Description: Dispatches an event on the event set `event`. When an event is triggered, the corresponding callback function will be called for processing.

**Note**: This function will not return without calling `mln_event_set_break`.

Return value: none



#### mln_event_set_fd

```c
int mln_event_set_fd(mln_event_t *event, int fd, mln_u32_t flag, int timeout_ms, void *data, ev_fd_handler fd_handler);

typedef void (*ev_fd_handler)  (mln_event_t *, int, void *);s
```

Description: Set file descriptor event, where:

- `fd` is the file descriptor that the event is concerned with.

- `flag` is divided into the following categories:

  - `M_EV_RECV` read event
  - `M_EV_SEND` write event
  - `M_EV_ERROR` error event
  - `M_EV_ONESHOT` fires only once
  - `M_EV_NONBLOCK` non-blocking mode
  - `M_EV_BLOCK` blocking mode
  - `M_EV_APPEND` appends an event, that is, an event has been set, such as a read event, and if you want to add another type of event, such as a write event, you can use this flag
  - `M_EV_CLR` clears all events

  These flags can be set simultaneously using the OR operator.

- `timeout_ms` event timeout, in milliseconds, the value of this field is:

  - `M_EV_UNLIMITED` never times out
  - `M_EV_UNMODIFIED` retains the previous timeout setting
  - `milliseconds` timeout period

- `data` is the user data structure related to event processing, which can be defined by yourself.

- `ev_fd_handler` is an event handler function. The function has three parameters: event structure, file descriptor and user-defined user data structure.

Return value: return `0` if successful, otherwise return `-1`



#### mln_event_set_fd_timeout_handler

```c
void mln_event_set_fd_timeout_handler(mln_event_t *event, int fd, void *data, ev_fd_handler timeout_handler);
```

Description: Set the descriptor event timeout handler, where:

- `fd` is the file descriptor.
- `data` is a user data structure related to timeout processing, which can be defined by yourself.
- `timeout_handler` is the same as the callback function type of `mln_event_set_fd` function to handle timeout events.

This function needs to be used in conjunction with the `mln_event_set_fd` function, first use `mln_event_set_fd` to set the event, and then use this function to set the timeout processing function.

The reason for this is that some events may not need special functions for processing after timeout, and if they are all set in the `mln_event_set_fd` function, it will lead to too many parameters and too complicated.

Return value: none



#### mln_event_set_timer

```c
int mln_event_set_timer(mln_event_t *event, mln_u32_t msec, void *data, ev_tm_handler tm_handler);

typedef void (*ev_tm_handler)  (mln_event_t *, void *);
```

Description: Set timer event where:

- `msec` is the timer millisecond value
- `data` user-defined data structure for timed events
- `tm_handler` timing event handler function, its parameters are: event structure and user-defined data

Every time a timed event starts, it will be automatically deleted from the event set. If you need to trigger the timed event all the time, you need to call this function in the handler function to set it.

Return value: return `0` if successful, otherwise return `-1`



#### mln_event_set_signal

```c
int mln_event_set_signal(mln_event_t *event, mln_u32_t flag, int signo, void *data, ev_sig_handler sg_handler);

typedef void (*ev_sig_handler) (mln_event_t *, int, void *);
```

Description: Set the signal event handler, where:

- `flag` is divided into two values, only one of the two can be set:
   - `M_EV_SET` set signal event
   - `M_EV_UNSET` unload signal event
- `signo` is the signal value.
- `data` is a custom user data structure.
- `ev_sig_handler` is a signal processing function. The parameters of this function are: event structure, signal value, custom user data

Since a signal allows multiple handlers to be set, which handler is unloaded when the signal is unloaded is matched by the `data` and `ev_sig_handler` pointer values.

Return value: return `0` if successful, otherwise return `-1`



#### mln_event_set_break

```c
void mln_event_set_break(mln_event_t *ev);
```

Description: Interrupt event processing so that the `mln_event_dispatch` function returns.

Return value: none



#### mln_event_set_callback

```c
void (mln_event_t *ev, dispatch_callback dc, void *dc_data);

typedef void (*dispatch_callback) (mln_event_t *, void *);
```

Description: Set the event processing callback function, which will be called once at the beginning of each time loop. Currently mainly used to handle configuration hot reloading.

`dc_data` is a user-defined data structure.

`dc` is a callback function, and its parameters are: event structure and user-defined data structure.

Return value: none



### Example

```c
#include <stdio.h>
#include <stdlib.h>
#include "mln_core.h"
#include "mln_log.h"
#include "mln_event.h"

static void timer_handler(mln_event_t *ev, void *data)
{
    mln_log(debug, "timer\n");
    mln_event_set_timer(ev, 1000, NULL, timer_handler);
}

static void mln_fd_write(mln_event_t *ev, int fd, void *data)
{
    mln_log(debug, "write handler\n");
    write(fd, "hello\n", 6);
    mln_event_set_fd(ev, fd, M_EV_CLR, M_EV_UNLIMITED, NULL, NULL);
}

int main(int argc, char *argv[])
{
    mln_event_t *ev;
    struct mln_core_attr cattr;

    cattr.argc = argc;
    cattr.argv = argv;
    cattr.global_init = NULL;
    cattr.master_process = NULL;
    cattr.worker_process = NULL;
    if (mln_core_init(&cattr) < 0) {
        fprintf(stderr, "init failed\n");
        return -1;
    }

    ev = mln_event_init(1);
    if (ev == NULL) {
        mln_log(error, "event init failed.\n");
        return -1;
    }

    if (mln_event_set_timer(ev, 1000, NULL, timer_handler) < 0) {
        mln_log(error, "timer set failed.\n");
        return -1;
    }

    if (mln_event_set_fd(ev, STDOUT_FILENO, M_EV_SEND, M_EV_UNLIMITED, NULL, mln_fd_write) < 0) {
        mln_log(error, "fd handler set failed.\n");
        return -1;
    }

    mln_event_dispatch(ev);

    return 0;
}
```
