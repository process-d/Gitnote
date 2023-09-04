实现用户级线程切换
1、定义上下文结构体，照着进程的context写就行了

```c
// 用户线程的上下文结构体
struct tcontext {
  uint64 ra;
  uint64 sp;
  // callee-saved  
  uint64 s0;
  uint64 s1;
  uint64 s2;
  uint64 s3;
  uint64 s4;
  uint64 s5;
  uint64 s6;
  uint64 s7;
  uint64 s8;
  uint64 s9;
  uint64 s10;
  uint64 s11;
};
```
2、修改thread结构
```c
struct thread {
  char            stack[STACK_SIZE];  /* the thread's stack */
  int             state;              /* FREE, RUNNING, RUNNABLE */
  struct tcontext context;            /* 用户进程上下文 */
};
```
3、模仿kernel/swtch.S，在user/uthread_switch.S中写入如下代码
```c
.text
.globl thread_switch
thread_switch:
    /* YOUR CODE HERE */
    sd ra, 0(a0)
    sd sp, 8(a0)
    sd s0, 16(a0)
    sd s1, 24(a0)
    sd s2, 32(a0)
    sd s3, 40(a0)
    sd s4, 48(a0)
    sd s5, 56(a0)
    sd s6, 64(a0)
    sd s7, 72(a0)
    sd s8, 80(a0)
    sd s9, 88(a0)
    sd s10, 96(a0)
    sd s11, 104(a0)
    ld ra, 0(a1)
    ld sp, 8(a1)
    ld s0, 16(a1)
    ld s1, 24(a1)
    ld s2, 32(a1)
    ld s3, 40(a1)
    ld s4, 48(a1)
    ld s5, 56(a1)
    ld s6, 64(a1)
    ld s7, 72(a1)
    ld s8, 80(a1)
    ld s9, 88(a1)
    ld s10, 96(a1)
    ld s11, 104(a1)
    ret    /* return to ra */
```
4、修改thread_scheduler
```c
if (current_thread != next_thread) {
   thread_switch((uint64)&t->context, (uint64)&current_thread->context);
} else
  next_thread = 0;
```
5、在thread_create中对context中的属性进行初始化，就和allocproc中进程一样
```c
t->context.ra = (uint64)func;                   // 设定函数返回地址  
t->context.sp = (uint64)t->stack + STACK_SIZE;  // 设定栈指针
```
