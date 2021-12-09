### Singleton模式

只能生成一个实例；当程序运行中，我们想只生成一个实例时（比如当做AppSetting），那我们如何确保只生成一个实例就是个问题。要避免`new`多个对象


``` swift
class Singleton {
    
    static let shareInstance = Singleton()
    
    var baseURL: String = "https://www.baidu.com"
}
```

那我们来使用一下

``` swift
print(Singleton.shareInstance.baseURL)
Singleton.shareInstance.baseURL = "https://www.sina.com"
print(Singleton.shareInstance.baseURL)
```

当我们使用`Singleton.shareInstance`时，就可确保不会`new`出多个`Singleton`实例