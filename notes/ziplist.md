# ziplist

ziplist为了压缩存储一个不算大的链表数据，其结构如下：

```
<zlbytes> <zltail> <zllen> <entry> <entry> ... <entry> <zlend>
```

其中：

1.  （uint32_t）zlbytes：最开始的一段uint32类型整数存放整个链表的大小。
2.  （uint32_t）zltail：存放链表最后的一个entry的偏移量，这是为了方便从后往前遍历链表之用。
3.  （uint16_t）zllen：entry的数量。从这个类型可知，zip链表元素数量不大于2^16 - 1。
4.  （uint8_t）zlend：magic number 255，用于标记链表的终止位置。

## entry格式

```
<prevlen> <encoding> <entry-data>
```

其中：
1.  prelen：用于存放上一个entry的大小，这也是为了从后往前遍历链表之用的。如果上一个entry大小小于254byte，那么prevlen只用一个字节就能表示其大小；否则，prevlen就使用5个字节来表示长度，其中第一个byte是254（0xFE），剩余的四个字节用于存储真实的长度。
2.  encoding域：用于存储不同类型的表示字符串或者整数的格式。

如果是用于表示字符串，那么最开始的2 bit存储encoding的类型，紧跟着才是字符串的真实长度：

    a)  |00pppppp|：1个字节。用于表示字符串长度小于等于63的字符串。
    b)  其他类型见代码注释，略。



