就是fork函数，完成写时复制的功能，就是页表时写时复制。
1、在kernel/riscv.h中选取PTE中的保留位定义标记一个页面是否为COW Fork页面的标志位
`#define PTE_F (1L << 8)`
2、一个页面是可以有多个引用的，在kalloc.c中修改。添加一个结构体，记录每一个页面的引用数。
```c
struct ref_stru {
  struct spinlock lock;
  int cnt[PHYSTOP / PGSIZE];  // 引用计数  
} ref;
```
修改kalloc.c中的函数
```c
void kinit()
{
  initlock(&kmem.lock, "kmem");
  initlock(&ref.lock, "ref");
  freerange(end, (void*)PHYSTOP);
}
void kfree(void *pa)
{
  struct run *r;
  if(((uint64)pa % PGSIZE) != 0 || (char*)pa < end || (uint64)pa >= PHYSTOP)
    panic("kfree");
  // 只有当引用计数为0了才回收空间  
  // 否则只是将引用计数减1  
  acquire(&ref.lock);
  if(--ref.cnt[(uint64)pa / PGSIZE] == 0) {
    release(&ref.lock);
    r = (struct run*)pa;
    // Fill with junk to catch dangling refs.  
    memset(pa, 1, PGSIZE);
    acquire(&kmem.lock);
    r->next = kmem.freelist;
    kmem.freelist = r;
    release(&kmem.lock);
  } else {
    release(&ref.lock);
  }
}
void* kalloc(void)
{
  struct run *r;
  acquire(&kmem.lock);
  r = kmem.freelist;
  if(r) {
    kmem.freelist = r->next;
    acquire(&ref.lock);
    ref.cnt[(uint64)r / PGSIZE] = 1;  // 将引用计数初始化为1  
    release(&ref.lock);
  }
  release(&kmem.lock);
  if(r)
    memset((char*)r, 5, PGSIZE); // fill with junk  
  return (void*)r;
}
void freerange(void *pa_start, void *pa_end)
{
  char *p;
  p = (char*)PGROUNDUP((uint64)pa_start);
  for(; p + PGSIZE <= (char*)pa_end; p += PGSIZE) {
    // 在kfree中将会对cnt[]减1，这里要先设为1，否则就会减成负数  
    ref.cnt[(uint64)p / PGSIZE] = 1;
    kfree(p);
  }
}
```

3、还需要增加几个关于引用的函数，当前引用数、增加引用和减少引用（已经有了kfree）
```c
int krefcnt(void* pa) {
  return ref.cnt[(uint64)pa / PGSIZE];
}
int kaddrefcnt(void* pa) {
  if(((uint64)pa % PGSIZE) != 0 || (char*)pa < end || (uint64)pa >= PHYSTOP)
    return -1;
  acquire(&ref.lock);
  ++ref.cnt[(uint64)pa / PGSIZE];
  release(&ref.lock);
  return 0;
}
```
4、对uvmcopy进行修改
```c
int uvmcopy(pagetable_t old, pagetable_t new, uint64 sz)
{
  pte_t *pte;
  uint64 pa, i;
  uint flags;
  for(i = 0; i < sz; i += PGSIZE){
    if((pte = walk(old, i, 0)) == 0)
      panic("uvmcopy: pte should exist");
    if((*pte & PTE_V) == 0)
      panic("uvmcopy: page not present");
    pa = PTE2PA(*pte);
    flags = PTE_FLAGS(*pte);
    // 仅对可写页面设置COW标记  
    if(flags & PTE_W) {
      // 禁用写并设置COW Fork标记  
      flags = (flags | PTE_F) & ~PTE_W;
      *pte = PA2PTE(pa) | flags;
    }
    if(mappages(new, i, PGSIZE, pa, flags) != 0) {
      uvmunmap(new, 0, i / PGSIZE, 1);
      return -1;
    }
    // 增加内存的引用计数  
    kaddrefcnt((char*)pa);
  }
  return 0;
}
```
5、修改usertrap函数
5.1、需要判断当前虚拟地址对应的物理页是否有写时复制的标志。
可以写在vm.c中
```c
int cowpage(pagetable_t pagetable, uint64 va) {
  if(va >= MAXVA)
    return -1;
  pte_t* pte = walk(pagetable, va, 0);
  if(pte == 0)
    return -1;
  if((*pte & PTE_V) == 0)
    return -1;
  return (*pte & PTE_F ? 0 : -1);
}
```
5.2、然后分配物理页
```c
void* cowalloc(pagetable_t pagetable, uint64 va) {
  if(va % PGSIZE != 0)
    return 0;
  uint64 pa = walkaddr(pagetable, va);  // 获取对应的物理地址  
  if(pa == 0)
    return 0;
  pte_t* pte = walk(pagetable, va, 0);  // 获取对应的PTE
  if(krefcnt((char*)pa) == 1) {
    // 只剩一个进程对此物理地址存在引用  
    // 则直接修改对应的PTE即可  
    *pte |= PTE_W;
    *pte &= ~PTE_F;
    return (void*)pa;
  } else {
    // 多个进程对物理内存存在引用  
    // 需要分配新的页面，并拷贝旧页面的内容  
    char* mem = kalloc();
    if(mem == 0)
      return 0;
    // 复制旧页面内容到新页  
    memmove(mem, (char*)pa, PGSIZE);
    // 清除PTE_V，否则在mappagges中会判定为remap  
    *pte &= ~PTE_V;
    // 为新页面添加映射  
    if(mappages(pagetable, va, PGSIZE, (uint64)mem, (PTE_FLAGS(*pte) | PTE_W) & ~PTE_F) != 0) {
      kfree(mem);
      *pte |= PTE_V;
      return 0;
    }
    // 将原来的物理内存引用计数减1  
    kfree((char*)PGROUNDDOWN(pa));
    return mem;
  }
}
```
5.3、修改usertrap函数
```c
uint64 cause = r_scause();
if(cause == 8) {
  ...
} else if((which_dev = devintr()) != 0){
  // ok  
} else if(cause == 13 || cause == 15) {
  uint64 fault_va = r_stval();  // 获取出错的虚拟地址  
  if(fault_va >= p->sz
    || cowpage(p->pagetable, fault_va) != 0
    || cowalloc(p->pagetable, PGROUNDDOWN(fault_va)) == 0)
    p->killed = 1;
} else {
  ...
}
```
