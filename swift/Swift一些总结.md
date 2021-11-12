#### Swift一些特性的总结

* 内存分配

  `struct`默认创建在栈区，分配内存更快；`class`则创建在堆区，由于堆区内存管理问题，则初始化时，比`struct`多出**额外的两个字段**：type，refCount。

  ```
  优化方式：
  对于频繁操作的内存，尽量使用struct，因为栈内存分配的更快，更安全，操作更快
  ```

  Tips：

  > 该说法仅仅是针对于基本数据类型的结构体来说，如果复杂的结构体，比如拥有`class`变量类型的结构体，可能会复制多份（多发生于将结构体当做参数传递时、拷贝结构体时）

* 派发方式

  派发方式分为三种：静态派发，函数表派发，消息机制派发；其中，静态派发速度是最快的

  ```
  静态派发：在编译器确定使用static dispatch后,会在生成的可执行文件内，直接指定包含了方法实现内存地址的指针。在运行时，直接通过指针调用特定的方法；
  
  内联展开：inline也叫内联展开，它可以人为声明，也可以通过编译器优化来实现。inline是将被调用方法的指针替换为方法实现体
  ```

  故：我们依据使用场景理应多选用`静态派发`的方式实现某个方法

* 函数表

  * V-Table：虚函数表；每种类型都会创建一张表，表内是一个包含了方法指针的数组

  ```
  class A {
    func foo()
    func bar()
    func bas()
  }
  
  sil @A_foo : $@convention(thin) (@owned A) -> ()
  sil @A_bar : $@convention(thin) (@owned A) -> ()
  sil @A_bas : $@convention(thin) (@owned A) -> ()
  
  sil_vtable A {
    #A.foo!1: @A_foo
    #A.bar!1: @A_bar
    #A.bas!1: @A_bas
  }
  
  class B : A {
    func bar()
  }
  
  sil @B_bar : $@convention(thin) (@owned B) -> ()
  
  sil_vtable B {
    #A.foo!1: @A_foo
    #A.bar!1: @B_bar
    #A.bas!1: @A_bas
  }
  
  class C : B {
    func bas()
  }
  
  sil @C_bas : $@convention(thin) (@owned C) -> ()
  
  sil_vtable C {
    #A.foo!1: @A_foo
    #A.bar!1: @B_bar
    #A.bas!1: @C_bas
  }
  ```

  * Protocol Witness Table：PWT；存储的是方法数组，里面包含了方法实现的指针地址，一般通过获取对象的内存地址和方法的位移`offset`去查找的
  * Value Witness Table：VWT；管理遵守了协议的`Protocol Type`实例的初始化，拷贝，内存消减和销毁的

* 写时复制（结构体）

  即优先使用内存指针。通过提高内存指针的使用，来降低堆区内存的初始化。降低内存消耗。在需要修改值的时候，会先检测引用计数检测，如果有大于1的引用计数，则开辟新内存，创建新的实例。在对内容进行变更的时候，会开启一块新的内存