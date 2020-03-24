# s_task - a co-routine library for C

## Features

 * "s_task" is a co-routine library written in pure C and asm (from boost library), **without** **C++** required.
 * supports various platforms, such as windows, linux, android, macos, stm32, and even stm8.
 * supports keywords **\_\_await\_\_** and **\_\_async\_\_** . :triangular_flag_on_post: For functions that may switch to other tasks, call it with 1st parameter \_\_await\_\_, for the caller function of which, define the 1st parameter as \_\_async\_\_, which make it is clear to know about context switching.
 * works with libuv for network programming.

### Special features on embedded platfrom (stm32/stm8/m051)

 * no dynamical memory allocation
 * very small memory footprint ( increased by ROM<1.5K, RAM<128 bytes + task stack size)

### [Example 1](examples/ex0_task.c) - simple task creation

```c
#include <stdio.h>
#include "s_task.h"

void* stack_main[64 * 1024];
void* stack0[64 * 1024];
void* stack1[64 * 1024];

void sub_task(__async__, void* arg) {
    int i;
    int n = (int)(size_t)arg;
    for (i = 0; i < 5; ++i) {
        printf("task %d, delay seconds = %d, i = %d\n", n, n, i);
        s_task_msleep(__await__, n * 1000);
        //s_task_yield(__await__);
    }
}

void main_task(__async__, void* arg) {
    int i;
    s_task_create(stack0, sizeof(stack0), sub_task, (void*)1);
    s_task_create(stack1, sizeof(stack1), sub_task, (void*)2);

    for (i = 0; i < 4; ++i) {
        printf("task_main arg = %p, i = %d\n", arg, i);
        s_task_yield(__await__);
    }

    s_task_join(__await__, stack0);
    s_task_join(__await__, stack1);
}

int main(int argc, char* argv) {
    __init_async__;

    s_task_init_system();
    s_task_create(stack_main, sizeof(stack_main), main_task, (void*)(size_t)argc);
    s_task_join(__await__, stack_main);
    printf("all task is over\n");
    return 0;
}
```

### [Example 2](examples/ex3_http_client.c) - asynchronized http client without callback function.

```c
void main_task(__async__, void *arg) {
    uv_loop_t* loop = (uv_loop_t*)arg;

    const char *HOST = "baidu.com";
    const unsigned short PORT = 80;

    //<1> resolve host
    struct addrinfo* addr = s_uv_getaddrinfo(__await__,
        loop,
        HOST,
        NULL,
        NULL);
    if (addr == NULL) {
        fprintf(stderr, "can not resolve host %s\n", HOST);
        goto out0;
    }

    if (addr->ai_addr->sa_family == AF_INET) {
        struct sockaddr_in* sin = (struct sockaddr_in*)(addr->ai_addr);
        sin->sin_port = htons(PORT);
    }
    else if (addr->ai_addr->sa_family == AF_INET6) {
        struct sockaddr_in6* sin = (struct sockaddr_in6*)(addr->ai_addr);
        sin->sin6_port = htons(PORT);
    }

    //<2> connect
    uv_tcp_t tcp_client;
    int ret = uv_tcp_init(loop, &tcp_client);
    if (ret != 0)
        goto out1;
    ret = s_uv_tcp_connect(__await__, &tcp_client, addr->ai_addr);
    if (ret != 0)
        goto out2;

    //<3> send request
    const char *request = "GET / HTTP/1.0\r\n\r\n";
    uv_stream_t* tcp_stream = (uv_stream_t*)&tcp_client;
    s_uv_write(__await__, tcp_stream, request, strlen(request));

    //<4> read response
    ssize_t nread;
    char buf[1024];
    while (true) {
        ret = s_uv_read(__await__, tcp_stream, buf, sizeof(buf), &nread);
        if (ret != 0) break;

        // output response to console
        fwrite(buf, 1, nread, stdout);
    }

    //<5> close connections
out2:;
    s_uv_close(__await__, (uv_handle_t*)&tcp_client);
out1:;
    uv_freeaddrinfo(addr);
out0:;
}
```

