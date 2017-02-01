# PHP 类与对象

类是什么，什么是对象，相信不需要我在这里解释。本文也不是要说什么OO思想，而是想探究一个问题。作为 PHP 底层的实现 C 语言，是一个面向过程的语言，C 语言是如何构建出可以使用类与对象的 PHP（PHP 绝对称不上是面向对象的语言），这就是本文探讨的重点。为了方便查阅，也方便说明，以下所涉及的源码和实现均来源于 PHP7，后面所有出现的 PHP 均表示 PHP7 版本。

## 类

通常，我们概念中，类是一种抽象，类似于 C 语言中的结构体，本身称不上是一个实体。但在 PHP 的实现中，为了赋予不同的类特定的行为和数据结构，类不得不作为一个实体存在。

### 类的数据结构

在 PHP 的源码里，类是承载在 `zend_class_entry` 数据结构上的。

```
struct _zend_class_entry {
	char type;
	zend_string *name;
	struct _zend_class_entry *parent;
	int refcount;
	uint32_t ce_flags;

	int default_properties_count;
	int default_static_members_count;
	zval *default_properties_table;
	zval *default_static_members_table;
	zval *static_members_table;
	HashTable function_table;
	HashTable properties_info;
	HashTable constants_table;

	union _zend_function *constructor;
	union _zend_function *destructor;
	union _zend_function *clone;
	union _zend_function *__get;
	union _zend_function *__set;
	union _zend_function *__unset;
	union _zend_function *__isset;
	union _zend_function *__call;
	union _zend_function *__callstatic;
	union _zend_function *__tostring;
	union _zend_function *__debugInfo;
	union _zend_function *serialize_func;
	union _zend_function *unserialize_func;

	zend_class_iterator_funcs iterator_funcs;

	/* handlers */
	zend_object* (*create_object)(zend_class_entry *class_type);
	zend_object_iterator *(*get_iterator)(zend_class_entry *ce, zval *object, int by_ref);
	int (*interface_gets_implemented)(zend_class_entry *iface, zend_class_entry *class_type); /* a class implements this interface */
	union _zend_function *(*get_static_method)(zend_class_entry *ce, zend_string* method);

	/* serializer callbacks */
	int (*serialize)(zval *object, unsigned char **buffer, size_t *buf_len, zend_serialize_data *data);
	int (*unserialize)(zval *object, zend_class_entry *ce, const unsigned char *buf, size_t buf_len, zend_unserialize_data *data);

	uint32_t num_interfaces;
	uint32_t num_traits;
	zend_class_entry **interfaces;

	zend_class_entry **traits;
	zend_trait_alias **trait_aliases;
	zend_trait_precedence **trait_precedences;

	union {
		struct {
			zend_string *filename;
			uint32_t line_start;
			uint32_t line_end;
			zend_string *doc_comment;
		} user;
		struct {
			const struct _zend_function_entry *builtin_functions;
			struct _zend_module_entry *module;
		} internal;
	} info;
};
```

可以看到，类的属性繁多，这里我不会一一展开说明。可以看到的是，它会占用不小的内存，具体是多少呢？568字节，这其中还不包括类定义的属性和方法所占用的内存。

因此，PHP 类是一个相对比较重的实体，之所以设计的重，也是为了减少真正的对象实体所消耗的资源，毕竟相对于对象，定义的类的数量要少的多。

### 接口和类有什么区别

接口、traits、类本质上都是一样的，都是基于上文描述的 `zend_class_entry` 结构。接口的行为和类很相似，只是在使用上有一些限制。比如，不能定义属性，在编译的阶段特殊处理一下，直接禁止接口使用诸如 `static_members_table` 这样的字段即可。

因此，接口和 traits 的开销与类差不多，我们可以通过以下的方式来验证：

```
$class = <<<'CL'
interface Bar { }
CL;
$m = memory_get_usage();
eval($class);
echo memory_get_usage() - $m . "\n"; /* 912 bytes */
```

### 继承是如何实现的

继承提供了一种明确表述共性的方法，是一个新类从现有的类中派生的过程。 继承产生的新类继承了原始类的特性，新类称为原始类的派生类（或子类）， 而原始类称为新类的基类（或父类）。

对于 PHP 而言，实现继承的关键点就在于将父类的属性和方法复制给子类。这个过程被放在了编译阶段。具体实现方法可以查看 Zend/zend_inheritance.c 中的 `zend_do_inheritance()` 函数。

