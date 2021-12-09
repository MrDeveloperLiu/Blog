### Chain Of Responseibility模式


将多个对象组成一条职责链，然后按照他们的职责链的顺序，一个一个地找出到底应该谁来负责

这就像我们日常推卸责任（踢皮球）：出线上问题，老板找到经理，经理找到测试，测试找到程序员定位问题


``` swift

protocol Support {
    var next: Support? {get set}
    func handle(_ t: Trouble) -> Bool
}

struct ProductManager: Support, CustomDebugStringConvertible {
    var debugDescription: String {
        return "产品经理"
    }
    
    var next: Support?
    
    func handle(_ t: Trouble) -> Bool {
        return false
    }
}

struct QAContact: Support, CustomDebugStringConvertible {
    var next: Support?
    
    var debugDescription: String {
        return "测试人员"
    }
    func handle(_ t: Trouble) -> Bool {
        return false
    }
}

struct Developer: Support, CustomDebugStringConvertible {
    var next: Support?
    
    var debugDescription: String {
        return "程序员"
    }

    func handle(_ t: Trouble) -> Bool {
        return true
    }
}

struct Trouble {
    var desc: String
}

struct Boss {
    
    var response: Support
    
    init(response: Support) {
        self.response = response
    }
    
    func trigger() {
        let t = Trouble(desc: "App崩溃啦")
        print("老板：WTM \(t.desc)")
        var response: Support? = response
        while let responseder = response {
            print("问: \(responseder) 可以处理问题吗")
            
            if responseder.handle(t) {
                print("可以处理问题")
                break
            }else{
                print("不能啊 --- Fuck")
            }
            response = response?.next
        }
        
    }
    
}
```

OK下面我们来讲一个段子，某天，老板发现app崩溃了，叫来的产品，来你给我处理下问题！！

于是就有下面的测试代码

``` swift
let coder = Developer(next: nil)
let qa = QAContact(next: coder)
let pm = ProductManager(next: qa)
let boss = Boss(response: pm)
boss.trigger()
```

```
老板：WTM App崩溃啦
问: 产品经理 可以处理问题吗
不能啊 --- Fuck
问: 测试人员 可以处理问题吗
不能啊 --- Fuck
问: 程序员 可以处理问题吗
可以处理问题
```

是不是发现了点什么？YES，跟iOS的响应者链是不是有点像