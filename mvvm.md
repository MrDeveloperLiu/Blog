## mvvm

* 框架遵循的引用关系

> /*
> <br> MVVM框架结构需要遵循如下关系：即`VC`或者`View`引用着`ViewModel`（反之不行），`ViewModel`引用着`Model`（同样：反之不行）
> <br>*/
> <br>那么为什么呢，为什么非要拥有如下关系？
> <br>`VC/View => VM => Model`


* 引用结构中的角色
> 1. `VC、View` 扮演整个页面如某些Page，某些独立与`VC`的全屏展示的`View` <br> 思考: 在传统的`MVC`架构中，`VC、View`处理的太多的逻辑，导致特别臃肿，难以维护。所谓的`VC、View`是否应该更多的关心，我某个控件应该如何展示

> 2. `VM` 扮演事件发送者，逻辑决策者，数据获取者 <br> 思考：`View`利用与`ViewModel`的绑定关系，将用户事件分发到`ViewModel`中，由`ViewModel`调动`Model`去获取数据等等

> 3. `Model` 扮演数据处理者 <br> ***当我们从服务获取完数据之后，由`Model`回调到`ViewModel`中时的`ViewModel`获取到最新的数据，再由`ViewModel`触发界面更新，将最新的数据绑定到`View`上

* 那我们来拿个个例子来说
```
///首先 我们先声明三条协议
protocol ViewUpdater {
    func update()
}

protocol ModelExecuter {
    func fetch(_ paramters: Any, with completion: (Any) -> Void)
}

protocol ViewModel {
    var model: ModelExecuter {get}
    var delegate: ViewUpdater {set get}
    var data: Any? {set get}
    func exec(_ text: String)
}

///其次根据协议创建具象VM与M类
class ConcreteModel: NSObject, ModelExecuter {
    func fetch(_ paramters: Any, with completion: (Any) -> Void) {
        print("fectch some datas with paramters \(paramters)");
        completion("some datas")
    }
}

class ConcreteViewModel: NSObject, ViewModel {
    var delegate: ViewUpdater
    var model: ModelExecuter = ConcreteModel()
    var data: Any? {
        didSet{
            self.delegate.update()
        }
    }
    
    init(delegate: ViewUpdater) {
        self.delegate = delegate
        super.init()
    }
    
    func exec(_ text: String) {
        model.fetch("paramters") { data in
            self.data = data
        }
    }
}

///最终实现VC、View
class ViewController: UIViewController, ViewUpdater{
    
    var btn: UIButton = UIButton(frame: CGRect(x: 10, y: 100, width: 100, height: 50))
    var viewModel: ViewModel?
    
    override func viewDidLoad() {
        super.viewDidLoad()
        
        btn.backgroundColor = .cyan
        btn.setTitle("点我试试看", for: .normal)
        btn.addTarget(self, action: #selector(btnAction), for: .touchUpInside)
        view.addSubview(btn)
        
        viewModel = ConcreteViewModel(delegate: self)
        viewModel?.delegate = self
    }
    
    @objc func btnAction() {
        viewModel?.exec("query")
    }
    
    func update() {
        print("update view")
        if let t = viewModel?.data as? String {
            self.btn.setTitle(t, for: .normal)
        }
    }
    
}

```

OK，到这里，就能够看到，`VM`中绑定了`Button`的点击事件，在事件里模拟请求服务，最终回调的结构用来更新`View`。这样业务逻辑是不是就更加清晰了~

想象一下，如果这时候我们使用了一些响应式框架，那么代码是否就可以变成类似这样（是不是很爽）：
```
class ViewController: UIViewController, ViewUpdater{
    
    var btn: UIButton = UIButton(frame: CGRect(x: 10, y: 100, width: 100, height: 50))
    var viewModel: ViewModel?
    
    override func viewDidLoad() {
        super.viewDidLoad()
        
        btn.backgroundColor = .cyan
        btn.setTitle("点我试试看", for: .normal)
        view.addSubview(btn)

        viewModel = ConcreteViewModel(delegate: self)
        viewModel?.delegate = self

        //假装这是有效的代码
        btn.bindTouchUpinsideEvent({ e in
            viewModel?.exec("query")
        })
        btn.bindTo(title, viewModel?.data)
    }
    

}
```
