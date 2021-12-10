### Memento模式

备忘录模式：通过将动作缓存下来，以达到恢复上一个状态的功能；

例如：在文本编译器中，我们通过cmd+s用来保存上一次的记录，不小心删了怎么办，通过撤销可以恢复上一个状态

同理：我们将设计一个画板程序，每次画出一条路径，可以撤销上次的更改

> 首先定义Memento，用来添加状态，与恢复上一个状态，再定义状态对象（指的是我们手指触摸的路径）


``` swift
struct Path {
    var points: [CGPoint] = []
    
    init(_ p: CGPoint) {
        points.append(p)
    }
    mutating func add(_ p: CGPoint) {
        points.append(p)
    }

    mutating func flush(_ p: CGPoint) {
        points.append(p)
    }
}

struct Memento<T> {
    var stores: [T] = []
    
    mutating func add(_ m: T) {
        stores.append(m)
    }
    
    mutating func restore() -> T? {
        return stores.popLast()
    }
}
```

> 其次书写简单的画板程序

``` swift

class TouchView: UIView {
            
    var currentPath: Path?
    var memento = Memento<Path>()
    
    override init(frame: CGRect) {
        super.init(frame: frame)
        
        let button = UIButton(frame: CGRect(x: 30, y: 100, width: 40, height: 40))
        button.backgroundColor = .black
        self.addSubview(button)
        button.addTarget(self, action: #selector(restore), for: .touchUpInside)
    }
    
    @objc func restore() {
        memento.restore()
        setNeedsDisplay()
    }
    
    required init?(coder: NSCoder) {
        fatalError("init(coder:) has not been implemented")
    }
    
    override func touchesBegan(_ touches: Set<UITouch>, with event: UIEvent?) {
        guard let p = touches.first?.location(in: self) else { return }
        currentPath = Path(p)
    }
    
    override func touchesMoved(_ touches: Set<UITouch>, with event: UIEvent?) {
        guard let p = touches.first?.location(in: self) else { return }
        currentPath?.add(p)
    }
    
    override func touchesEnded(_ touches: Set<UITouch>, with event: UIEvent?) {
        guard let p = touches.first?.location(in: self) else { return }
        currentPath?.flush(p)
        if let currentPath = currentPath {
            memento.add(currentPath)
        }
        currentPath = nil
        setNeedsDisplay()
    }
 
    override func draw(_ rect: CGRect) {
        guard let ctx = UIGraphicsGetCurrentContext(), !memento.stores.isEmpty else { return }
        
        memento.stores.forEach { p in
            let cgpath = CGMutablePath()
            for (idx, point) in p.points.enumerated() {
                if idx == 0 {
                    cgpath.move(to: point)
                }else{
                    cgpath.addLine(to: point)
                }
            }
            ctx.addPath(cgpath)
        }
        ctx.setStrokeColor(UIColor.red.cgColor)
        ctx.setLineWidth(1)
        ctx.strokePath()
    }
}
```

画板程序的功能是，手指触摸一次，滑动，离开屏幕，记录一下当前的状态；有个按钮可以恢复上一次的状态

> 上代码测试一下

``` swift
let v = TouchView(frame: UIScreen.main.bounds)
v.backgroundColor = .white
PlaygroundPage.current.liveView = v
```

OK，我们得到了想要的效果
