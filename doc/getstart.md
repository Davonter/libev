## 1. overview

libev是一个高性能的IO事件驱动框架，我们从官方提供的一个demo开始探索libev的内涵。

## 2. 源码说明

demo程序如下：

```c
// a single header file is required
#include <ev.h>

#include <stdio.h> // for puts

// every watcher type has its own typedef'd struct
// with the name ev_TYPE
ev_io stdin_watcher;
ev_timer timeout_watcher;

// all watcher callbacks have a similar signature
// this callback is called when data is readable on stdin
static void
stdin_cb(EV_P_ ev_io *w, int revents)
{
    puts("stdin ready");
    // for one-shot events, one must manually stop the watcher
    // with its corresponding stop function.
    ev_io_stop(EV_A_ w);

    // this causes all nested ev_run's to stop iterating
    ev_break(EV_A_ EVBREAK_ALL);
}

// another callback, this time for a time-out
static void
timeout_cb(EV_P_ ev_timer *w, int revents)
{
    puts("timeout");
    // this causes the innermost ev_run to stop iterating
    ev_break(EV_A_ EVBREAK_ONE);
}

int main(void)
{
    // use the default event loop unless you have special needs
    struct ev_loop *loop = EV_DEFAULT;
    printf("Hello world!\n");

    int major = ev_version_major();
    int minor = ev_version_minor();

    printf(">> major version: %d, minor version: %d\n", major, minor);

    // initialise an io watcher, then start it
    // this one will watch for stdin to become readable
    ev_io_init(&stdin_watcher, stdin_cb, /*STDIN_FILENO*/ 0, EV_READ);
    ev_io_start(loop, &stdin_watcher);

    // initialise a timer watcher, then start it
    // simple non-repeating 5.5 second timeout
    ev_timer_init(&timeout_watcher, timeout_cb, 5.5, 1.);
    ev_timer_start(loop, &timeout_watcher);

    // now wait for events to arrive
    ev_run(loop, 0);

    // break was called, so exit
    return 0;
}
```

下面，我们从main函数的第一行开始，逐步解析libev的源码结构。

### Loop

* struct ev_loop 结构体
  首先函数会定义一个struct loop的指针，这个结构体定义如下。其中包含一个时间戳和 ev_vars.h中的所有内容

```c
  struct ev_loop
  {
    ev_tstamp ev_rt_now;
    #define ev_rt_now ((loop)->ev_rt_now)
    #define VAR(name,decl) decl;
      #include "ev_vars.h"
    #undef VAR
  };
```

* EV_DEFAULT
  默认的event loop
  ```
  // 1. 宏定义，可以看出来是一个函数
  #define EV_DEFAULT  ev_default_loop (0)          /* the default loop as sole arg */
  // 2. 找到函数声明
  extern struct loop *ev_default_loop(unsigned int flag);
  // 3. 找到函数定义
  static struct ev_loop default_loop_struct
  extern struct ev_loop* ev_default_loop_ptr = 0;

  struct ev_loop * ecb_cold
  ev_default_loop (unsigned int flags) EV_THROW
  {
    if (!ev_default_loop_ptr)
      {
        struct ev_loop *loop = ev_default_loop_ptr = &default_loop_struct;


        loop_init (loop, flags);

        if (ev_backend (loop))
          {
  #if EV_CHILD_ENABLE
            ev_signal_init (&childev, childcb, SIGCHLD);
            ev_set_priority (&childev, EV_MAXPRI);
            ev_signal_start (EV_A_ &childev);
            ev_unref (EV_A); /* child watcher should not keep loop alive */
  #endif
          }
        else
          ev_default_loop_ptr = 0;
      }

    return ev_default_loop_ptr;
  }
  ```

    接下来我们看一下loop_init接口，只留下了主要擦桉树，省略了一些不重要的参数：

```c
static void noinline ecb_cold
loop_init (struct ev_loop *loop, unsigned int flags) EV_THROW
{
  if (!backend)
    {
      origflags = flags;

      ...
      invoke_cb          = ev_invoke_pending;
      ...

      if (!(flags & EVBACKEND_MASK))
        flags |= ev_recommended_backends ();


      if (!backend && (flags & EVBACKEND_EPOLL )) backend = epoll_init  (EV_A_ flags);

      ev_prepare_init (&pending_w, pendingcb);

      ev_init (&pipe_w, pipecb);
      ev_set_priority (&pipe_w, EV_MAXPRI);
    }
}
```



invoke_cb

    这个参数定义了一个函数指针ev_invoke_pending，这个指针正是事件的回调函数，定义如下：

    `typedef void (*ev_loop_callback)(EV_P);`

epoll_init

    使用epoll方式

### ev_run的工作方式
