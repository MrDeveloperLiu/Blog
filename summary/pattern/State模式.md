### State模式

状态模式其实就是用类来表示状态

那么我们设计这么一个程序：时钟

规则就是：一种状态是计时器状态的，另一种则是正常的展示状态


``` swift 


protocol State {
    func run(_ second: Int)
}

struct Alarm: State {
    var second: Int
    
    func run(_ second: Int) {
        if self.second == second {
            print("时间到了")
        }
    }
}

struct Normal: State {
    func run(_ second: Int) {
        print("第\(second)秒")
    }
}

class Clock {
    var state: State
    
    init(state: State) {
        self.state = state
    }
    
    func changeState(_ s: State) {
        print("状态切换")
        state = s
    }
    
    func run(_ running: (Int) -> Void) {
        for i in 1..<20 {
            Thread.sleep(forTimeInterval: 1)
            state.run(i)
            running(i)
        }
    }
}
```

测试一下 

``` swift

var c = Clock(state: Normal())
print("开始")
c.run { sec in
    if sec == 10 {
        c.changeState(Alarm(second: 15))
    }
}
print("结束")
```

```
开始
第1秒
第2秒
第3秒
第4秒
第5秒
第6秒
第7秒
第8秒
第9秒
第10秒
状态切换
时间到了
结束
```

