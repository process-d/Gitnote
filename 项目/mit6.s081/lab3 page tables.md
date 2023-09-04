task1、打印列表
参考freewalk递归打印，要注意打印的格式和实验要求的一致。页表是存放了目录的物理页，类似一个数组，每一项是一个物理页的基地址。物理页存放目录的叫做页表，存放数据叫做数据页。
```c
void vmprinthelper(pagetable_t pagetable, int level)
{
  // there are 2^9 = 512 PTEs in a page table.  
  for(int i = 0; i < 512; i++){  
    pte_t pte = pagetable[i];
   //判断当前页表项是否使用（页表项是指页表中的一项）
    if(pte & PTE_V){
        printf("..");
      for(int i = 0; i < level; i++)printf(" ..");
      uint64 child = PTE2PA(pte);
      printf("%d: pte %p pa %p\n", i, pte, child);
      if((pte & (PTE_R|PTE_W|PTE_X)) == 0){
          // this PTE points to a lower-level page table.  
          uint64 child = PTE2PA(pte);  
          vmprinthelper((pagetable_t)child,level+1);
      }
    }
  }
}
vmprint(pagetable_t pagetable)
{
  printf("page table %p\n", pagetable);
  vmprinthelper(pagetable, 0);
}
```
**task2：A kernel page table per process**
为每一个进程创建一个内核页表，并且内核页表要包括用户页表的映射。
跟着提示做：
1、向proc结构体中添加字段
```c
char name[16];               // Process name (debugging)
pagetable_t kpagetable;
```
2、添加了属性就要初始化和回收，先在vm.c中写kpagetable的初始化和回收代码。就是照到内核页表初始化写就行
```c
pagetable_t proc_kvmcreate()
{
   //创建页表
    pagetable_t pagetable = uvmcreate();
  //完成映射
    // uart registers  
    proc_kvmmap(pagetable,UART0, UART0, PGSIZE, PTE_R | PTE_W);
    // virtio mmio disk interface  
    proc_kvmmap(pagetable,VIRTIO0, VIRTIO0, PGSIZE, PTE_R | PTE_W);
    proc_kvmmap(pagetable,PLIC, PLIC, 0x400000, PTE_R | PTE_W);
    proc_kvmmap(pagetable,KERNBASE, KERNBASE, (uint64)etext-KERNBASE, PTE_R | PTE_X); 
    proc_kvmmap(pagetable,(uint64)etext, (uint64)etext, PHYSTOP-(uint64)etext, PTE_R | PTE_W); 
    proc_kvmmap(pagetable,TRAMPOLINE, (uint64)trampoline, PGSIZE, PTE_R | PTE_X);
    return pagetable;
}
void proc_kvmmap(pagetable_t pagetable, uint64 va, uint64 pa, uint64 sz, int perm)
{
    if (mappages(pagetable, va, sz, pa, perm) != 0)
        panic("proc_kvmmap");
}
```

回收页表，这里要搞清除uvmunmap和freewalk这两个函数的作用，uvmunmap是用来消除页表（最后一级页表）与数据页（存了数据的物理页）的映射，并且可以回收物理页的资源。freewalk是回收页表所占的物理页资源，所以在freewalk之前要先调用uvmunmap函数，不然数据页没法回收了。在释放进程的内核页表是不能释放数据页的，因为内核页表还映射着数据页，只能释放进程的内核页表。
```c
void proc_kvmfree(pagetable_t pagetable) {
    // 清空页表项和回收页表  
    for(int i = 0; i < 512; i++){
        pte_t pte = pagetable[i];
        if(pte & PTE_V){
            pagetable[i] = 0;
            if ((pte & (PTE_R|PTE_W|PTE_X)) == 0){
                uint64 child = PTE2PA(pte);
                proc_kvmfree((pagetable_t)child);
            }
        }
    }
    kfree((void*)pagetable);
}
```
fork时不用复制内核页表，不用修改fork
3、对allocproc函数修改，将initproc中的初始化内核栈的代码拷贝过来

```c
p->pagetable = proc_pagetable(p);  
  if(p->pagetable == 0){
    freeproc(p);
    release(&p->lock);
    return 0;
  }
//part2
p->kpagetable = proc_kvmcreate();  
  if (p->kpagetable==0) {
      freeproc(p);
      release(&p->lock);
      return 0;
  }
  //内核栈代码
  char *pa = kalloc();
  if(pa == 0)
      panic("kalloc");
  uint64 va = KSTACK((int) (p - proc));
  //记得修改页表为进程内核页表
  proc_kvmmap(p->kpagetable, va, (uint64)pa, PGSIZE, PTE_R | PTE_W);
  p->kstack = va;
```
4、在freeproc函数中释放进程的内核页表和内核栈
```c
//先要把内核栈对应的数据物理页回收  
  uvmunmap(p->kpagetable, p->kstack, 1, 1);
  p->kstack =0;
  if(p->kpagetable)
    proc_kvmfree(p->kpagetable);
  p->kpagetable = 0;
```
5、在进程调度时，将内核页表切换到进程的内核页表
```c
p->state = RUNNING;
c->proc = p;  
// switch kernel page table  
w_satp(MAKE_SATP(p->kpagetable));
sfence_vma();
```
**task3：Simplify copyin/copyinstr。这里是想把用户页表复制到内核页表。用户页表不会覆盖内核页表中的设备相关的映射吗？**

1、先写一个copy页表的函数
```c
void kvmmapuser(pagetable_t pagetable, pagetable_t kpagetable, uint64 oldsz, uint64 newsz)
{
    pte_t *pte_from, *pte_to;
    oldsz = PGROUNDUP(oldsz);
    for (uint64 i = oldsz; i < newsz; i += PGSIZE){
        if((pte_from = walk(pagetable, i, 0)) == 0)
            panic("kvmmapuser: src pte does not exist");
        if((pte_to = walk(kpagetable, i, 1)) == 0)
            panic("kvmmapuser: pte walk failed");
        *pte_to = (*pte_from)&(~PTE_U);
    }
}
```
2、分别在userinit、fork、sbrk、exec函数调用kvmmapuser函数

fork添加，task2不用复制内核页表，但是现在内核页表包含了用户页表，需要复制了
```c
 kvmmapuser(np->pagetable, np->kpagetable, 0, np->sz);
 safestrcpy(np->name, p->name, sizeof(p->name));
```
sbrk调用了growproc函数，在其中添加
```c
if(n > 0){
      if (PGROUNDUP(sz + n) >= PLIC){
          return -1;
      }
      if((sz = uvmalloc(p->pagetable, sz, sz + n)) == 0) {
          return -1;
      }
      kvmmapuser(p->pagetable, p->kpagetable, sz - n, sz);
  }
```
n小于0不用调用kvmmapuser函数。
exec函数中：在exec完成用户页表的建设和映射后就可以了。
```c
uvmclear(pagetable, sz-2*PGSIZE);
  sp = sz;
  stackbase = sp - PGSIZE;
  vmprint(pagetable);
  kvmmapuser(pagetable, p->kpagetable, 0, sz);
```


