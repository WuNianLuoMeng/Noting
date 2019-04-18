# redis-数据结构与对象

### *简单动态字符串

+ SDS(简单动态字符串)

+ SDS的定义

  ~~~ c++
  struct sdshdr{    
      int len;//记录buf数组中已使用字节的数量=SDS中所保存的字符串的长度
      int free;//记录buf数组中未使用的字节的数量
      char buf[];//字符数组。用于保存字符串，最后的一个字节为'\0'
  }
  ~~~

+ **SDS与C的区别**

  + **常数复杂度获取字符串的长度**

    对于c语言的字符串数组来说，每次获取数组的长度是O(n)，然而对于SDS来讲，它的结构体中存储了len这个变量，设置和更新SDS长度的工作是由SDS的API在执行的时候自动完成更新的，不需要手工修改长度的工作，因此对于SDS来说，获取字符串的长度的时间复杂度就变成O(1).

  + **杜绝缓冲区溢出**

    - 对于c语言来说，如果给字符串S分配的空间资源小于字符串S最终拼接之后的空间大小的话，那么就会造成缓冲区的溢出。例子：两个字符串相邻排列，但对第一个字符串进行拼接字符串的时候，那么就会影响到第二个字符串。

    - 当SDSAPI需要对SDS进行修改的时候，API会先检查SDS的空间是否够用，如果不满足的话，API会自动将SDS的空间扩展至执行所需要的空间大小，然后才会去执行字符串的连接。

  + **减少修改字符串时带来的内存重分配次数**

    + 当c语言的字符串增加或者删减的时候，程序都要对更新过后的字符串进行内存的重新分配。
    + SDS操作
      + 空间预分配：用于优化SDS的字符串的增长操作，当SDS的API对一个SDS进行操作的时候，并且需要对SDS进行空间扩展的时候，程序不仅会为SDS分配修改所需要的空间，还会为SDS分配额外未使用的空间。通过这种空间预分配的策略，可以减少执行字符串增长操作所需要的的内存重新分配的次数。也就是说通过每次扩容时，多分配一些空间来减少扩容的次数。
      + 惰性空间释放：用于优化SDS的字符串缩短操作，当SDS的API要对SDS进行缩短的操作时，程序并不会立即收回缩短后多余出来的字节，而是使用<font color="red">free属性</font>保存起来，并等待下次使用。通过这种策略主要是为了防止缩短过后的操作时连接操作，这样一来，就不需要申请空间了。

  + **二进制安全**

    - c语言在存储字符串的时候，遇见空格就会停止读取，所以只能保存文本文件，而不能保存图片，音频，视频，压缩文件这样的二进制数据。
    - SDS的API都是二进制安全的，所有的SDSAPI都会以处理二进制的方式来处理SDS存放的buf数组中的数据。

  + SDS兼容了C字符串的函数

    + SDS中buf字符串的末尾都会以‘\0’结尾，这是为了可以重用一部分<string.h>库中定义的函数，比如：strcat等函数。

### *链表

链表节点的结构

~~~ c++
typedef struct listNode{
    //前置节点
    struct listNode *prev;
    //后置节点
    struct listNode *next;
    //节点的值
    void *value;
}
~~~



list用来持有链表，操作更加方便一些

~~~ c++
typedef struct list{
    //表头结点
    listNode *head;
    //表尾节点
    listNode *tail;
    //链表所包含的节点数量
    unsinged long len;
    //节点值赋值函数
    void *(*dup)(void *ptr);
    //节点值释放函数
    void (*free)(void *ptr);
    //节点值对比函数
    int (*match) (void *ptr,void *key);
}
~~~



### *字典

+ Redis的字典使用哈希表作为底层实现，一个哈希表里面可以有多个哈希表节点，而每个哈希表节点就保存了字典中的一个键值对。

+ 字典的实现

  ```c++
  typedef struct dict{
      //类型特定函数
      dictType *type;
      //私有数据
      void *privdata;
      //哈希表
      dictht ht[2];
      //rehash索引，当rehash不再进行的时候，值为-1
      int trehashidx;    
  }dict;
  ```

  ht属性是一个包含两个项的数组，数组中的每一项都是dictht哈希表一般情况下，字典只使用ht[0]哈希表，h[1]的作用是用来扩容时的临时的哈希表。

+ 哈希表

  ~~~ c++
  typedef struct dictht{
      //哈希表数组
      dictEntry **table;
      //哈希表大小
      unsigned long size;
      //哈希表掩码，用于扩容的时候sizemask=len-1；
      unsigned long sizemask;
      //该哈希表已经使用的节点数量
      unsigned long used;
  }dicht;
  ~~~

+ 哈希表节点

  ~~~ c++
  typedef struct dictEntry{
      //键
      void *key;
      //值
      union{
          void *val;
          uint64_t u64;
          int64_t s64;
      } v;
      //指向下个哈希表节点，形成链表，用来解决冲突
      struct dictEntry *next;
  }dictEntry;
  ~~~

  解决哈希冲突的方法是<font color="red">链地址法</font>,即头插法。

+ rehash操作

  + 为字典的ht[1]哈希表分配空间，这个哈希表的空间大小要取决于所执行的操作
    + 如果执行的是扩展操作，那么ht[1]的大小为第一个大于等于<font color="red">ht[0].used*2的2^n</font>。
    + 如果执行的是收缩操作，那么ht[1]的大小为第一个大于等于<font color="red">ht[0].used的2^n</font>。
  + 将保存在ht[0]中的所有键值对rehash到ht[1]上面。
  + 当ht[0]中的所有键值对都rehash到ht[1]上之后，释放ht[0],将ht[1]设置成ht[0],并在ht[1]处创建一个空白哈希表，为下一次rehash做准备。

+ rehash的情况

  + 服务器目前没有执行BGSAVE命令或者BGREWRITEAOF命令，并且哈希表的负载因子大于等于1

  + 服务器目前正在执行BGSAVE命令或者BGREWRITEAOF命令，并且哈希表的负载因子大于等于5

    负载因子：哈希表已保存的节点数量/哈希表的大小(ht[0].size)