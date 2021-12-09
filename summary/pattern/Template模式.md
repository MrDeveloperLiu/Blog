#### Template模式

其实模板模式有些些类似于面向对象的多态思想：将具体的处理交给子类
* 父类实现了一些具体逻辑之后，暴露给子类一些抽象方法
* 让子类去实现抽象方法
像这样**在父类中定义处理流程的框架，在子类中实现具体处理的模式**就称之为Template Method


``` swift
open class AbstractDisplay {
    
    open func display() {
        fatalError("请使用子类实现display()")
    }
    
    private func head() {
        print("****BEGIN****")
    }
    private func trail() {
        print("****END****")
    }
    
    final func show() {
        head()
        display()
        trail()
    }
}

class StringDisplay: AbstractDisplay {
    override func display() {
        print("展示的是String(字符串)")
    }
}
```

下面我们来测试一下

``` swift
let d = StringDisplay()
d.show()
```

那么当然，你如果直接初始化一个抽象类`AbstractDisplay`并调用`show()`时，会引起一个运行时错误，它将提醒我们，需要用子类去实现方法后方能使用