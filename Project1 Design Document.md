# <center>设计文档
<center>11510050  

王戈扬</center>
## TASK 1
### 源码解析  
原来实现sleep的代码如下

```C
void
timer_sleep (int64_t ticks) //Is this a alarm thread?   YES
{
  int64_t start = timer_ticks (); //获取开始时间

  ASSERT (intr_get_level () == INTR_ON);
  while (timer_elapsed (start) < ticks) //未超过给定时间 循环循环执行（sleep）
    thread_yield ();
}
```
thread_yield():  
这个函数

```C
/* Yields the CPU.  The current thread is not put to sleep and
   may be scheduled again immediately at the scheduler's whim. */
void
thread_yield (void)
{
  struct thread *cur = thread_current (); /*获得当前线程*/
  enum intr_level old_level;

  ASSERT (!intr_context ());

  old_level = intr_disable ();
  if (cur != idle_thread)
    list_push_back (&ready_list, &cur->elem); //该进程重返就绪队列
  cur->status = THREAD_READY; //将状态改为就绪
  schedule (); //进行调度
  intr_set_level (old_level);
}
```
### 实现机制解析  
这是忙等待，将计时线程不断改为就绪状态加入就绪队列，占用CPU资源
### 改进思路  
思路来源：http://www.cnblogs.com/laiy/p/pintos_project1_thread.html  
### 1. Data structure and functions  
``` C
struct thread
  {
    /* Owned by thread.c. */
    tid_t tid;                          /* Thread identifier. */
    enum thread_status status;          /* Thread state. */
    char name[16];                      /* Name (for debugging purposes). */
    uint8_t *stack;                     /* Saved stack pointer. */
    int priority;                       /* Priority. */
    struct list_elem allelem;           /* List element for all threads list. */

    int64_t alarm_ticks               /* Left ticks number that the thread should been resume*/

    /* Shared between thread.c and synch.c. */
    struct list_elem elem;              /* List element. */

#ifdef USERPROG
    /* Owned by userprog/process.c. */
    uint32_t *pagedir;                  /* Page directory. */
#endif

    /* Owned by thread.c. */
    unsigned magic;                     /* Detects stack overflow. */
  };

```
[//]:/*/
改写 timer_sleep 添加当前线程结构体alarm_ticks 然后阻塞
```C
void
timer_sleep (int64_t ticks) //Is this a alarm thread?   YES
{
  ...
}
```

每个tick系统会产生时钟中断，中断会调用thread.c中的thread_tick（void）将被阻塞的sleep线程计时器-1（HOW？）
```C

/* Called by the timer interrupt handler at each timer tick.
   Thus, this function runs in an external interrupt context. */
void
thread_tick (void)
{
  struct thread *t = thread_current ();

  /* Update statistics. \*/
  if (t == idle_thread)
    idle_ticks++;
#ifdef USERPROG
  else if (t->pagedir != NULL)
    user_ticks++;
#endif
  else
    kernel_ticks++;

  //check alarm thread. or add at timer_interrupt which calls this function
  thread_foreach(update_alarm,NULL);

  /* Enforce preemption.\*/  
  if (++thread_ticks >= TIME_SLICE)
    intr_yield_on_return ();
}
```

利用此函数便历所有线程进行检查，`*action` 传入函数，将每个线程的alarm_ticks-1 `thread_action_func`是一个类型声明
```C
void thread_foreach(thread_action_func *action,void *aux)
```
将这个程的alarm_ticks-1
```C
void update_alarm(struct thread *t){
      // if alarm_ticks not 0
      if(t->alarm_ticks ){
        t->alarm_ticks--;
        if (t->alarm_ticks==0){
          thread_unblock(t);//wake up this thread (go to ready queue)
        }
      }
      // otherwise not alarm, do nothing.
}
```
### 2. Algorithms  
调用timer_sleep的时候直接把线程阻塞掉，然后给线程结构体加一个成员alarm_ticks来记录这个线程被sleep了多少时间， 然后利用操作系统自身的时钟中断（每个tick会执行一次）加入对线程状态的检测， 每次检测将ticks_blocked减1, 如果减到0就唤醒这个线程。
### 3. Synchronization
### 4. Rationale

## Task 2
schedule() is responsible for switching threads. It is internal to threads/thread.c and called only by
the three public thread functions that need to switch threads: thread_block(), thread_exit(), and
thread_yield(). Before any of these functions call schedule(), they disable interrupts (or ensure that
they are already disabled) and then change the running thread’s state to something other than running.

## 思路
在ticks 打断的时候检查时间片，到时间就轮换。
注意不能轮换不可被打断的线程  

thread.c
```c
void
thread_tick (void)
...
/* Enforce preemption. */
if (++thread_ticks >= TIME_SLICE)
  intr_yield_on_return ();
...
interrupt.c
```c
/* Returns true during processing of an external interrupt
   and false at all other times. */
bool
intr_context (void)
{
  return in_external_intr;
}

/* During processing of an external interrupt, directs the
   interrupt handler to yield to a new process just before
   returning from the interrupt.  May not be called at any other
   time. */
void
intr_yield_on_return (void)
{
  ASSERT (intr_context ());
  yield_on_return = true;
  //make some change
}
```
## 3 思路
将ready_list改为最大堆的数据结构，堆顶是优先级最高的线程。保持接口。

```c
bool thread_elem_less(struct * elem , struct * e, void * aux ){
  //return false if elem has higher priority of e
}
```
