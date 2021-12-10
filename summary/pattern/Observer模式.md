### Observer模式

观察者模式；顾名思义，就是众多的观察者，通过一些技术手段监听着某个对象发出的通知；比如`iOSer`熟知的`KVO`

那么KVO就是通过添加观察者，观察对象的某个键值属性，当值发生变更时，观察者能够收到回调

在这里，我们简单举个例子

``` swift
protocol Observer {
    func notify(_ msg: String)
}


struct Subject {
    var observers: [Observer] = []
    
    mutating func add(_ o: Observer) {
        observers.append(o)
    }
    
    func notify(_ msg: String) {
        observers.forEach{ $0.notify(msg) }
    }
}


struct ConcreteObserver: Observer {
    
    var name: String
    
    func notify(_ msg: String) {
        print("观察者:\(name) 收到消息: \(msg)")
    }
}
```

测试一下

``` swift
var subject = Subject()
let obv1 = ConcreteObserver(name: "1")
subject.add(obv1)
let obv2 = ConcreteObserver(name: "2")
subject.add(obv2)
subject.notify("卖水果拉")
```

```
观察者:1 收到消息: 卖水果拉
观察者:2 收到消息: 卖水果拉
```