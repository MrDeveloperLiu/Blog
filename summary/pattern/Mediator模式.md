### Mediator模式

* 一方面：当麻烦的事情发生时，通知仲裁者；当发生设计全体组员的事情时，页通知仲裁者。当仲裁者下达指令时，组员会立即执行；
* 另一方面：仲裁者站在整个团队的角度上对组员上报的事情做出决定

我们模拟这样一个场景：
拍卖场与顾客；那么拍卖场上的主持人相当于一个仲裁者，顾客就是在座的所有的喊价的人员

``` swift 
protocol Mediator {
    func createColleagues(_ c: [Colleague])
    func colleagueSent(_ msg: String)
}

protocol Colleague {
    func setMediator(_ m: Mediator)
    func receiveReport(_ msg: String)
}

class ConcreteMediator: Mediator {
    var colleagues: [Colleague]?
    
    func createColleagues(_ c: [Colleague]) {
        colleagues = c
    }
    
    func colleagueSent(_ msg: String) {
        switch msg {
        case "ConcreteColleague1 出价 300":
            colleagues?.forEach { $0.receiveReport(msg + "出价一次") }
            colleagues?.forEach { $0.receiveReport("两次") }
            colleagues?.forEach { $0.receiveReport("三次") }
            colleagues?.forEach { $0.receiveReport("成交") }
        default:
            colleagues?.forEach { $0.receiveReport(msg) }
        }
    }
}

class ConcreteColleague1: Colleague {
    var mediator: Mediator?

    func setMediator(_ m: Mediator) {
        mediator = m
    }
    
    func receiveReport(_ msg: String) {
        print(msg)
        
        switch msg {
        case "拍卖玉如意":
            DispatchQueue.main.asyncAfter(deadline: .now() + 1) { [weak self] in
                self?.mediator?.colleagueSent("ConcreteColleague1 出价 300")
            }
        default:
            break
        }
    }
}

class ConcreteColleague2: Colleague {
    var mediator: Mediator?

    func setMediator(_ m: Mediator) {
        mediator = m
    }
    
    func receiveReport(_ msg: String) {
        print(msg)
    }
}

```

测试一下

``` swift 
var m = ConcreteMediator()
var c1 = ConcreteColleague1()
c1.setMediator(m)
var c2 = ConcreteColleague2()
c2.setMediator(m)
m.createColleagues([c1, c2])
m.colleagueSent("拍卖玉如意")
```

```
拍卖玉如意
拍卖玉如意
ConcreteColleague1 出价 300出价一次
ConcreteColleague1 出价 300出价一次
两次
两次
三次
三次
成交
成交
```

可能我们这个比喻不太合适，不过也比较典型。仲裁者优先发布拍卖消息，分别通知给了许多的顾客；顾客收到消息后，给予反馈；仲裁者再将最终的结果通知到每一位顾客