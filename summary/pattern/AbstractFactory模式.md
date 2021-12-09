### AbstractFactory模式

前面我们提到过工厂模式的作用：生成实例

抽象工厂模式的作用：将"抽象零件"组装成"抽象产品"；也就是说我们不用关心零件的具体实现，只用关心接口（API）。我们仅使用该接口（API）将零件组装成为产品

* 首先定义抽象工厂
* 定义抽象零件
* 定抽象产品
* 实例化：工厂、零件、产品
* 使用抽象工厂生产产品

``` swift
/// 抽象协议
protocol AbstractItem {
    func makeHTML() -> String
}

protocol AbstractProduct {
    func descrption() -> String
}

protocol AbstractFactory {
    init()
    func createLink(_ url: String)
    func createDiv(_ text: String)
    func product() -> AbstractProduct
}

///抽象零件
protocol Div: AbstractItem {
    var innerText: String {get}
}
extension Div {
    func makeHTML() -> String {
        return "<div>" + innerText + "</div>"
    }
}

protocol Link: AbstractItem {
    var url: String {get}
}

extension Link {
    func makeHTML() -> String {
        return "<a>" + url + "</a>"
    }
}

///具体零件
struct ConcreteLink: Link {
    var url: String
}

struct ConcreteDiv: Div {
    var innerText: String
}

///具体产品
struct ConcreteHTML: AbstractProduct {
    var body: String
    
    func descrption() -> String {
        return "<html><body>\(body)</body></html>"
    }
}
///具体工厂
class HTMLFactory: AbstractFactory {
    
    var bodys: [AbstractItem] = []
    
    required init() {}
    
    func createLink(_ url: String) {
        bodys.append(ConcreteLink(url: url))
    }
    
    func createDiv(_ text: String) {
        bodys.append(ConcreteDiv(innerText: text))
    }

    func product() -> AbstractProduct {
        let body = bodys.reduce("") { partialResult, item in
            partialResult.appending(item.makeHTML())
        }
        return ConcreteHTML(body: body)
    }
}
///生成工厂帮助类
open class Factory {
    static func getFactory<T>(_ type: T.Type) -> T where T: AbstractFactory{
        return T.init()
    }
}

```

> 不太熟悉swift的同学可能看见下面代码就很懵。
static func getFactory<T>(_ type: T.Type) -> T where T: AbstractFactory
我给解释一下，这个其实是用来替代比如
getFactory(with name: String)；这里的泛型只是为了直接通过类型，生成一个对象的写法

OK，那我们使用一下

``` swift
var f = Factory.getFactory(HTMLFactory.self)
f.createDiv("我是第一个")
f.createLink("https://www.baidu.com")
let p = f.product()
print(p.descrption())
```

```
<html><body><div>我是第一个</div><a>https://www.baidu.com</a></body></html>
```