### Adapter模式

> 适配器模式也被成为Wrapper模式，有<font color=red>"包装器"</font>的意思；就像用精美的包装纸将普通的商品包装成礼物那样；替我们把某样东西包装起来，使其能够用于其他的用途。

适配器模式有两种实现方式：
* 类适配器模式<font color=cyan>（使用继承）</font>
* 对象适配器模式<font color=cyan>（使用协议）</font>

我们在这里拿对象适配器来演示：
* 首先定义`InputAdapter`协议，用来输出文档
* 初始化两个适配器`PDFInput`与`WordInput`分别实现了`InputAdapter`协议
* 被适配者实际上是`HPPrinter`，在适配者调用`execute()`时，执行适配器的输出

``` swift
protocol InputAdapter {
    func output() -> String
}

struct HPPrinter {
    
    var adaptee: InputAdapter
    
    init(_ adaptee: InputAdapter) {
        self.adaptee = adaptee
    }
    
    func execute() {
        print("HPPrinter: Preparing..")
        print(adaptee.output())
        print("HPPrinter: Done!")
    }
}

struct PDFInput: InputAdapter {
    func output() -> String {
        return "输出PDF文档"
    }
}

struct WordInput: InputAdapter {
    func output() -> String {
        return "输出Word文档"
    }
}
```

* 则上测试代码

``` swift
let printerPDF = HPPrinter(PDFInput())
printerPDF.execute()

let printerWord = HPPrinter(WordInput())
printerWord.execute()
```

* 控制台打印

```
HPPrinter: Preparing..
输出PDF文档
HPPrinter: Done!
HPPrinter: Preparing..
输出Word文档
HPPrinter: Done!
```

#### 拓展
> 什么时候使用适配器模式

在实际项目中，若你是做服务端开发的同学，难免遇痛点是什么？

开发人员A：产品，你这个需求不改了对吧？

产品：目前是，但今后不确定

开发人员A：wtm...

开发人员A对客户端开发B说：这样吧，我先给你v1版本的接口，今后改需求，那就升级v2版本

> 于是乎，这种场景就可以用到适配器模式

下面上伪代码：

```
if (version == '1.0') {

}
if (version == '2.0') {

}
if (version == '3.0') {

}

....我擦，版本多了怎么办？
```

> 适配器来也

想想一下，如果我们将代码改成上述的示例之后，再使用HashMap映射一下

``` swift
struct Adapter10: Adapter{...}
struct Adapter20: Adapter{...}
struct Adapter30: Adapter{...}
....

let map: [String: Adapter] = [
    '1.0': Adapter10(),
    '2.0': Adapter20()
    '3.0': Adapter30()
]
let adapter = map[version] ?? map['1.0']
adapter.doSth()
```

WTM...漂亮，随你产品怎么改