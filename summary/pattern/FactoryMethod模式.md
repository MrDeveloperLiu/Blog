### FactoryMethod模式

类比于模板模式我们在父类中规定处理的流程，在子类中实现具体的处理；但是如果我们将该模式用于实例的生成，就演变为我们本节要介绍的工厂模式

* 定义产品
* 定义工厂

``` swift
protocol Product {
    func use()
}
 
protocol Factory {
    func createProduct() -> Product
}

struct IDCard: Product {
    func use() {
        print("使用身份证")
    }
}

struct IDCardFactory: Factory {
    func createProduct() -> Product {
        return IDCard()
    }
}
```

* 工厂生产产品

``` swift 
let f = IDCardFactory()
let p = f.createProduct()
p.use()
```

则这就是利用不同的工厂，生成不同的产品