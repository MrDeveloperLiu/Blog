### NSKeyValueCoding

`KVC`相信大家都并不陌生；在日常开发中，我们几乎都需要使用到它，无论是自我实现的功能也好，还是使用一些三方库也好；我们都需要将`Json`串转化为模型去使用

#### 基础功能
可大多人都只是知道基本的用法，往往只有面试过程中才会被问到原理。至于原理...其实笔者想说，你看得到`Foundation.framework`里的实现吗？当然，我们看不到。但是，我们有头文件，那么请移步`NSKeyValueCoding.h`中

```
@interface NSObject(NSKeyValueCoding)
@property (class, readonly) BOOL accessInstanceVariablesDirectly;
- (nullable id)valueForKey:(NSString *)key;
- (void)setValue:(nullable id)value forKey:(NSString *)key;
- (nullable id)valueForKeyPath:(NSString *)keyPath;
- (void)setValue:(nullable id)value forKeyPath:(NSString *)keyPath;
@end
```

以上五个（属性、方法）均为日常常用的，也是大家最先想到方法， 相信大多数同学都没有仔细研读过上方的注释；其实也难怪，我们日常开发中，经常使用的三方库例如`YYModel`使用到这些方法的机会都太少了，因为其对类型的判断，以及正确的调用API让我们很难在开发中遇到Bug


那么我们先看一下每个函数具体含义

|	函数	|	含义	|
|:--|:--|
|-valueForKey:|根据传入Key值获取对象内部的Value值|
|-valueForKeyPath:|根据传入Key值获取对象内部的Value值（可深层次查找例如"model.age"）|
|-setValue:forKey:|根据传入Value值更新Key对应的属性|
|-setValue:forKeyPath:|根据传入Value值更新Key对应的属性（可深层次赋值例如"model.age"）|

但事实上，如果我们使用的是对象中，并没有`key`或者`keyPath`时，结果如何呢？

那么依据`NSKeyValueCoding.h`中的头文件，以及笔者亲自实验的demo的结论来讲的话

* 若Setter与Getter均未实现，但是拥有成员变量，上述四个方法将从对象的Ivar中查找并赋值、取值

|	按照优先级（由上到下）	|	例 	|
|:---|:---|
|id _`<key>`|id _name|
|id_is`<Key>`|id _isName|
|id `<key>`|id name|
|id is`<Key>`|id isName|

到这里，我们还有一个属性没有讲：@property (class, readonly) BOOL accessInstanceVariablesDirectly;

那就是，这个值默认为`YES`，但是如果将其返回`NO`的话，对成员量变进行查找并赋值、取值将会**失效**


* `-valueForKey:` | `-valueForKeyPath:`  会优先从以下查找

实现的Getter

|	按照优先级（由上到下）	|	例 	|
|:---|:---|
|-(id)get`<key>`|-(id)getName|
|-(id)`<key>`|-(id)name|
|-(id)is`<key>`|-(id)isName|
|-(id)_`<key>`|-(id)_name|

**若上述中，Ivar与Getter均未实现**则会调用`- (nullable id)valueForUndefinedKey:(NSString *)key;`方法，该方法会抛出一个运行时异常

* `-setValue:forKey:` | `-setValue:forKeyPath:` 会优先执行下列步骤

实现的Setter

|	按照优先级（由上到下）	|	例 	|
|:---|:---|
|-set`<Key>`:|-setName:|
|-_set`<Key>`|-_setName:|

**若上述中，Ivar与Setter均未实现**则会调用`- (void)setValue:(nullable id)value forUndefinedKey:(NSString *)key;`方法，该方法会抛出一个运行时异常
**若给定的key对应的属性是非对象类型**，如果所传入value=nil的话，则会调用`- (void)setNilValueForKey:(NSString *)key;`的方法，该方法也会抛出一个运行时异常

#### 特殊功能
实际上，`-valueForKeyPath:` 这个依靠键值查找某个值还包含一些特殊的匹配模式

例如，如下关系

```
@interface Some : NSObject
@property (nonatomic, copy) NSString *key;
@property (nonatomic, strong) NSMutableArray *value;
@end

@interface SomeClass : NSObject
@property (nonatomic, assign) float age;
@property (nonatomic, strong) Some *some;
@end
```

模式匹配格式`[some keyPath].[@some order].[some keyPath]` 
表示 某个键值下-获取集合的命令-获取右值的键值；（当集合中的元素为NSNumber类型是，右值的keyPath==self），如下

```
//对SomeClass对象`cls`中的some属性中value数组，获取它的数量
 id value = [cls valueForKeyPath:@"some.value.@count"];
 //对SomeClass对象`cls`中的some属性中value数组，获取它的平均值
 id value = [cls valueForKeyPath:@"some.value.@avg.self"];
 //对SomeClass对象`cls`中的some属性中value数组，获取它的和
 id value = [cls valueForKeyPath:@"some.value.@sum.self"];
 //对SomeClass对象`cls`中的some属性中value数组，获取它的最大值
 id value = [cls valueForKeyPath:@"some.value.@max.self"];
 //对SomeClass对象`cls`中的some属性中value数组，获取它的最小值
 id value = [cls valueForKeyPath:@"some.value.@min.self"];

 //对SomeClass对象`cls`中的some属性中value数组，以它的右值的键值路径（元素）构建为一个新的集合，返回值为数组
 id value = [cls valueForKeyPath:@"some.value.@unionOfObjects.self"];
 //对SomeClass对象`cls`中的some属性中value数组，以它的右值的键值路径（元素）去重并构建为一个新的集合，返回值为数组
 id value = [cls valueForKeyPath:@"some.value.@distinctUnionOfObjects.self"];
 
 //对SomeClass对象`cls`中的some属性中value数组，以它的右值的键值路径（数组）构建为一个新的集合，返回值为数组
 id value = [cls valueForKeyPath:@"some.value.@unionOfArrays.self"];
 //对SomeClass对象`cls`中的some属性中value数组，以它的右值的键值路径（数组）去重并构建为一个新的集合，返回值为数组
 id value = [cls valueForKeyPath:@"some.value.@distinctUnionOfArrays.self"];
 //对SomeClass对象`cls`中的some属性中value数组，以它的右值的键值路径（Set）去重并构建为一个新的集合，返回值为数组
 id value = [cls valueForKeyPath:@"some.value.@distinctUnionOfSets.self"];
```

#### 集合类型
当我们使用`KVO`监听了某个对象，比如SomeClass中的some属性时，若some.value发生add或者remove的操作时，键值观察者并不会接收回调
则，我们可以通过以下方法。获取some.value的类型后，对其进行添加或者删除的操作，则观察者会收到回调

```
- (NSMutableArray *)mutableArrayValueForKey:(NSString *)key;
- (NSMutableOrderedSet *)mutableOrderedSetValueForKey:(NSString *)key
- (NSMutableSet *)mutableSetValueForKey:(NSString *)key;
```

#### 其他方法

* `- (NSDictionary<NSString *, id> *)dictionaryWithValuesForKeys:(NSArray<NSString *> *)keys;` 此方法可将所传入的键值数组一一查找后，返回一个字典结构
* `- (void)setValuesForKeysWithDictionary:(NSDictionary<NSString *, id> *)keyedValues;`此方法可以接收一个字段对象，将对象使用字典中的键值对，进行批量赋值操作