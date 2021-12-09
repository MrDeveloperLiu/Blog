### Prototype模式

在开发中难免会遇到问题：
* 对象种类繁多，无法将它们整合到一个类中
* 难以根据类生成实例时
* 想解耦框架与生成的实例时

例如：Objc中的`NSCoping`协议，我们知道当一个类作为字典的键的时候，该对象需要遵守该协议；原理是字典类将保存一份基于键克隆出来的对象，作为键值对的键

那么类似的，我们拿`swift`来模拟一下实现过程；我们模拟一个序列

``` swift
protocol Cloneable {
    func clone() -> Self
}


struct Book: Cloneable {
    var name: String
    
    func clone() -> Book {
        return Book(name: self.name)
    }
}

struct Paper: Cloneable {
    var index: Int
    
    func clone() -> Paper {
        return Paper(index: self.index)
    }
}


struct AnySequence: CustomDebugStringConvertible {
    var store: [Cloneable] = []
    
    mutating func insert(_ c: Cloneable) {
        store.append(c.clone())
    }
    
    var debugDescription: String {
        return "AnySequence: ->store \(store)"
    }
}
```

来测试一下

``` swift 
var seq = AnySequence()
seq.insert(Paper(index: 100))
seq.insert(Book(name: "Hello Mr"))
print(seq)
```

那么我们通过传入的对象，保存了它的一个副本

```
AnySequence: ->store [__lldb_expr_14.Paper(index: 100), __lldb_expr_14.Book(name: "Hello Mr")]
```