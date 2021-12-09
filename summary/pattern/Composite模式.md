### Composite模式

组合模式就是用于创造出容器与内容的结构的模式，使得容器与内容具有一致性，创造出递归结构的模式

我们拿文件与文件夹来举例子，众所周知，文件可以放到文件夹中，而文件夹也可以放到文件夹中；那么在这里，内容是（文件+文件夹）容器是（文件夹）

``` swift 

protocol Content: CustomDebugStringConvertible {
    var name: String {get}
    var size: Int {get}
    
    func show(_ prefix: String)
}

extension Content {
    var debugDescription: String {
       return "\(name)（\(size)）"
    }
}


struct ConcreteFile: Content {
    var name: String
    var size: Int
        
    func show(_ prefix: String) {
        print("\(prefix)/\(self)")
    }
}


struct ConcreteDirectory: Content {
    var name: String
    var size: Int {
        return contents.reduce(0) { partialResult, content in
            return partialResult + content.size
        }
    }
    var contents: [Content] = []
    
    mutating func add(_ content: Content) {
        contents.append(content)
    }
    
    func show(_ prefix: String) {
        print("\n")
        let dir = "\(prefix)/\(name)"
        print("\(dir)/（\(size)）\n")
        contents.forEach { content in
            content.show(dir)
        }
    }
}
```

测试一下

``` swift
var dir = ConcreteDirectory(name: "root")

let file0 = ConcreteFile(name: "1.txt", size: 100)
let file1 = ConcreteFile(name: "2.txt", size: 500)

var dir1 = ConcreteDirectory(name: "next")
let file2 = ConcreteFile(name: "3.txt", size: 0)
let file3 = ConcreteFile(name: "4.txt", size: 10000)
dir1.add(file2)
dir1.add(file3)


dir.add(file0)
dir.add(file1)
dir.add(dir1)

dir.show("usr/bin")
```

```
usr/bin/root/（10600）

usr/bin/root/1.txt（100）
usr/bin/root/2.txt（500）


usr/bin/root/next/（10000）

usr/bin/root/next/3.txt（0）
usr/bin/root/next/4.txt（10000）
```