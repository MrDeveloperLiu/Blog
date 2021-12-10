### Command模式

如果我们有一个类，用来表示，请进行某个工作；那么每一个工作就不再是方法的调用，而是表示命令的类的实例；

有时命令也被称之为事件，类似于事件，被放置队列中，依次去处理这些命令；


``` swift


protocol Receiver {
    func action()
}

protocol Command {
    var receiver: Receiver {get}
}

extension Command {
    func execute() {
        receiver.action()
    }
}

struct ConcreteReceiver: Receiver {
    func action() {
        print("执行操作")
    }
}

struct ConcreteCommand: Command {
    var receiver: Receiver
}

struct Invoker {
    
    func exe(_ c: Command) {
        c.execute()
    }
}


``` swift
let invoker = Invoker()
let receiver = ConcreteReceiver()
let cmd = ConcreteCommand(receiver: receiver)
invoker.exe(cmd)
``` 

```
执行操作
```

> 那么使用场景是什么呢？


试想一下，当点击某些不同的按钮的时候，我们需要执行不同的操作，假如就是一个简单的接口请求+Toast，那么用命令模式会非常方便，不同的命令，拥有不同的接收者，接收者去发送请求。

调用者通过调用不同的命令，完成不同的操作，这样点击事件就变成了

```
new Command

execute()
```