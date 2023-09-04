**task1：Eliminate allocation from sbrk**
sys_sbrk不再调用growproc函数，就只是加+n
```c
uint64 sys_sbrk(void)
{
  int addr;
  int n;
  if(argint(0, &n) < 0)
    return -1;
  addr = myproc()->sz;
  myproc()->sz += n;
  return addr;
}
```
**task2: Lazy allocation**
1、在sbrk系统调用时，没有物理分配内存，在使用这段虚拟地址时，会发生异常。
在trap.c的usertrap中处理异常
```c
else if (r_scause() == 13||r_scause()==15){
      uint64 va = r_stval();
      char* pa;
      if (va>p->sz || va<PGROUNDUP(p->trapframe->sp)||(pa=kalloc())==0) {
          p->killed=1;
      } else {
          memset(pa, 0, PGSIZE);
          if ((mappages(p->pagetable, PGROUNDDOWN(va),PGSIZE,(uint64)pa,PTE_R | PTE_W | PTE_X | PTE_U)) !=0) {
              kfree(pa);
              p->killed = 1;
          }
      }
  } else if((which_dev = devintr()) != 0){
    // ok  
  }
```
2、由于sbrk只加了sz，没有分配物理页，所以要修改uvmunmap函数。有的虚拟地址可以会没有分配物理页，进程就开始释放了。
因为进程的用户页表是直到进程kill才会释放页表内存和页表映射的数据页，平时的增减内存，只是会把最后一级的页表的pte设置为0，同时删除相关数据页的映射。
所以uvmunmap要将`*pte&PTE_v==0`之后也改为continue
```c
for(a = va; a < va + npages*PGSIZE; a += PGSIZE){
    if((pte = walk(pagetable, a, 0)) == 0)
      //panic("uvmunmap: walk");
        continue;
    if((*pte & PTE_V) == 0)
      //panic("uvmunmap: not mapped");
        continue;
    if(PTE_FLAGS(*pte) == PTE_V)
      panic("uvmunmap: not a leaf");
}
```
3、处理sbrk的参数为负数的情况。n为负数就直接取消映射
```c
uint64 sys_sbrk(void)
{
  int addr;
  int n;
  if(argint(0, &n) < 0)
    return -1;
  struct proc* p = myproc();
  addr = p->sz;
  uint64 sz = p->sz;
  if(n > 0) {
    // lazy allocation  
    p->sz += n;
  } else if(sz + n > 0) {
    sz = uvmdealloc(p->pagetable, sz, sz + n);
    p->sz = sz;
  } else {
    return -1;
  }
  return addr;
}
```

4、处理fork拷贝
fork拷贝有调用uvmcopy函数拷贝用户页表，用continue代替panic就行
```c
int uvmcopy(pagetable_t old, pagetable_t new, uint64 sz)
{
  ...
  for(i = 0; i < sz; i += PGSIZE){
    if((pte = walk(old, i, 0)) == 0)
      continue;
    if((*pte & PTE_V) == 0)
      continue;
    ...
  }
  ...
}
```
5、还要处理一种情况，sbrk()之后，直接调用write函数对分配的内存进行读写，这样只会进入usertrap的系统调用，不会进入缺页异常处理。在接收地址时，对虚拟地址进行判断。
sbrk之后，一种是直接使用sbrk的内存，这样程序会异常，从而进入缺页处理；另一种是write使用地址，但是这种不会走缺页异常处理，而是进入了系统调用。
```c
int argaddr(int n, uint64 *ip)
{
  *ip = argraw(n);
  struct proc* p = myproc();
  // 处理向系统调用传入lazy allocation地址的情况  
  if(walkaddr(p->pagetable, *ip) == 0) {
    if(PGROUNDUP(p->trapframe->sp) - 1 < *ip && *ip < p->sz) {
      char* pa = kalloc();
      if(pa == 0)
        return -1;
      memset(pa, 0, PGSIZE);
      if(mappages(p->pagetable, PGROUNDDOWN(*ip), PGSIZE, (uint64)pa, PTE_R | PTE_W | PTE_X | PTE_U) != 0) {
        kfree(pa);
        return -1;
      }
    } else {
      return -1;
    }
  }
  return 0;
}
```
#TODO
一个问题：为什么程序异常就会进入内核？在内核发生异常不会再次进入trap吗？