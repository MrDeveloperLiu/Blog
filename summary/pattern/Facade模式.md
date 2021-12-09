### Facade模式

门面模式，也叫窗口模式；指的是将数量庞大的类之间的错综复杂的关系整理出一些高层的接口，其中，窗口仅对外面暴露一个简单的接口；

例如计算机有什么组成：主机，显示屏，键盘，鼠标；当我们开机的时候，电脑主机会分别给键盘，鼠标，以及显示屏通电（假设显示器与主机一体）

``` swift 

protocol Components {
    func start()
    func ready() -> Bool
}


struct Mouse: Components {
    func start() {
        print("鼠标启动")
    }
    func ready() -> Bool {
        return true
    }
}
struct Keyboard: Components {
    func start() {
        print("键盘启动")
    }
    func ready() -> Bool {
        return true
    }
}

struct Displayer: Components {
    func start() {
        print("显示器启动")
    }
    func ready() -> Bool {
        return true
    }
}

struct Computer {
    var mouse = Mouse()
    var keyboard = Keyboard()
    var displayer = Displayer()
    
    func boost() {
        mouse.start()
        keyboard.start()
        displayer.start()
        
        if mouse.ready(), keyboard.ready(), displayer.ready() {
            print("启动完成")
        }
    }
        
}
```

那么`Computer`的门面模式下的Api就是`boost`