# PHP7 内核分析：变量

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


## 变量的生命周期

### 变量的赋值和销毁

### 变量的作用域

### 全局变量

## 类型转换
