<!--
 * @Author: Firefly
 * @Date: 2020-09-10 14:31:43
 * @Descripttion: 
 * @LastEditTime: 2020-09-10 19:47:08
-->



### bio.c buffer input ouput

对每一个块进行缓存， 注意是块， 不用管是文件系统组织的， 因此实现新的文件系统时， 这个不用修改

### file.c 

定义了一个文件的数据结构和inode，一个inode 存储的直接磁盘物理地址为 13 * block size大小（
查看 inode 的定义），因此没有大文件的存储？？？， 这个不用修改

### ide.c 

对驱动的调用， 每次处理一个 block， 其它放到队列中， 处理了磁盘的中断, 这个也不用修改

### fs.c

需要修改

现在的磁盘已经获取不到相关的物理信息了，所以现在的一些FS使用的方法是将磁盘分为block group，每
一个group是在磁盘上是连续的。

```
+--------------------+-------------------+----------------------+
|                    |                   |                      |
|      Group 0       |     Group 1       |    Group 2           |
|                    |                   |                      |
+--------------------+-------------------+----------------------+
```

为什么可以加快速度？ 一个inode 直接存储 13 个 物理磁盘地址， 如果不分组，可能13块分割的很远，
当分配的时候，尽可能地往同一个 group 分配，seek 的时候可以减少seek距离因而可以减少seek 时间

在原来的基础上， inode bitmap、 data bitmap，用于指示后面的indoes、data哪些部分被使用了，
方便快速找到哪些group 中满足分配的空间。


实际的unix 实现：
- 对于目录: 查找有最少数量的目录和最多空闲的inodes中，将目录的inode和数据放在这里。
- 对于文件: 保证inode和数据块在一个group内，将同一个目录下面的文件放在一个group内。


我们的实现思路， 尽可能将数据库放到同一个组中， 查找哪里的哪一个 group中的数据最多空闲， 因此将 inode 存的 `addrs` 尽可能放到一个 group 中。



```
superblock sp;
sb.size = xint(FSSIZE);
sb.nblocks = xint(nblocks);
sb.ninodes = xint(NINODES);
sb.nlog = xint(nlog);
sb.logstart = xint(2);
sb.inodestart = xint(2+nlog);
sb.bmapstart = xint(2+nlog+ninodeblocks);
```
已经提供磁盘一共有多少块的数据，直接在 superblock 中拿取，然后分组， 在分配 block 时，修改 `static uint balloc(uint dev)）`， 直接使用偏移量取， 根据偏移量来定位group，根据位图判断哪些还没有分配！

超级快的初始化是os 在启动是会 调用  mkfs.c 进行分配磁盘第一块！ 因此在后面可以直接用



实现步骤：

首先， 查看源代码是如何分配一整块内存的，首先定位到如下代码，查看当写入文件时的操作：
```c
int writei(struct inode *ip, char *src, uint off, uint n) {
  uint tot, m;
  struct buf *bp;
  if (ip->type == T_DEV) {
    if (ip->major < 0 || ip->major >= NDEV || !devsw[ip->major].write)
      return -1;
    return devsw[ip->major].write(ip, src, n);
  }
  if (off > ip->size || off + n < off) return -1;
  if (off + n > MAXFILE * BSIZE) return -1;
  // bmap 这里连续分配内
  for (tot = 0; tot < n; tot += m, off += m, src += m) {
    bp = bread(ip->dev, bmap(ip, off / BSIZE));
    m = min(n - tot, BSIZE - off % BSIZE);
    memmove(bp->data + off % BSIZE, src, m);
    log_write(bp);
    brelse(bp);
  }
  if (n > 0 && off > ip->size) {
    ip->size = off;
    iupdate(ip);
  }
  return n;
}
```

这里调用 bmap 连续分配了内存（block map 映射一块内存）,继续看 bmap 的实现

```c
// bn 为 block 的编号， ip为inode， 也就是说给 bn 分配一块磁盘块
static uint bmap(struct inode *ip, uint bn) {
  uint addr, *a;
  struct buf *bp;
  if (bn < NDIRECT) {
    if ((addr = ip->addrs[bn]) == 0)
      ip->addrs[bn] = addr = balloc_ffs(ip->dev, ip->addrs, bn);
    return addr;
  }
  bn -= NDIRECT;
  if (bn < NINDIRECT) {
    // Load indirect block, allocating if necessary.
    if ((addr = ip->addrs[NDIRECT]) == 0)
      ip->addrs[NDIRECT] = addr = balloc_ffs(ip->dev, ip->addrs, bn);
    bp = bread(ip->dev, addr);
    a = (uint *)bp->data;
    if ((addr = a[bn]) == 0) {
      a[bn] = addr = balloc_ffs(ip->dev, ip->addrs, bn);
      log_write(bp);
    }
    brelse(bp);
    return addr;
  }
  panic("bmap: out of range");
}
```
可以看到当磁盘存储不够的时候会分配足够的内存出来！而 bn 刚好是分配的
内存编号，我们的实现的实验一的依据块组分配内存方案从这里入手！
要使分配的内存连续，需要历史的分配数据， 于是乎，将历史分配的地址也传进去！ 
`balloc_ffs(ip->dev, ip->addrs, bn);`

分配的策略就是：
```c
static uint balloc_ffs(uint dev, uint *addr, uint bn){
  int group, ret;
  struct buf *bp;

  bp = bread(dev, 1); // 读取出头
  group = 0;
  // 第一次分配，选择最多空闲的组分配分配
  // 作用： 当分配不足时，可以减少换区 
  if(!bn){
    ret = balloc_select(dev,bp);
  }
//默认选择上一次分配的块所在的组进行分配， 这样连续分配空间时， 就可以把块都放在
//同一组了！！
//然后通过根据 bn 前一个块地址的数值，判断所在的所在分组的地方， 把bn也分配在
//那一组
  else {
    group = addr[bn - 1] / GSIZE;
    if((ret = balloc_by_group(dev, group)) == -1){
      ret = balloc_select(dev,bp);
    }
  }
  bp->data[sizeof(bp->data) - group - 1]++;
  brelse(bp);
  return ret;
}
```

任务二：优化
当文件还没有分配过磁盘空间时如何快速找到哪个组分配的空间最少？
我们使用的策略是，超级块的尾部记录每个组的分配数量， 然后就可以直接取了！！
`bp->data[sizeof(bp->data) - group - 1]++;`
bp 是超级快的缓存， 把data数据计算到尾部的位置加上 1， 表示这个组分配数量增加了
一位，反之同理！


任务三：

由于大文件会占据一大块的空间，访问后面的文件时会造成比较昂贵的seek操作。FFS会将文件的前面几个block放到文件所在的group，而将后面的数据分成比较大的chunk，保存在其它的group中。虽然这里可能造成一些额外的seek操作，但是由于文件比较大，会被数据的传输时间”掩盖”.


我们把大文件的分配放到最尾部， 然后让这这些连续的几个块合并在一起变成一个较大的块， 这样所需要的索引数量就可以减少，从而达到减少seek的时间花费。```w`                                                                                                                                                       