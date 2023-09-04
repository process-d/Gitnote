1、System call tracing

写一个能追踪输入的命令使用了哪些系统调用。

照着提示做

（1）在Makefile的UPROGS中添加$U/_trace，这个应该是让文件能编译链接（不熟悉makefile，猜的）

![[Pasted image 20230904161846.png]]

（2）将系统调用的原型添加到user/user.h，存根添加到user/usys.pl，以及将系统调用编号添加到kernel/syscall.h。
user/usys.pl文件
![[Pasted image 20230904161943.png]]
user/user.h文件
![[Pasted image 20230904161949.png]]
kernel/syscall.h
![[Pasted image 20230904162014.png]]
kernel/syscall.c文件
![[Pasted image 20230904162041.png]]
然后就开始写trace的实现了。首先sysfile.c是关于文件的系统调用，sysproc.c是关于进程的系统调用，将sys_trace写入到sysproc.c。
先在proc.h中的proc结构体中添加一个mask标记，用来判断要追踪哪些系统调用
![[Pasted image 20230904162117.png]]
由于还要追踪子进程的用了哪些系统调用，要改下fork函数，将父进程的mask赋值给子进程。
![[Pasted image 20230904162126.png]]
在inituser和freeproc中：加入p->mask = 0；
![[Pasted image 20230904162153.png]]
![[Pasted image 20230904162145.png]]
追踪函数的具体实现，就是把传入一个mask值，并赋值给当前进程的mask
```c
uint64 sys_trace(void)
{
  int mask;
  if(argint(0, &mask) < 0)
    return -1;
  myproc()->mask = mask;
  return 0;
}
```
由于系统调用是在syscall.c的syscall函数中完成的，只需要在函数返回之前加个判断是否为追踪的系统调用。
```c
char const *syscall_names[]={"fork", "exit", "wait", "pipe", "read","kill", "exec", "fstat", "chdir", "dup", "getpid", "sbrk", "sleep","uptime", "open", "write", "mknod", "unlink", "link", "mkdir","close","trace","sysinfo"};
void syscall(void)
{
  int num;
  struct proc *p = myproc();
  num = p->trapframe->a7;
  if(num > 0 && num < NELEM(syscalls) && syscalls[num]) {
    p->trapframe->a0 = syscalls[num]();
    if ((p->mask) & (1 << num)){
      printf("%d: syscall %s -> %d\n", p->pid, syscall_names[num - 1], p->trapframe->a0);
    }
  } else {
    printf("%d %s: unknown sys call %d\n",
            p->pid, p->name, num);
    p->trapframe->a0 = -1;
  }
}
```

**2、Sysinfo：打印系统的内部信息**
照着第一个实验步骤设置系统调用sysinfo，
要向用户态返回一个结构体：包括空闲内存和未使用的进程数
空闲内存：物理内存是通过链表将一个个大小为PGSIZE的物理页连接起来的
```c
uint64 freemem(void)
{
  struct run *r;
  uint64 freepage = 0;
  acquire(&kmem.lock);
  r = kmem.freelist;
  while (r)
  {
    freepage += 1;
    r = r->next;
  }
  release(&kmem.lock);
  return (freepage << PGSHIFT);
}
```


未使用的进程数：xv6维护一个进程数组，这是xv6的所有进程，遍历数组找到处于UNUSED状态的进程即可，统计下数量。
```c
uint64 usedproc(void)
{
  struct proc *p;
  uint64 usednum = 0;
  for(p = proc; p < &proc[NPROC]; p++)
  {
    if(p->state != UNUSED) {
      usednum++;
    }
  }
  return usednum;
}

```

然后就是将结构体从内核态返回到用户态了。参考sys_fstat和filestat函数。通过将用户态的结构体的地址读到内核态中，在利用copyout函数将内核态中的数据写入到用户态。
```c
uint64 sys_sysinfo(void)
{
  uint64 addr;
  if(argaddr(0, &addr) < 0)
    return -1;
  struct proc *p = myproc();
  struct sysinfo info;
  info.freemem = freemem();
  info.nproc = usedproc();
  if(copyout(p->pagetable, addr, (char *)&info, sizeof(info)) < 0)
      return -1;
  return 0;
}
```
