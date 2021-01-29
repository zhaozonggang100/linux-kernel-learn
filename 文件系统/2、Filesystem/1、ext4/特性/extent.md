### ref
```
https://blog.csdn.net/sara4321/article/details/8609610（通过inode号查找文件数据所在块组以及块）
https://blog.csdn.net/sara4321/article/details/8610135
https://www.xuebuyuan.com/1470659.html
```
### 1、概要    

- 1、ext3、ext2是使用的`间接块`的方式存储大文件的数据，最多三级间接块，最大能存储4T大小的文件  

- 2、ext4支持两种数据管理方式

    1、inline  

    将数据存储在inode节点内部

    2、extent  

    将文件数据结构组织成一颗B树

    3、同时兼容ext2、ext3的间接块方式  

### 2、extent的意义  

&emsp;&emsp;当一个文件特别大的时候，如果还使用ext2、ext3的多级inode映射block方式，每个存储数据的bloc在创建或删除文件时都要统计，这样效率非常低，extent是多个地址连续的block的集合，超大的文件会分解在多个extent中，**extent减少了内存碎片**。

### 3、extent在磁盘上如何组织  

&emsp;&emsp;首先，extent是以B树来组织block的，extent tree中每一个节点都是一个block，对于存放extent tree节点信息的block，我们称为extent block，**root**节点存在在inode表中，所以root节点就不用extent block来描述了。  
&emsp;&emsp;每个节点（block）在其偏移0的位置存放数据结构**ext4_extent_header**用于描述该节点。ext4_extent_header的大小为**12字节**，定义如下：
```c
/*              
 * Each block (leaves and indexes), even inode-stored has header.
 */     
struct ext4_extent_header {
        // ext4 extent的标识0xF30A
        __le16  eh_magic;       /* probably will support different formats */
        // 当前节点中有效entries的数目 
        __le16  eh_entries;     /* number of valid entries */
        // 当前节点中entry的最大数目
        __le16  eh_max;         /* capacity of store in entries */
        // 当前节点在树中的深度
        __le16  eh_depth;       /* has tree real underlying blocks? */
        __le32  eh_generation;  /* generation of the tree */
};

/*
    tips ：
        1、叶子节点深度为0，指向数据extent
        2、tree中最大深度的节点是root节点
        3、深度最大为5（logic block number最大为2^32），满足4*(((blocksize - 12)/12)^n) >= 2^32 条件的最小n是5
*/
```
&emsp;&emsp;节点中存放的**ext4_extent_header**数据之后都是很多**entry**(即表项)，每个entry大小为**12**bytes。如果是非叶子节点(所谓非叶子节点，即ext4_extent_header->eh_depth > 0)，每个entry中存放是index数据，由**struct ext4_extent_idx**描述，每一个entry索引都指向一个**extent block**；如果是叶子节点(所谓叶子节点，即ext4_extent_header->eh_depth = 0)，每个entry都是指向一个extent，由**struct ext4_extent**描述。  
```c
/*
 * ext4_inode has i_block array (60 bytes total).
 * The first 12 bytes store ext4_extent_header;
 * the remainder stores an array of ext4_extent.
 * For non-inode extent blocks, ext4_extent_tail
 * follows the array.
 */

/*
 * This is the extent tail on-disk structure.
 * All other extent structures are 12 bytes long.  It turns out that
 * block_size % 12 >= 4 for at least all powers of 2 greater than 512, which
 * covers all valid ext4 block sizes.  Therefore, this tail structure can be
 * crammed into the end of the block without having to rebalance the tree.
 */
// 存放校验值
struct ext4_extent_tail {
        __le32  et_checksum;    /* crc32c(uuid+inum+extent_block) */
};

/*
 * This is the extent on-disk structure.
 * It's used at the bottom of the tree.
 */
// ext4_extent是用于表示extent的数据结构
struct ext4_extent {
        // extent的起始block地址
        __le32  ee_block;       /* first logical block extent covers */
        // extent的长度
        __le16  ee_len;         /* number of blocks covered by extent */
        // extent起始block的物理地址的高16位
        __le16  ee_start_hi;    /* high 16 bits of physical block */
        // extent起始block的物理地址的低32位
        __le32  ee_start_lo;    /* low 32 bits of physical block */
};
/*
    tips :
        1、关于extent的成员变量ee_len，需要说明的是：如果该值<=32768,那么这个extent已经初始化的。如果该值>32768,这个extent还没有初始化，并且extent长度是ee_len-32768.因此一个初始化的extent，其最大长度是32768，而一个未初始化的extent，最大长度是32767。
        2、前述已经介绍过checksums。Header和extent entries并未占用整个block。因为2^12 % 12 >= 4，因此block中至少还剩余4bytes的未用空间。那么32位的校验值可以存放在该空间中。当然inode中可能表示的4个extents并不需要校验，因为inode已经进行了校验。（为什么这么解释，原因是，除根节点外的所有的extent block都进行了校验）。
*/

/*
 * This is index on-disk structure.
 * It's used at all the levels except the bottom.
 */
 // ext4_extent_idx是用于描述extent block的一个结构体，通过该结构体，可以寻找到下一级的节点。
struct ext4_extent_idx {
        // 索引所覆盖的文件范围的起始block
        __le32  ei_block;       /* index covers logical blocks from 'block' */
        // 下一级extent block的逻辑地址的低32位
        __le32  ei_leaf_lo;     /* pointer to the physical block of the next *
                                 * level. leaf or next index could be there */
        // 下一级extent block的逻辑地址的高16位
        __le16  ei_leaf_hi;     /* high 16 bits of physical block */
        __u16   ei_unused;
};
```

- 下面赋一个**extent block**的layout  

<table border="1">
<tr>
<td bgcolor="lightblue" width="150"><center>header</center></td>
<td bgcolor="lightyellow" width="50"><center>entry</center></td>
<td bgcolor="lightyellow" width="50"><center>...</center></td>
<td bgcolor="lightyellow" width="50"><center>entry</center></td>
<td bgcolor="#00FFFF" width="30">tail</td>
</tr>
</table>