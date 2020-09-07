


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


我们的实现思路， 尽可能将数据库放到同一个组中， 查找哪里的哪一个 group中的数据最多空闲， 因此将
inode 存的 `addrs` 尽可能放到一个 group 中。

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
已经提供磁盘一共有多少块的数据，直接在 superblock 中拿取，然后分组， 在分配 block 时，
修改 `static uint balloc(uint dev)）`， 直接使用偏移量取， 根据偏移量来定位group，根据位图
判断哪些还没有分配！


大文件， 不实现
```
由于大文件会占据一大块的空间，访问后面的文件时会造成比较昂贵的seek操作。FFS会将文件的前面几个block放到文件所在的group，而将后面的数据分成比较大的chunk，保存在其它的group中。虽然这里可能造成一些额外的seek操作，但是由于文件比较大，会被数据的传输时间”掩盖”.
```