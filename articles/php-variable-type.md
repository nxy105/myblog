# PHP7 内核分析：变量的设计

## 变量的结构

### 变量的数据结构

变量保存在 `zval` 的结构体中（与 PHP5 相同，但数据结构做了很大改变）。`zval` 结构体定义在 `Zend/zend_types.h` 文件中，结构体如下：

```
typedef struct _zval_struct     zval;

struct _zval_struct {
	zend_value        value;			/* value */
	union {
		struct {
			ZEND_ENDIAN_LOHI_4(
				zend_uchar    type,			/* active type */
				zend_uchar    type_flags,
				zend_uchar    const_flags,
				zend_uchar    reserved)	    /* call info for EX(This) */
		} v;
		uint32_t type_info;
	} u1;
	union {
		uint32_t     next;                 /* hash collision chain */
		uint32_t     cache_slot;           /* literal cache slot */
		uint32_t     lineno;               /* line number (for ast nodes) */
		uint32_t     num_args;             /* arguments number for EX(This) */
		uint32_t     fe_pos;               /* foreach position */
		uint32_t     fe_iter_idx;          /* foreach iterator index */
		uint32_t     access_flags;         /* class constant access flags */
		uint32_t     property_guard;       /* single property guard */
	} u2;
};
```

`zval` 的结构体中有3个属性：

- zend_value 保存变量的值。
- u1 保存变量的类型和是否为用户定义的常量等信息。
- u2 保存变量的额外信息（遇到具体场景再详细说明属性的含义）。

> 这里可以看到设计上与 PHP5 相比本质的区别在于减少了 `zval` 结构体所占的空间大小。我们来做一个简单的计算，`zend_value` 结构体占用内存为8个字节（详情在后续章节里展开说明），`u1` 占用了4个字节，`u2` 占用了4个字节，总共为8 + 4 + 4 = 16字节。相比较 PHP5 中变量需要占用48个字节，减少了有2/3之多。这样的优化，对于 PHP7 的效率提升起到了关键的作用。 


#### 变量的值

结构体中的 `zend_value` 属性保存了变量的值，同样在 `Zend/zend_types.h` 文件中定义：

```
typedef union _zend_value {
	zend_long         lval;				/* long value */
	double            dval;				/* double value */
	zend_refcounted  *counted;
	zend_string      *str;
	zend_array       *arr;
	zend_object      *obj;
	zend_resource    *res;
	zend_reference   *ref;
	zend_ast_ref     *ast;
	zval             *zv;
	void             *ptr;
	zend_class_entry *ce;
	zend_function    *func;
	struct {
		uint32_t w1;
		uint32_t w2;
	} ww;
} zend_value;
```

`zend_value` 是一个联合体，不同变量类型的值保存在不同的属性中。从联合体的定义可以看出，除了整型和双精度类型数据，其他的数据都以指针的方式存储在联合体中。这样做使得 `zend_value` 联合体的大小仅仅为8个字节。同时对于整型和双精度数据则不使用额外的指针，减少了这两种类型数据读取时的操作，算是平衡了存储和计算资源的一种设计。

#### 变量的类型

`zval.u1.v.type` 属性表示变量的类型，常规的数据类型定义如下：

```
#define IS_UNDEF					0
#define IS_NULL						1
#define IS_FALSE					2
#define IS_TRUE						3
#define IS_LONG						4
#define IS_DOUBLE					5
#define IS_STRING					6
#define IS_ARRAY					7
#define IS_OBJECT					8
#define IS_RESOURCE					9
#define IS_REFERENCE				10

/* constant expressions */
#define IS_CONSTANT					11
#define IS_CONSTANT_AST				12

/* fake types */
#define _IS_BOOL					13
#define IS_CALLABLE					14
#define IS_ITERABLE					19
#define IS_VOID						18

/* internal types */
#define IS_INDIRECT             	15
#define IS_PTR						17
#define _IS_ERROR					20
```

通过 `Z_TYPE` 宏我们可以能容易的获取到变量的类型。

```
static zend_always_inline zend_uchar zval_get_type(const zval* pz) {
	return pz->u1.v.type;
}
/* we should never set just Z_TYPE, we should set Z_TYPE_INFO */
#define Z_TYPE(zval)				zval_get_type(&(zval))
```

这里建议设置变量类型时使用 `Z_TYPE_INFO`，我们先看下 `Z_TYPE_INFO` 的定义：

```
#define Z_TYPE_INFO(zval)			(zval).u1.type_info
#define Z_TYPE_INFO_P(zval_p)		Z_TYPE_INFO(*(zval_p))

#define ZVAL_UNDEF(z) do {				\
		Z_TYPE_INFO_P(z) = IS_UNDEF;	\
	} while (0)
```

`Z_TYPE_INFO` 表示 `zval.ui.type_info`。这样设置类型的时候，不需要对它的字段分别赋值，而是可以统一赋值：

```
zval1.u1.type_info = zval2.u1.type_info
```

