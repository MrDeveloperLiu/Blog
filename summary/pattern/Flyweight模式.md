### Flyweight模式


享元模式：实际上就是将一些占用计算机内存较大的对象共享使用避免浪费储存的开销

其实在我们程序中多用的是什么？单例，比如说一个网络请求的参数对象，如果服务器要求我们每次毕传许多header，与许多公共参数等；那么为何我们不将他们保存在一个单例对象中，每次从对象中构造呢？

其实这个例子也不是特别典型，真正典型的例子则是BitMap，用字节流存储数据，又比如，如果我们存储一个白色对象，那为什么我们不存储一个（0xffffff）呢。起码值类型，比引用类型消耗的内存更小一些吧？这些都是细节

那我们将模拟一个大字符串存储


``` swift 

struct ShareMap {
    static let words: [String] = [
        "a",
        "b",
        "c",
        "d",
        "e",
        "f",
        "g"
    ]
          
    func getString(_ d: Data) {
        var str = ""
        for idx in 0..<d.count {
            let index = Int(d[idx])
            str += ShareMap.words[index % ShareMap.words.count]
        }
        print(str)
    }
}

```

测试一下，我们将下标为1,2,3塞入Data中，不出意外，应该输出bcd

```
let m = ShareMap()

var d = Data()
d.append(0x01)
d.append(0x02)
d.append(0x03)
m.getString(d)

```

```
bcd
```


我们这里的`元`就是`static let words: [String]`