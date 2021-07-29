## 泛型

* 假如有如下关系

```
@interface Person<__covariant T> : NSObject
@property (nonatomic, strong) T country;
@end

@interface Country : NSObject
@end

@interface China : Country
@end
```

* 协变 （__covariant）<br>
在指定泛型时，可通过父类接收子类的指针，否则会被编译器提示<br>
例如  China: Country
```
Person<Country*> *p;
Person<China*> *pc;
p = pc;
pc = p; //这段代码将会被编译器提示
```

* 逆变 （__contravariant）
在指定泛型时，可通过子类接收父类的指针，否则会被编译器提示<br>
例如  China: Country
```
Person<Country*> *p;
Person<China*> *pc;
p = pc; //这段代码将会被编译器提示
pc = p;
```

