### Builder模式

有着这么一个比喻：
在建造大楼时，我们需要先打牢地基，搭建框架，然后自下而上的一层一层盖起来；通常，在建造这种具有复杂结构的物理时，很难一气呵成，我们需要首先建造组成这个物体的各个部分，然后分阶段将他们组装起来

* 定义一个建造者协议：功能有建造车身、建造车轮、建造引擎等
* Audi流水线：流水线通过建造工人，建造一个完整的汽车
* Audi建造工人：建造汽车的各个部分

``` swift
protocol Builder {
    mutating func makeBody(_ b: String)
    mutating func makeWheel(_ w: String)
    mutating func makeEngine(_ e: String)
    func done()
}

struct AudiCarPipeline {
    
    private var builder: Builder
    
    init(_ builder: Builder) {
        self.builder = builder
    }
    
    mutating func construct() {
        builder.makeBody("糖果白")
        builder.makeEngine("Audi. EA888")
        builder.makeWheel("米其林")
        builder.done()
    }
}

struct AudiBuilder: Builder {
    private var car: String = ""
    
    mutating func makeBody(_ b: String) {
        car.append("车身: \(b)\n")
    }
    
    mutating func makeEngine(_ e: String) {
        car.append("引擎: \(e)\n")
    }
    
    mutating func makeWheel(_ w: String) {
        car.append("轮胎: \(w)\n")
    }
    
    func done() {
        print(car)
    }
}
```

来测试一下

``` swift
let builder = AudiBuilder()
var pipeline = AudiCarPipeline(builder)
pipeline.construct()
```

```
车身: 糖果白
引擎: Audi. EA888
轮胎: 米其林
```