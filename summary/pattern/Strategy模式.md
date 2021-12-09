### Strategy模式

无论什么程序，其目的都是为了解决问题。解决问题时，我们又要编写特定的算法，那么策略模式可以整体的替换算法的实现部分；让我们们轻松的以不同的算法去解决同一个问题，这种模式就是策略模式

* 制定策略
* 实现策略
* 根据条件筛选策略

``` swift

protocol PayStrategy {
    func pay(_ money: String)
}



struct AliPay: PayStrategy {
    
    func pay(_ money: String) {
        print("调用支付宝支付\(money)元")
    }
}

struct WeixinPay: PayStrategy {
    
    func pay(_ money: String) {
        print("调用微信支付\(money)元")
    }
}

struct Store {
    
    enum PayType {
        case alipay
        case weixinpay
    }
    
    func charge(_ money: String, type: PayType) {
        switch type {
        case .alipay:
            AliPay().pay(money)
        case .weixinpay:
            WeixinPay().pay(money)
        }
    }
    
}
```

那么着这里就是商店收银的时候，售货员询问顾客，用什么支付呀？客人说微信，好的，那我选择微信收款；叮咚，微信10元到账

来测试一下

``` swift
let s = Store()
s.charge("10", type: .alipay)
```

```
调用支付宝支付10元
```