就相当于如下序列赋值：

```
zval1.u1.v.type = zval2.u1.v.type
zval1.u1.v.type_flags = zval2.u1.v.type_flags
zval1.u1.v.const_flags = zval2.u1.v.const_flags
zval1.u1.v.reserved = zval2.u1.v.reserved
```

定义 `zval.u1` 需要 `ZEND_ENDIAN_LOHI_4` 宏，目的是保证在大端或小端的机器上，定义的字段都按照一样的顺序排列，保证了上面统一赋值的设计的可移植型。 

## 变量的类型

前面，我们介绍了变量的类型的数据结构，接下来，我们来看看一些特殊的变量类型的设计。

### 引用类型

相较于 PHP5，引用类型的变量设计方式进行了比较大的改动。

通常的变量，根据写时拷贝的原则，当变量发生改变之前，我们会将指向相同值的变量拷贝分离。但对于引用类型来说，这样做肯定是不正确的，因为我们希望引用同一个值的变量同时发生改变。在 PHP5 的内核中，通过 `is_ref` 标志位表示一个变量是否是一个引用类型，通过下面的例子，我们可以看到引用在 PHP5 中是如何工作的：

```
$a = [];  // $a     -> zval_1(type=IS_ARRAY, refcount=1, is_ref=0) -> HashTable_1(value=[])
$b =& $a; // $a, $b -> zval_1(type=IS_ARRAY, refcount=2, is_ref=1) -> HashTable_1(value=[])

$b[] = 1; // $a = $b = zval_1(type=IS_ARRAY, refcount=2, is_ref=1) -> HashTable_1(value=[1])
```

这样设计最大的问题在于，没有办法把同一个数据在引用和非引用的变量上共享。

```
$a = [];  // $a         -> zval_1(type=IS_ARRAY, refcount=1, is_ref=0) -> HashTable_1(value=[])
$b = $a;  // $a, $b     -> zval_1(type=IS_ARRAY, refcount=2, is_ref=0) -> HashTable_1(value=[])
$c = $b   // $a, $b, $c -> zval_1(type=IS_ARRAY, refcount=3, is_ref=0) -> HashTable_1(value=[])

$d =& $c; // $a, $b -> zval_1(type=IS_ARRAY, refcount=2, is_ref=0) -> HashTable_1(value=[])
          // $c, $d -> zval_1(type=IS_ARRAY, refcount=2, is_ref=1) -> HashTable_2(value=[])
          // 因为 is_ref 不同，这个时候虽然 $c/$d 的值还是和 $a/$b 相同，但我们不得不拷贝一份 HashTable_2
```

到了 PHP7 时代，引用有了自己独立的类型和数据结构：

```
struct _zend_reference {
	zend_refcounted_h gc;
	zval              val;
};
```

之前的简单例子在 PHP7 下是这样工作的：

```
$a = [];  // $a                                     -> zend_array_1(refcount=1, value=[])
$b =& $a; // $a, $b -> zend_reference_1(refcount=2) -> zend_array_1(refcount=1, value=[])

$b[] = 1; // $a, $b -> zend_reference_1(refcount=2) -> zend_array_1(refcount=1, value=[1])
```

我们再来看看同一个数据在引用和非引用变量之间是如何做到共享的：

```
$a = [];  // $a         -> zend_array_1(refcount=1, value=[])
$b = $a;  // $a, $b,    -> zend_array_1(refcount=2, value=[])
$c = $b   // $a, $b, $c -> zend_array_1(refcount=3, value=[])

$d =& $c; // $a, $b                                 -> zend_array_1(refcount=3, value=[])
          // $c, $d -> zend_reference_1(refcount=2) ---^
          // 创建引用变量的时候，不需要再复制原始的数据了，只需要创建一个引用类型数据并且指向原始数据就可以了

$d[] = 1; // $a, $b                                 -> zend_array_1(refcount=2, value=[])
          // $c, $d -> zend_reference_1(refcount=2) -> zend_array_2(refcount=1, value=[1])
          // 只有值发生改变时，才拷贝一份新数据，符合写时拷贝的原则
```

总结一下，PHP7 相较之前的设计，最大的收益在于可以在引用和非引用的变量之间共享真实的数据，从而真正做到了写时拷贝。

### 数组类型

