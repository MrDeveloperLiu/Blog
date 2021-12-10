### Proxy模式

这里的代理模式与平时在iOS开发中的Delegate不太一样，这里的代理模式强调的是：只在必要的时候生成实例


``` swift 
protocol Printable {
    mutating func fprint(_ file: String)
}


struct Printer {
    var name: String
    
    func printFile(_ file: String) {
        print("打印机:\(name) 打印文件 \(file)")
    }
}

struct PrinterProxy: Printable {
    
    var printer: Printer?
    
    mutating func fprint(_ file: String) {
        if printer == nil {
            printer = Printer(name: "HP")
        }
        printer?.printFile(file)
    }
}
```

测试一下

``` swift
var proxy = PrinterProxy()
proxy.fprint("Word文档")
```

```
打印机:HP 打印文件 Word文档
```