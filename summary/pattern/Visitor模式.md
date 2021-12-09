### Visitor模式

在数据结构中保存着许多元素，我们会对这些元素进行处理。在访问者模式中，数据结构与处理被分离开来；

我们编写一个表示"访问者"的类，来访问数据结构中的元素，并把对各个元素的处理交给访问者类；

这样，当需要增加新的处理时，我们只需要编写新的访问者，然后让数据结构可以接受访问者的访问即可；

``` swift
protocol Visitor {
    func visit(_ file: File)
    func visit(_ dir: Directory)
}

protocol Element {
    var name: String {get}
    
    func accept(_ v: Visitor)
}

struct Directory: Element {
    var name: String
    private(set) var files: [Element] = []
    
    mutating func add(_ e: Element) {
        files.append(e)
    }
    
    func accept(_ v: Visitor) {
        v.visit(self)
    }
}

struct File: Element {
    var name: String
    
    func accept(_ v: Visitor) {
        v.visit(self)
    }
}

struct ListVisitor: Visitor {
    
    func visit(_ file: File) {
        print("[FILE]:\(file.name)")
    }
    
    func visit(_ dir: Directory) {
        print("\(dir.name)/")
              
        dir.files.forEach { e in
            e.accept(self)
        }
    }
    
}
```

其实这个模式也是说的云里雾里；笔者认为，这个访问者模式的好处就在于，假如增加了元素类型，只需要让`Visitor`添加对元素的访问支持即可

``` swift
var dir = Directory(name: "root")
let f0 = File(name: "0.txt")
let f1 = File(name: "1.txt")
dir.add(f0)
dir.add(f1)
dir.accept(ListVisitor())
```

```
root/
[FILE]:0.txt
[FILE]:1.txt
```