PHP 的数据是通过 HashTable 实现，关于 PHP7 的 HashTable 的设计，我已经在一篇译文里 [[译] PHP7 数组：HashTable](http://joshuais.me/yi-php7-shu-zu-hashtable/) 里详细描述了，就不在此展开说明了。

### 对象类型

我们还是首先看下在 PHP5 时代，对象是如何实现的。那时，对象被存储在 `_zend_object_value` 结构体中：

```
typedef struct _zend_object_value {
    zend_object_handle handle;
    const zend_object_handlers *handlers;
} zend_object_value;
```

其中，`handle` 是一个可以在全局的对象池里索引到指定对象的唯一标识。`handlers` 是个包含了多个指针函数的结构体，这些指针函数包含对对象属性的操作。

然而，对象真正的数据并没有直接保存在该结构体里，而需要通过全局的对象池中索引到。对象真正的数据存储在 `_zend_object` 结构体中：

```
typedef struct _zend_object {
    zend_class_entry *ce;
    HashTable *properties;
    zval **properties_table;
    HashTable *guards;
} zend_object;
```

其中，`ce` 是对象所对应的类的结构，`properties` 和 `properties_table` 都是存放对象的属性，只是访问的方式不同。

之前说到对象被存储在对象池中，通过 `EG(objects_store)` 宏访问。对象池的作用是缓存生成的对象，方便对象的复用，节省内存的开销。对象池的存储结构为 `zend_objects_store` 结构体，如下：

```
typedef struct _zend_objects_store {
    zend_object_store_bucket *object_buckets;
    zend_uint top;
    zend_uint size;
    int free_list_head;
} zend_objects_store;
 
typedef struct _zend_object_store_bucket {
    zend_bool destructor_called;
    zend_bool valid;
    union _store_bucket {
        struct _store_object {
            void *object;
            zend_objects_store_dtor_t dtor;
            zend_objects_free_object_storage_t free_storage;
            zend_objects_store_clone_t clone;
            const zend_object_handlers *handlers;
            zend_uint refcount;
            gc_root_buffer *buffered;
        } obj;
        struct {
            int next;
        } free_list;
    } bucket;
} zend_object_store_bucket;
```

这个结构体包含了很多信息。其中，`_store_object` 中的 `object` 属性就是指向真正对象的指针。

PHP7 的设计，致力于减少重复的引用计数，减少内存使用和减少间接的访问。基于以上思想，`zend_object` 的设计如下：

```
struct _zend_object {
    zend_refcounted   gc;
    uint32_t          handle;
    zend_class_entry *ce;
    const zend_object_handlers *handlers;
    HashTable        *properties;
    zval              properties_table[1];
};
```

可以看到，对象不再需要 `_zend_object_value` 结构体作为访问的外壳了，减少了对内存多次访问。同时，对象池被“架空”了（这里的架空指的是对象池依然存在，但并不在访问对象时使用）。

这里说明两个问题。

第一，为什么对象池没有被移除而是保留下来。因为，在一次请求处理完成后，PHP 会进行 SHUTDOWN 的处理流程，这时，我们需要对所有本次请求中创建但没有主动注销的对象执行析构处理。这时，对象池就提供了一个方便处理的数据集合。

第二，`_zend_object` 为什么还保留了可以索引到对象池的 `handle`。因为对象池依然被使用，创建对象时需要在对象池中创建对象，同时，主动注销对象当然也需要通过 `handle` 找到并维护对象池中的数据。

总结一下，PHP7 相较之前的设计，减少了重复的引用计数（refcount），并且内存使用更小了（只需要40字节用于存储对象和对象的每个属性16字节的容量），取消了访问对象池，将之前需要4次内存的访问，减少到可以1次内存直接获取到的对象。

## 结束语

从开题到完成这篇文章，其中断断续续持续了有一个多月的时间。除了最近有些懈怠之外，最大的原因莫过于开始的时候把题目定的过于庞大。

PHP 的变量几乎贯穿于整个 PHP 内核的设计，从变量的数据结构设计，到变量的生命周期，到变量的类型转换，几乎每项内容可以总结出一两篇文字。开篇的时候没有把主旨和结构设计好，导致写的过程感觉不是思路断片，就是整体产生不平衡的感觉。直到这两天，重新整理了一遍思路，才决定只保留变量的数据结构和特殊的类型变量的基础设计内容，旨在将内核中的变量以什么样的方式存在这件事讲明白。至于和变量相关其他的内容，还是放在特定的命题下展开说明比较合适。

另外，上一篇总结 HashTable 的文章，是翻译了国外优秀的文章。虽然，译文也需要思考和考证，也是收获颇丰，但终究不是按照自己的思路探究事物，少了很多自己的思考。所以，这次先确定了架构和问题，然后找寻答案，再把答案消化后转换为文字总结于此。如果其中我的理解有所偏差，欢迎发现问题的你留言交流。

最后的最后，能把这篇文字写完的感觉，就像雾霾过后迎来久别的晴朗。回顾了一下，觉得写得还是极其简陋（笑）。ANYWAY，人生需要里程碑，完成不代表结束，而是一次新的迭代的开始。

## 参考文献

[Internal value representation in PHP 7 - Part 1](http://nikic.github.io/2015/05/05/Internal-value-representation-in-PHP-7-part-1.html)

[Internal value representation in PHP 7 - Part 2](http://nikic.github.io/2015/06/19/Internal-value-representation-in-PHP-7-part-2.html)

[深入理解PHP7之zval](https://github.com/laruence/php7-internal/blob/master/zval.md)

[变量的结构和类型](http://www.php-internals.com/book/?p=chapt03/03-01-00-variables-structure)

