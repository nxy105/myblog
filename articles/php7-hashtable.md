# PHP7 数组：HashTable

## 简介

几乎每个C程序中都会使用到哈希表。鉴于C语言只允许使用整理作为数组的键名，PHP 创建了哈希表，将字符串的键名哈希映射到大小有限的数组中。这样无法避免的会产生碰撞，因此，PHP 使用了链表解决这个问题。

众多哈希表的实现方式，无一完美。每种设计都着眼于某一个侧重点，有的优化了 CPU，有的优化了内存，有的则是支持线程级的扩展。

> There exists many ways to design HashTables, depending on what factor you want to promote.

## HashTable 数据结构

开始介绍之前，我们需要事先声明一些事情：

- 哈希表的键名可能是字符串或者是整数。当是字符串时，我们声明类型为`zend_string`；当是整数时，声明为`zend_ulong`。
- 哈希表的顺序遵循表内元素的插入顺序。
- 哈希表的容量是自动伸缩的。
- 在内部，哈希表的容量总是2的倍数。
- 哈希表中每个元素一定是`zval`类型的数据。

以下是 HashTable 的结构体：

```
struct _zend_array {
	zend_refcounted_h gc;
	union {
		struct {
			ZEND_ENDIAN_LOHI_4(
				zend_uchar    flags,
				zend_uchar    nApplyCount,
				zend_uchar    nIteratorsCount,
				zend_uchar    reserve)
		} v;
		uint32_t flags;
	} u;
	uint32_t          nTableMask;
	Bucket           *arData;
	uint32_t          nNumUsed;
	uint32_t          nNumOfElements;
	uint32_t          nTableSize;
	uint32_t          nInternalPointer;
	zend_long         nNextFreeElement;
	dtor_func_t       pDestructor;
};
```

这个结构体占56个字节。

其中最重要的字段是`arData`，它是一个指向`Bucket`类型数据的指针，`Bucket`结构定义如下：

```
typedef struct _Bucket {
	zval              val;
	zend_ulong        h;                /* hash value (or numeric index)   */
	zend_string      *key;              /* string key or NULL for numerics */
} Bucket;
```

`Bucket` 中不再使用指针指向一个 `zval` 类型的变量，而是直接使用变量本身。因为在 PHP7 中，`zval` 不再使用堆分配内存。（字面含义，我理解是 `zval` 不再有时存储变量本身，而是一个指向变量值得指针，使得 `zval` 大小控制在比较小的范围）。

![alt](http://jpauli.github.io//img/php7-hashtables/simple_hash.png)

我们注意到所有的Bucket都是按顺序存放的。

### 插入元素

PHP 会保证插入数组的元素保持插入的顺序。这样当使用 `foreach` 循环数组时，能够按照插入的顺序遍历。假设我们有这样的数组：

```
$a = [9 => "foo", 2 => 42, []];
var_dump($a);

array(3) {
    [9]=>
    string(3) "foo"
    [2]=>
    int(42)
    [10]=>
    array(0) {
    }
}
```

所有的数据在内存上都是相邻的。

![alt](http://jpauli.github.io//img/php7-hashtables/simple_hash_data_1.png)

这样做，处理哈希表的迭代器的逻辑就变得相当简单。只需要直接遍历 `arData` 数组即可。遍历内存中相邻的数据，将会极大的利用 CPU 缓存。因为 CPU 缓存能够读取到整个 `arData` 的数据，访问每个元素将在微妙级。

```
size_t i;
Bucket p;
zval val;

for (i=0; i < ht->nTableSize; i++) {
    p   = ht->arData[i];
    val = p.val;
    /* do something with val */
}
```

如你所见，数据被顺序存放到 `arData` 中。为了实现这样的结构，我们需要知道下一个可用的节点的位置。这个位置保存在数组结构体中的 `nNumUsed` 字段中。

每当添加一个新的数据时，我们保存后，会执行 `ht->nNumUsed++`。当 `nNumUsed` 值到达哈希表所有元素的最大值（`nNumOfElements`）时，会触发“压缩或者扩容”的算法。

以下是向哈希表插入元素的简单实现示例：

```
idx = ht->nNumUsed++; /* take the next avalaible slot number */
ht->nNumOfElements++; /* increment number of elements */
/* ... */
p = ht->arData + idx; /* Get the bucket in that slot from arData */
p->key = key; /* Affect it the key we want to insert at */
/* ... */
p->h = h = ZSTR_H(key); /* save the hash of the current key into the bucket */
ZVAL_COPY_VALUE(&p->val, pData); /* Copy the value into the bucket's value : add operation */
```
我们可以看到，插入时只会在 `arData` 数组的结尾插入，而不会填充已经被删除的节点。

### 删除元素


