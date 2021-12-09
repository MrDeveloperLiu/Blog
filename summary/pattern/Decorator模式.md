### Decorator模式

不断地为对象添加装饰的设计模式，被称之为装饰器模式


``` swift
protocol Decorator {
    func toString() -> String
}

struct Border: Decorator {
    var border: String

    func toString() -> String {
        return border
    }
}

struct Some: Decorator {
    var title: String
    var decorator: Decorator
    
    func toString() -> String {
        return "\(decorator.toString()) \(title) \(decorator.toString())"
    }
}
```

测试代码

``` swift
let b = Border(border: "|")
let s = Some(title: "Some", decorator: b)
let s1 = Some(title: "Some1", decorator: s)
```

```
| Some |
| Some | Some1 | Some |
```

装饰物`Border`与被装饰物`Some`的一致性；在这里被装饰物也可以当成装饰物