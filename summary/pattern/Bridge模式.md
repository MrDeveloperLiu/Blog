### Bridge模式

桥接模式的作用是将类的功能层次结构和类的实现层次结构之间搭建桥梁
* 类的功能层次结构

> 假设有一个类Class当我们想为其扩充新的功能时，通常会创建一个子类SubClass，在子类中为其扩充功能；像这种SubClass: Class的结构，称之为类的层次结构
>   * 父类具有基本的功能
>   * 在子类中添加新的功能

这种层次结构被称之为类的功能层次结构 

* 类的实现层次结构

> 假设有一个抽象父类与一个子类，ConcreteClass: AbstractClass。通过在抽象父类中添加抽象方法，让子类来实现具体方法来增加新的功能
>   * 父类通过声明抽象方法来定义接口
>   * 子类通过实现具体方法来实现接口

这种层次结构被称为类的实现层次结构


``` swift
protocol Display {
    func open()
    func close()
    func print()
}

extension Display {
    func display() {
        open()
        print()
        close()
    }
}

protocol DisplayImpl {
    func rawOpen()
    func rawClose()
    func rawPrint()
}


struct ConcreteDisplay: Display {
    var impl: DisplayImpl
    
    init(_ impl: DisplayImpl) {
        self.impl = impl
    }
    
    func open() {
        impl.rawOpen()
    }
    func close() {
        impl.rawClose()
    }
    func print() {
        impl.rawPrint()
    }
}

struct ConcreteDisplayImpl: DisplayImpl {
    func rawOpen() {
        print("open file")
    }
    func rawClose() {
        print("close file")
    }
    func rawPrint() {
        print("读取文档")
    }
}
```

来测试一下

``` swift 
let impl = ConcreteDisplayImpl()
let d = ConcreteDisplay(impl)
d.display()
```

```
open file
读取文档
close file
```