```
ZEND_API void zend_do_inheritance(zend_class_entry *ce, zend_class_entry *parent_ce) /* {{{ */
{
	zend_property_info *property_info;
	zend_function *func;
	zend_string *key;

	if (UNEXPECTED(ce->ce_flags & ZEND_ACC_INTERFACE)) {
		/* Interface can only inherit other interfaces */
		if (UNEXPECTED(!(parent_ce->ce_flags & ZEND_ACC_INTERFACE))) {
			zend_error_noreturn(E_COMPILE_ERROR, "Interface %s may not inherit from class (%s)", ZSTR_VAL(ce->name), ZSTR_VAL(parent_ce->name));
		}
	} else if (UNEXPECTED(parent_ce->ce_flags & (ZEND_ACC_INTERFACE|ZEND_ACC_TRAIT|ZEND_ACC_FINAL))) {
		/* Class declaration must not extend traits or interfaces */
		if (parent_ce->ce_flags & ZEND_ACC_INTERFACE) {
			zend_error_noreturn(E_COMPILE_ERROR, "Class %s cannot extend from interface %s", ZSTR_VAL(ce->name), ZSTR_VAL(parent_ce->name));
		} else if (parent_ce->ce_flags & ZEND_ACC_TRAIT) {
			zend_error_noreturn(E_COMPILE_ERROR, "Class %s cannot extend from trait %s", ZSTR_VAL(ce->name), ZSTR_VAL(parent_ce->name));
		}

		/* Class must not extend a final class */
		if (parent_ce->ce_flags & ZEND_ACC_FINAL) {
			zend_error_noreturn(E_COMPILE_ERROR, "Class %s may not inherit from final class (%s)", ZSTR_VAL(ce->name), ZSTR_VAL(parent_ce->name));
		}
	}
	
	...
}
```

### 多态是如何实现的

至于多态，是调用同一个方法，对于不同的对象能够产生不同的行为，往往会定义一个接口规范具体实现类需要实现的方法：

```
<?php
interface Animal {
    public function run();
}
 
class Dog implements Animal {
    public function run() {
        echo 'dog run';
    }
}
 
class Cat implements Animal{
    public function run() {
        echo 'cat run';
    }
}

class Context {
    private $_animal;
 
    public function __construct(Animal $animal) {
        $this->_animal = $animal;
    }
 
    public function run() {
        $this->_animal->run();
    }
}
```

在 PHP 的实现中，核心点在于传入方法的参数可以通过指定接口或者任意级的父类来限制。在 PHP 的设计中，这个语法称之为类型提示。相较于标准类型，对象的类型提示的判断需要额外处理一些逻辑，实现代码在 Zend/zend_execute.c 中：

```
static zend_always_inline int zend_verify_arg_type(zend_function *zf, uint32_t arg_num, zval *arg, zval *default_value, void **cache_slot)
{
	...

	if (UNEXPECTED(!zend_check_type(zf, cur_arg_info, arg, &ce, cache_slot, default_value, 0))) {
		zend_verify_arg_error(zf, cur_arg_info, arg_num, ce, arg);
		return 0;
	}

	return 1;
}

static zend_always_inline zend_bool zend_check_type(
		const zend_function *zf, const zend_arg_info *arg_info,
		zval *arg, zend_class_entry **ce, void **cache_slot,
		zval *default_value, zend_bool is_return_type)
{
	...

	ZVAL_DEREF(arg);
	if (EXPECTED(arg_info->type_hint == Z_TYPE_P(arg))) {
		if (arg_info->class_name) {
			if (EXPECTED(*cache_slot)) {
				*ce = (zend_class_entry *) *cache_slot;
			} else {
				*ce = zend_verify_arg_class_kind(arg_info);
				if (UNEXPECTED(!*ce)) {
					return 0;
				}
				*cache_slot = (void *) *ce;
			}
			if (UNEXPECTED(!instanceof_function(Z_OBJCE_P(arg), *ce))) {
				return 0;
			}
		}
		return 1;
	}

	...
}

```

可以看到，如果参数类型被定义为类或者接口，则调用 `zend_verify_arg_class_kind()` 函数获取对应的类数据，然后调用 `instanceof_function()` 函数。此函数首先会遍历实例所在类的所有接口，递归调用其本身，判断实例的接口是否为指定类的实例。

## 对象