## API

### Task

```c
/* Return values -- 
 * For all functions marked by __async__ and hava an int return value, will
 *     return 0 on waiting successfully,
 *     return -1 on waiting cancalled by s_task_cancel_wait() called by other task.
 */

/* Function type for task entrance */
typedef void(*s_task_fn_t)(__async__, void *arg);

/* Create a new task */
void s_task_create(void *stack, size_t stack_size, s_task_fn_t entry, void *arg);

/* Wait a task to exit */
int s_task_join(__async__, void *stack);

/* Sleep in milliseconds */
int s_task_msleep(__async__, uint32_t msec);

/* Sleep in seconds */
int s_task_sleep(__async__, uint32_t sec);

/* Yield current task */
void s_task_yield(__async__);

/* Cancel task waiting and make it running */
void s_task_cancel_wait(void* stack);
```

### Mutex
```c
/* Initialize a mutex */
void s_mutex_init(s_mutex_t *mutex);

/* Lock the mutex */
int s_mutex_lock(__async__, s_mutex_t *mutex);

/* Unlock the mutex */
void s_mutex_unlock(s_mutex_t *mutex);
```

### Event
```c
/* Initialize a wait event */
void s_event_init(s_event_t *event);

/* Wait event */
int s_event_wait(__async__, s_event_t *event);

/* Set event */
void s_event_set(s_event_t *event);

/* Wait event with timeout */
int s_event_wait_msec(__async__, s_event_t *event, uint32_t msec);

/* Wait event with timeout */
int s_event_wait_sec(__async__, s_event_t *event, uint32_t sec);
```

## Compatibility

"s_task" can run as standalone co-routine library, or work with library libuv (compiling with macro **USE_LIBUV**).

| Platform                       | co-routine         | libuv              |
|--------------------------------|--------------------|--------------------|
| Windows                        | :heavy_check_mark: | :heavy_check_mark: |
| Linux                          | :heavy_check_mark: | :heavy_check_mark: |
| MacOS                          | :heavy_check_mark: | :heavy_check_mark: |
| Android                        | :heavy_check_mark: | :heavy_check_mark: |
| MingW (https://www.msys2.org/) | :heavy_check_mark: | :heavy_check_mark: |
| ARMv6-M(M051)                  | :heavy_check_mark: | :x:                |
| ARMv7-M(STM32F103,STM32F302)   | :heavy_check_mark: | :x:                |
| STM8S103                       | :heavy_check_mark: | :x:                |

   linux tested on 
   * i686 (ubuntu-16.04)
   * x86_64 (centos-8.1)
   * arm (raspiberry 32bit)
   * aarch64 (raspiberry 64bit)
   * mipsel (openwrt ucLinux 3.10.14 for MT7628)
   * mips64 (fedora for loongson 3A-4000)

## Build

### Linux / MacOS

    cd build
    cmake .
    make

### Other platforms

| Platform  | Project                           | Tool chain                                    |
|-----------|-----------------------------------|-----------------------------------------------|
| Windows   | build\windows\s_task.sln          | visual studio 2019                            |
| Android   | build\android\cross_build_arm*.sh | android ndk 20, API level 21 (test in termux) |
| STM8S103  | build\stm8s103\Project.eww        | IAR workbench for STM8                        |
| STM32F103 | build\stm32f103\Project.uvproj    | Keil uVision5                                 |
| STM32F302 | build\stm32f302\Project.uvporj    | Keil uVision5                                 |
| M051      | build\m051\Project.uvporj         | Keil uVision5                                 |

## How to use in your project ?

On linux/unix like system, after cmake build, you may get the libraries for your project

* add libs_task.a to your project
* #include "s_task.h"
* #include "s_uv.h"
* build with predefined macro USE_LIBUV

On windows or other system, please find the project in folder "build" as the project template.

## How to make port ?

[Please find document here](porting.md)

## Contact

使用中有任何问题或建议，欢迎QQ加群 567780316 交流，您可能第一个加群的哦。。

![s_task交流群](qq.png)
