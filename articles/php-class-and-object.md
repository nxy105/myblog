# PHP 类与对象

## PHP5

### 学习要点
- 类的数据结构是 `_zend_class_entry`
- 内部定义类和用户定义的类的区别在于，用户定义的类每个请求都需要创建/销毁类数据，但单次请求不需要的类就不会加载；内部定义的类是永驻内存的，不管你请求是否需要。 
- 接口（interface）、类（class）、trait 都是使用 `_zend_class_entry` 结构体承载。
- 类绑定：准备类和类相关的数据以便用户使用的过程。其中有个步骤，就是把类注册到类表中，以便使用类时可以查找到类的信息。（注：我理解是执行时定义类的过程。作者认为这个步骤有着不小的开销，所以开始时就建议不尽可能少包含不需要使用的类，而是用自动加载机制）。
- 类不但保持了静态变量，同时也保存了对象的动态变量，方便在没有修改的时候复用，以减少内存的消耗。
- 对象不是引用，即使他能像引用一样传递到方法中依然能够修改其属性。
- 为什么对象能够像引用一样使用，是因为对象除了new/clone/unserialize，其他操作都无法在对象池中创建对象，只能创建对象的外壳数据，其属性 `handle` 都指向对象池中的同一个对象。
- $this 是运行类方法时的动态变量，作为方法调用时的上下文的一个特定值存储。
- 无法区别 $this 和当前类产生的对象 $object，导致私有属性可能会被错误的赋值。（PHP7 是否如此？依然如此）
- 析构（destruct）函数调用顺序不稳定，受代码编写情况影响，并且在产生致命错误的时候，不会执行析构。因此，依赖析构的逻辑，最好自行销毁对象。

## PHP7

### 学习要点

- 移除了从对象池里读取对象的操作，对象池依然保留，只能写入和删除。

## 问题记录

- $this 如何实现

