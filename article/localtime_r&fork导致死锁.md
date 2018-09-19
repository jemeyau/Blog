### 1. 背景

[Redis][redis] 自己实现了一个名为 [`nolocks_localtime`][nolocks_localtime] 的函数，并且在注释中解释了该函数存在的原因。

注释：
```c
/* This is a safe version of localtime() which contains no locks and is
 * fork() friendly. Even the _r version of localtime() cannot be used safely
 * in Redis. Another thread may be calling localtime() while the main thread
 * forks(). Later when the child process calls localtime() again, for instance
 * in order to log something to the Redis log, it may deadlock: in the copy
 * of the address space of the forked process the lock will never be released.
```

大概是说系统自带的`localtime`甚至是`localtime_r`在 Redis 的某些场景下也是不安全的，会导致死锁。例如：父进程中同时有两个线程在运行，如果在线程1调用`localtime_r`尚未返回时，线程2`fork`了一个子进程，并且该子进程也调用了`localtime_r`（比如打日志时会频繁调用该函数），那么子进程会因为`localtime_r`而进入死锁。

### 2. 原因：

因为`localtime_r`在将`Unix时间戳`转换为`本地时间时`会获取系统的时区信息，而时区信息又存放在一个全局变量中`__tzname`。由于`localtime_r`本身是线程安全的，所以在函数内部会先取得`__tzname`对应的锁。如果此时`fork`子进程，子进程会继承父进程的锁信息，但是又不会被释放，所以当子进程调用`localtime_r`时就死锁了。

`localtime_r`代码:
```c
/* Return the `struct tm' representation of *T in local time,
   using *TP to store the result.  */
struct tm *
__localtime_r (const time_t *t, struct tm *tp)
{
  return __tz_convert (t, 1, tp);
}
```
`__tz_convert`代码 （省略了部分代码）
```c
//全局锁的声明
__libc_lock_define_initialized (static, tzset_lock)

/* Return the `struct tm' representation of *TIMER in the local timezone.
   Use local time if USE_LOCALTIME is nonzero, UTC otherwise.  */
struct tm *
__tz_convert (const time_t *timer, int use_localtime, struct tm *tp)
{
  //加锁
  __libc_lock_lock (tzset_lock);

  //转换操作
  
  //释放锁
  __libc_lock_unlock (tzset_lock);

  return tp;
}
```

### 3. 以下是重现该场景的代码：
```c
#include <time.h>
#include <unistd.h>
#include <stdio.h>
#include <stdlib.h>
#include <pthread.h>
#include <sys/types.h>
#include <sys/wait.h>
#include <signal.h>

void *thread_func(void *arg)
{
    time_t t = time(NULL);
    struct tm date;
    for (;;)
    {
        localtime_r(&t, &date);
        usleep(10);
    }
}

void sigalrm(int sig)
{
    //'\n' is needed, without it the printf() would be buffered,
    //and nothing output to the console.
    printf("sig\n");
}

void main()
{
    pthread_t id;
    pthread_create(&id, NULL, thread_func, NULL);

    time_t t = time(NULL);
    struct tm date;
    for (;;)
    {
        pid_t pid = fork();

        if (0 == pid)
        {
            //we would go into sigalrm() **only if** we get stuck.
            //because if not, process would exit by 'exit(0)'.
            signal(SIGALRM, sigalrm);
            alarm(1);

            //if there are lots of "sig" printed,
            //that means the child process has been deadlocked by localtime_r().
            localtime_r(&t, &date);

            exit(0);
        }

        wait3(NULL, WNOHANG, NULL);
        usleep(2500);
    }
}
```

### 4. 其它
可以得出如果有其他类似`localtime_r`的函数调用，如参考1中的`setenv`和`unsetenv`，也会导致死锁。但是又不是必现的，所以，如果工程中有类似的场景就需要注意了，说不定哪天就踩坑了。

#### References:

1. [Multithreaded forking and environment access locks](https://rachelbythebay.com/w/2014/08/16/forkenv/)

[redis]: https://github.com/antirez/redis
[nolocks_localtime]:https://github.com/antirez/redis/blob/unstable/src/localtime.c#L59