谈到对象，其实与 PHP 的变量的设计息息相关。在 [PHP7 内核分析：变量的设计](http://joshuais.me/php7-nei-he-fen-xi-bian-liang-de-she-ji/) 一文中已经略有提及，由于篇幅原因，有些问题没有展开说明，那就在本篇里补全吧。

### 对象的数据结构

还是要把 PHP 对象的数据结构说明一下：

```
struct _zend_object {
    zend_refcounted_h gc;
    uint32_t          handle;
    zend_class_entry *ce;
    const zend_object_handlers *handlers;
    HashTable        *properties;
    zval              properties_table[1]; /* C struct hack */
};
```

这个结构体包含以下几个属性：

- `ce` 是指向上一章节我们提到的类数据的指针。
- `handle` 在 [PHP7 内核分析：变量的设计](http://joshuais.me/php7-nei-he-fen-xi-bian-liang-de-she-ji/) 的关于对象池的概念中已经说明了，内容比较多，不再次展开说明。
- `properties` 用于存储定义在类中的属性。
- `properties_table` 用于存储动态定义的属性。（这里还利用了 C 语言的 struct hack 的 trick）
- `handlers` 指向了标准对象处理函数 `std_object_handlers`。
- `gc` 记录了当前对象被引用的状态，用于垃圾回收机制处理。

### 对象是怎样创建的

创建对象的过程分成两个步骤：

```
<?php
class A {

}

$a = new A();
```

我们通过 VLD 可以看到，创建对象的操作产生了两个 OPCODE：

```
Finding entry points
Branch analysis from position: 0
Jump found. Position 1 = -2
filename:       /Users/joshua/Programs/local/test/php/opcode/object.php
function name:  (null)
number of ops:  5
compiled vars:  !0 = $a
line     #* E I O op                           fetch          ext  return  operands
-------------------------------------------------------------------------------------
   3     0  E >   NOP
   7     1        NEW                                              $2      :-6
         2        DO_FCALL                                      0
         3        ASSIGN                                                   !0, $2
         4      > RETURN                                                   1

branch: #  0; line:     3-    7; sop:     0; eop:     4; out1:  -2
path #1: 0,
Class A: [no user functions]
```

第一步，初始化对象变量。通过 `zend_fetch_class()` 获取存储类的变量，然后调用 `zend_objects_new()`（代码位于 Zend/zend_objects.c） 函数为对象分配内存空间。

```
ZEND_API zend_object *zend_objects_new(zend_class_entry *ce)
{
	zend_object *object = emalloc(sizeof(zend_object) + zend_object_properties_size(ce));

	zend_object_std_init(object, ce);
	object->handlers = &std_object_handlers;
	return object;
}
```

第二步，调用类定义的构造函数，其调用和其它的函数调用是一样。

### 对象方法调用的过程是怎样的

我们来看下对象在调用方法的时候的处理逻辑：

```
<?php

class A {

    public function do() {
        echo 'test';
    }
}

$a = new A();
$a->do();
```

同样使用 VLD 查看产生的 OPCODE：

```
Finding entry points
Branch analysis from position: 0
Jump found. Position 1 = -2
filename:       /Users/joshua/Programs/local/test/php/opcode/object.php
function name:  (null)
number of ops:  7
compiled vars:  !0 = $a
line     #* E I O op                           fetch          ext  return  operands
-------------------------------------------------------------------------------------
   3     0  E >   NOP
  10     1        NEW                                              $2      :-6
         2        DO_FCALL                                      0
         3        ASSIGN                                                   !0, $2
  11     4        INIT_METHOD_CALL                                         !0, 'do'
         5        DO_FCALL                                      0
         6      > RETURN                                                   1

branch: #  0; line:     3-   11; sop:     0; eop:     6; out1:  -2
path #1: 0,
Class A:
Function do:
Finding entry points

// 省略调用对象方法产生的 OPCODE
...
```

可以看到相对于调用函数，在调用对象方法之前，会执行 `ZEND_INIT_METHOD_CALL` 操作。这个操作主要为了处理 `__call()` 和 `__callStatic()` 魔术方法，以及验证方法的调用权限，以及是否为静态调用等。

对象的方法调用与非对象的方法调用基本无异，唯一的区别在于可以使用 `$this` 变量获取到当前所在对象。因此，在 `ZEND_INIT_METHOD_CALL` 操作中，最后初始化调用栈的时候会将当前对象传入。

```
static ZEND_OPCODE_HANDLER_RET ZEND_FASTCALL ZEND_INIT_METHOD_CALL_SPEC_CONST_CONST_HANDLER(ZEND_OPCODE_HANDLER_ARGS)
{
	...

	call = zend_vm_stack_push_call_frame(call_info,
		fbc, opline->extended_value, called_scope, obj);
	call->prev_execute_data = EX(call);
	EX(call) = call;

	ZEND_VM_NEXT_OPCODE();
}
```

最终在 Zend/zend_execute.h 文件的 `zend_vm_init_call_frame()` 的函数被设置到调用上下文数据中，供后续流程使用。

```
static zend_always_inline void zend_vm_init_call_frame(zend_execute_data *call, uint32_t call_info, zend_function *func, uint32_t num_args, zend_class_entry *called_scope, zend_object *object)
{
	call->func = func;
	if (object) {
		Z_OBJ(call->This) = object;
		ZEND_SET_CALL_INFO(call, 1, call_info);
	} else {
		Z_CE(call->This) = called_scope;
		ZEND_SET_CALL_INFO(call, 0, call_info);
	}
	ZEND_CALL_NUM_ARGS(call) = num_args;
}
```

## 结语

PHP 向我们展示了，如何用面向过程的语言实现一门支持类与对象的语言。其设计的思路和方法，大有我们学习和参考的价值，有兴趣的同学可以就感兴趣的问题再深入挖掘，也欢迎大家和我分享交流。

## 参考文献

http://jpauli.github.io/2015/03/24/zoom-on-php-objects.html

http://www.php-internals.com/book/?p=chapt05/05-04-class-inherit-abstract