就是多个CPU对应一个物理页的空闲链表，太慢了，要每个CPU都有自己的空闲链表
1、新加一个结构体
```c
struct {
  struct spinlock lock;
  struct run *freelist;
} kmem[NCPU];
```
2、修改kinit，kinit只会被一个CPU调用，将物理页都分配到当前cpu的空闲链表
```c
void kinit()
{
  char lockname[8];
  for(int i = 0;i < NCPU; i++) {
    snprintf(lockname, sizeof(lockname), "kmem_%d", i);
    initlock(&kmem[i].lock, lockname);
  }
  freerange(end, (void*)PHYSTOP);
}
```

3、修改kfree，释放当前cpu对应的空闲链表
```c
void kfree(void *pa)
{
  struct run *r;
  if(((uint64)pa % PGSIZE) != 0 || (char*)pa < end || (uint64)pa >= PHYSTOP)
    panic("kfree");
  // Fill with junk to catch dangling refs.  
  memset(pa, 1, PGSIZE);
  r = (struct run*)pa;
  push_off();  // 关中断  
  int id = cpuid();
  acquire(&kmem[id].lock);
  r->next = kmem[id].freelist;
  kmem[id].freelist = r;
  release(&kmem[id].lock);
  pop_off();  //开中断  
}
```
4、修改kalloc，使得在当前CPU的空闲列表没有可分配内存时窃取其他cpu的空闲链表
```c
void *kalloc(void)
{
  struct run *r;
  push_off();// 关中断  
  int id = cpuid();
  acquire(&kmem[id].lock);
  r = kmem[id].freelist;
  if(r)
    kmem[id].freelist = r->next;
  else {
    int antid;  // another id  
    // 遍历所有CPU的空闲列表  
    for(antid = 0; antid < NCPU; ++antid) {
      if(antid == id)
        continue;
      acquire(&kmem[antid].lock);
      r = kmem[antid].freelist;
      if(r) {
        kmem[antid].freelist = r->next;
        release(&kmem[antid].lock);
        break;
      }
      release(&kmem[antid].lock);
    }
  }
  release(&kmem[id].lock);
  pop_off();  //开中断
  if(r)
    memset((char*)r, 5, PGSIZE); // fill with junk  
  return (void*)r;
}
```
task2 ：Buffer cache

跟上一个task相似，buf是磁盘的缓冲区，多个CPU可能会使用buf，只有一把锁太慢了。

1、根据提示，定义哈希桶结构

#define NBUCKET 13  
#define HASH(id) (id % NBUCKET)

struct hashbuf {

  struct buf head;       // 头节点  
  struct spinlock lock;  // 锁  
};

struct {

  struct buf buf[NBUF];

  struct hashbuf buckets[NBUCKET];  // 散列桶  
} bcache;

2、修改binit

void

binit(void) {

  struct buf* b;

  char lockname[16];

  for(int i = 0; i < NBUCKET; ++i) {

    // 初始化散列桶的自旋锁  
    snprintf(lockname, sizeof(lockname), "bcache_%d", i);

    initlock(&bcache.buckets[i].lock, lockname);

    // 初始化散列桶的头节点  
    bcache.buckets[i].head.prev = &bcache.buckets[i].head;

    bcache.buckets[i].head.next = &bcache.buckets[i].head;

  }

  // Create linked list of buffers  
  for(b = bcache.buf; b < bcache.buf + NBUF; b++) {

    // 利用头插法初始化缓冲区列表,全部放到散列桶0上  
    b->next = bcache.buckets[0].head.next;

    b->prev = &bcache.buckets[0].head;

    initsleeplock(&b->lock, "buffer");

    bcache.buckets[0].head.next->prev = b;

    bcache.buckets[0].head.next = b;

  }

}

3、添加时间戳timestamp，

struct buf {

  ...

  ...

  uint timestamp;  // 时间戳  
};

4、修改brelse，不再是全局锁了

void

brelse(struct buf* b) {

  if(!holdingsleep(&b->lock))

    panic("brelse");

  int bid = HASH(b->blockno);

  releasesleep(&b->lock);

  acquire(&bcache.buckets[bid].lock);

  b->refcnt--;

  // 更新时间戳  
  // 由于LRU改为使用时间戳判定，不再需要头插法  
  acquire(&tickslock);

  b->timestamp = ticks;

  release(&tickslock);

  release(&bcache.buckets[bid].lock);

}

5、更改bget，最麻烦的一个。当没有找到指定的缓冲区时进行分配，分配方式是优先从当前列表遍历，找到一个没有引用且timestamp最小的缓冲区，如果没有就申请下一个桶的锁，并遍历该桶，找到后将该缓冲区从原来的桶移动到当前桶中，最多将所有桶都遍历完。这里也可以修改，先从第一个桶中去找，毕竟所有的空闲缓存区都在第一个桶中。

static struct buf*

bget(uint dev, uint blockno) {

  struct buf* b;

  int bid = HASH(blockno);

  acquire(&bcache.buckets[bid].lock);

  // Is the block already cached?  
  for(b = bcache.buckets[bid].head.next; b != &bcache.buckets[bid].head; b = b->next) {

    if(b->dev == dev && b->blockno == blockno) {

      b->refcnt++;

      // 记录使用时间戳  
      acquire(&tickslock);

      b->timestamp = ticks;

      release(&tickslock);

      release(&bcache.buckets[bid].lock);

      acquiresleep(&b->lock);

      return b;

    }

  }

  // Not cached.  
  b = 0;

  struct buf* tmp;

  // Recycle the least recently used (LRU) unused buffer.  
  // 从当前散列桶开始查找  
  for(int i = bid, cycle = 0; cycle != NBUCKET; i = (i + 1) % NBUCKET) {

    ++cycle;

    // 如果遍历到当前散列桶，则不重新获取锁  
    if(i != bid) {

      if(!holding(&bcache.buckets[i].lock))

        acquire(&bcache.buckets[i].lock);

      else

        continue;

    }

    for(tmp = bcache.buckets[i].head.next; tmp != &bcache.buckets[i].head; tmp = tmp->next)

      // 使用时间戳进行LRU算法，而不是根据结点在链表中的位置  
      if(tmp->refcnt == 0 && (b == 0 || tmp->timestamp < b->timestamp))

        b = tmp;

    if(b) {

      // 如果是从其他散列桶窃取的，则将其以头插法插入到当前桶  
      if(i != bid) {

        b->next->prev = b->prev;

        b->prev->next = b->next;

        release(&bcache.buckets[i].lock);

        b->next = bcache.buckets[bid].head.next;

        b->prev = &bcache.buckets[bid].head;

        bcache.buckets[bid].head.next->prev = b;

        bcache.buckets[bid].head.next = b;

      }

      b->dev = dev;

      b->blockno = blockno;

      b->valid = 0;

      b->refcnt = 1;

      acquire(&tickslock);

      b->timestamp = ticks;

      release(&tickslock);

      release(&bcache.buckets[bid].lock);

      acquiresleep(&b->lock);

      return b;

    } else {

      // 在当前散列桶中未找到，则直接释放锁  
      if(i != bid)

        release(&bcache.buckets[i].lock);

    }

  }

  panic("bget: no buffers");

}

6、更改bpin和unpin

void

bpin(struct buf* b) {

  int bid = HASH(b->blockno);

  acquire(&bcache.buckets[bid].lock);

  b->refcnt++;

  release(&bcache.buckets[bid].lock);

}

void

bunpin(struct buf* b) {

  int bid = HASH(b->blockno);

  acquire(&bcache.buckets[bid].lock);

  b->refcnt--;

  release(&bcache.buckets[bid].lock);

}