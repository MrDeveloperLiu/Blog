> 日常开发中：我们免不了使用一下自定义的CollectionViewLayout
>
> 故：百度了一下，找到了个总结，记录一下
>
> 注意：本文摘自[链接](https://www.jianshu.com/p/ac40c2be4209)；并非笔者写，仅仅作为记录的笔记使用



#### UICollectionViewLayout

```
- (instancetype)init NS_DESIGNATED_INITIALIZER;
- (nullable instancetype)initWithCoder:(NSCoder *)aDecoder NS_DESIGNATED_INITIALIZER;
//当前布局的collectionview
@property (nullable, nonatomic, readonly) UICollectionView *collectionView;

//使布局失效(所有)
- (void)invalidateLayout;
//传入的上下文失效,优化布局性能
- (void)invalidateLayoutWithContext:(UICollectionViewLayoutInvalidationContext *)context NS_AVAILABLE_IOS(7_0);

//注册装饰view
- (void)registerClass:(nullable Class)viewClass forDecorationViewOfKind:(NSString *)elementKind;
- (void)registerNib:(nullable UINib *)nib forDecorationViewOfKind:(NSString *)elementKind;
```

#### UICollectionViewLayout (UISubclassingHooks)

```
#if UIKIT_DEFINE_AS_PROPERTIES
//如果你为你的布局对象自定义了一个无效内容子类，你需要重写此方法.
@property(class, nonatomic, readonly) Class layoutAttributesClass; 
//返回用来创建布局属性对象的类。如果你的
//UICollectionViewLayoutAttributes的子类为了管理额外的布局属性，你
//需要重写这个方法并返回他自己的自定义子类。当创建新的布局属性
//对象时这个方法使用这个类创建布局属性。这个方法只会用到这个子类，而不会调用你的代码。
@property(class, nonatomic, readonly) Class invalidationContextClass NS_AVAILABLE_IOS(7_0); 
#else
+ (Class)layoutAttributesClass; // override this method to provide a custom class to be used when instantiating instances of UICollectionViewLayoutAttributes
+ (Class)invalidationContextClass NS_AVAILABLE_IOS(7_0); // override this method to provide a custom class to be used for invalidation contexts
#endif
/** -------------------- 布局属性 -------------------------------- */
//准备布局调用的方法.
- (void)prepareLayout;

/*
返回UICollectionViewLayoutAttributes 类型的数组，
UICollectionViewLayoutAttributes 对象包含cell或view的布局信息。
子类必须重载该方法，并返回该区域内所有元素的布局信息，包括cell,追加视图和装饰视图。
*/
- (nullable NSArray<__kindof UICollectionViewLayoutAttributes *> *)layoutAttributesForElementsInRect:(CGRect)rect; // return an array layout attributes instances for all the views in the given rect

/*
返回指定indexPath的item的布局信息。子类必须重载该方法,该方法
只能为cell提供布局信息，不能为补充视图和装饰视图提供。
*/
- (nullable UICollectionViewLayoutAttributes *)layoutAttributesForItemAtIndexPath:(NSIndexPath *)indexPath;

/*
如果你的布局支持追加视图的话，必须重载该方法，该方法返回的是
追加视图的布局信息，kind这个参数区分段头还是段尾的，在collectionview注册的时候回用到该参数。
*/
- (nullable UICollectionViewLayoutAttributes *)layoutAttributesForSupplementaryViewOfKind:(NSString *)elementKind atIndexPath:(NSIndexPath *)indexPath;

/*
如果你的布局支持装饰视图的话，必须重载该方法，该方法返回的是装饰视图的布局信息，
ecorationViewKind这个参数在collectionview注册的时候回用到
*/
- (nullable UICollectionViewLayoutAttributes *)layoutAttributesForDecorationViewOfKind:(NSString*)elementKind atIndexPath:(NSIndexPath *)indexPath;

/** ------------------------ 无效布局------------------------ */
/*
该方法用来决定是否需要更新布局。如果collection view需要重新布局
返回YES,否则返回NO,默认返回值为NO。
*/
- (BOOL)shouldInvalidateLayoutForBoundsChange:(CGRect)newBounds; // return YES to cause the collection view to requery the layout for geometry information

/*
当bounds发生改变时返回一个需要变化的定义了部分布局的上下文对象
这个方法的默认实现是创建一个invalidationContextClass提供的类的实
例并且返回这个实例。如果你想要使用对你的布局自定义的无效上下文
对象，需要重写这个方法返回你的自定义类的实例。
如果你想要在响应bounds改变时创建并配置你的自定义无效上下文你
可以重写这个方法。如果你重写了这个方法，你必须首先调用super来
得到返回的无效上下文对象。得到这个对象之后，设置你想要设置的属性，然后返回这个对象。
*/
- (UICollectionViewLayoutInvalidationContext *)invalidationContextForBoundsChange:(CGRect)newBounds NS_AVAILABLE_IOS(7_0);

/**
preferredAttributes [UICollectionViewLayoutAttributes]cell的- (UICollectionViewLayoutAttributes *)preferredLayoutAttributesFittingAttributes:(UICollectionViewLayoutAttributes *)layoutAttributes;
方法返回的布局属性。
originalAttributes [UICollectionViewLayoutAttributes] - 布局对象对cell的最初建议属性。

一个自动调整大小的cell改变时，询问布局对象是否需要一个
较大的更新。当一个Collection View
包含自动调整大小的Cell时，让这些Cell在自己的布局属性
生效之前有机会修改这些属性。一个自动调整大小的Cell可
能指定一个不同的Cell大小而不是提供的布局对象。当这个Cell提供一个不同的属性集合，Collection View
调用这个方法来确定Cell的改变是否需要较大的布局更新。如果你
实现一个自定义布局，你可以重写这个方法并且用这个方法根据给定的
属性来确定是否你的布局需要被无效化。这个方法的默认实现是返回NO
。
*/
- (BOOL)shouldInvalidateLayoutForPreferredLayoutAttributes:(UICollectionViewLayoutAttributes *)preferredAttributes withOriginalAttributes:(UICollectionViewLayoutAttributes *)originalAttributes NS_AVAILABLE_IOS(8_0);

/**
当一个动态Cell改变时，返回一个上下文对象来标示出需要改变的部分布局。
这个方法的默认实现是创建一个invalidationContextClass提供的类的实
例，并返回这个实例。如果你想要对你的布局使用一个自定义无效上下
文对象，重写这个方法，并返回你的自定义的类的实例。
子类化可以重写这个方法，使用这个方法在返回之前来执行额外的对无
效上下文的配置。在你的自定义实现中调用super，以便父类可以执行对这个对象的基本配置。
*/
- (UICollectionViewLayoutInvalidationContext *)invalidationContextForPreferredLayoutAttributes:(UICollectionViewLayoutAttributes *)preferredAttributes withOriginalAttributes:(UICollectionViewLayoutAttributes *)originalAttributes NS_AVAILABLE_IOS(8_0);

/** -------------------- 布局属性 -------------------------------- */
/**
当滚动停止时，会调用该方法确定collectionView滚动到的位置

 * proposedContentOffset CGPoint - 停止滚动的建议位置点（在
collection view的内容区域中）。这个值如果没有调整将是自然停止的
点。这个点指的是可见区域的左上角的点的坐标。
 * velocity CGPoint - 沿着水平或垂直滚动的当前速度。这个值表示的是每秒滚动的点的数量。
*/
- (CGPoint)targetContentOffsetForProposedContentOffset:(CGPoint)proposedContentOffset withScrollingVelocity:(CGPoint)velocity; 
- (CGPoint)targetContentOffsetForProposedContentOffset:(CGPoint)proposedContentOffset NS_AVAILABLE_IOS(7_0);

/**
返回collectionView内容区的宽度和高度，子类必须重载该方法，
返回值代表了所有内容的宽度和高度，而不仅仅是可见范围的，
collectionView通过该信息配置它的滚动范围，默认返回 CGSizeZero。
*/
#if UIKIT_DEFINE_AS_PROPERTIES
@property(nonatomic, readonly) CGSize collectionViewContentSize; 
#else
- (CGSize)collectionViewContentSize; 
#endif
```

#### UICollectionViewLayout (UIUpdateSupportHooks)

```

/** -------------------协助动画改变 ---------------------------- */
/**
为了动态改变视图的bounds或者Item的插入删除准备布局对象。

Collection View在执行任何动态改变视图的bounds或者动态的插入删
除Item之前都会调用这个方法。这个方法让布局对象有机会为这些动态
改变进行任何需要的计算准备。具体来说，你可能使用这个方法计算插
入或删除的Item的最初的和最终的位置，以便当需要这些值的时候你可
以返回这些值。
你也可以使用这个方法来处理额外的动画。你创建的任何动画被添加到
动画的block中，用来处理插入、删除以及bounds的改变。
*/
- (void)prepareForAnimatedBoundsChange:(CGRect)oldBounds; // UICollectionView calls this when its bounds have changed inside an animation block before displaying cells in its new bounds

/**
在动态改变视图的bounds或者Item的插入删除之后进行清理。

Collection View在创建改变视图的bounds的动画或者动态插入或删除
Item之后调用这个方法。这个方法让布局对象有机会清除和这些操作有
关的内容。
你也可以使用这个方法来处理额外的动画。你创建的任何动画被添加到
动画的block中，用来处理插入、删除以及bounds的改变。
*/
- (void)finalizeAnimatedBoundsChange; // also called inside the animation block

/**-------------------------- 布局之间的过度 -------------------- */
/**
 告诉布局对象准备为Collection View安装布局。

 在执行布局转换之前，Collection View调用这个方法，以便与你的布局
 对象可以执行一些最初的有需要的计算来生成布局属性。
*/
- (void)prepareForTransitionToLayout:(UICollectionViewLayout *)newLayout NS_AVAILABLE_IOS(7_0);
/**
告诉布局对象，这个正要从Collection View中被移除的布局。

在执行布局转换之前，Collection View调用这个方法，以便与你的布局
对象可以执行一些最初的有需要的计算来生成布局属性。
*/
- (void)prepareForTransitionFromLayout:(UICollectionViewLayout *)oldLayout NS_AVAILABLE_IOS(7_0);
/**
告诉布局对象在过渡动画发生之前执行任何最终步骤。

当Collection View已经了解了从一个布局转换到另一个布局所有需要的
布局属性后，Collection View调用这个方法。你可以使用这个方法清除
任何数据结构或者清除你在prepareForTransitionFromLayout或
prepareForTransitionToLayout方法中实现时产生的缓存。
*/
- (void)finalizeLayoutTransition NS_AVAILABLE_IOS(7_0);  // called inside an animation block after the transition

/** ----------------- 响应UICollectionview 更新 -------------------- */
/**
通知布局对象，Collection View的内容将要更新。

当Item被插入或删除时，Collection View会通知它的布局对象，以便它
可以根据需要调整布局。过程的第一步事调用这个方法使布局对象知道
预计的改变。之后，继续调用收集整个Collection View中将要将要有插
入，删除和移动动画的Item的布局信息。
*/
- (void)prepareForCollectionViewUpdates:(NSArray<UICollectionViewUpdateItem *> *)updateItems;

/**
Collection View更新期间执行任何额外动画或者清理所需。

Collection View调用这个方法作为之前动画在当前位置改变的最后一
步。这个方法在动画的block里面被调用，用来执行所有插入，删除和
移动动画，所以你可以使用这个方法创建需要的额外动画。或者，你可
以使用它来执行任何与管理你的布局对象的状态信息相关的最后一分钟任务。
*/
- (void)finalizeCollectionViewUpdates; // called inside an animation block after the update


/**
 返回一个Item插入到Collection View中开始的布局信息。

 这个方法在- (void)prepareForCollectionViewUpdates: (NSArray<UICollectionViewUpdateItem *> *)updateItems;方法后调 
 用，在- (void)finalizeCollectionViewUpdates;方法之前调用，在任何
 一个Item被插入之前都会调用。
 你的实现需要返回描述了Item的最初位置和状态的布局信息。
 Collection View使用这个信息作为动画的开始位置。（动画的终点位 置
 是这个Item在Collection View中的新位置。）如果你返回nil，那么Item
 会使用它最后的属性作为动画的开始和结束位置。
 这个方法的默认实现就是返回nil。
 */
- (nullable UICollectionViewLayoutAttributes *)initialLayoutAttributesForAppearingItemAtIndexPath:(NSIndexPath *)itemIndexPath;

/**
返回一个Item从Collection View中被移除时的结束布局信息。

这个方法在- (void)prepareForCollectionViewUpdates:(NSArray<UICollectionViewUpdateItem *> *)updateItems;方法后调
用，在- (void)finalizeCollectionViewUpdates;方法之前调用，在任何
一个Item被删除之前都会调用。
Collection View使用这个信息作为动画的结束位置。（动画的起始位置
是这个Item在Collection View中的当前位置。）如果你返回nil，那么
Item会使用它最后的属性作为动画的开始和结束位置。
这个方法的默认实现就是返回nil。
*/
- (nullable UICollectionViewLayoutAttributes *)finalLayoutAttributesForDisappearingItemAtIndexPath:(NSIndexPath *)itemIndexPath;

/**
返回一个Supplementary View插入到Collection View中开始的布局信息。

这个方法在- (void)prepareForCollectionViewUpdates:(NSArray<UICollectionViewUpdateItem *> *)updateItems;方法后调
用，在- (void)finalizeCollectionViewUpdates;方法之前调用，在任何
一个Supplementary View被插入之前都会调用。
你的实现需要返回描述了Supplementary View的最初位置和状态的布
局信息。Collection View使用这个信息作为动画的开始位置。（动画的
终点位置是这个Supplementary View在Collection View中的新位
置。）如果你返回nil，那么Supplementary View会使用它最后的属性
作为动画的开始和结束位置。
这个方法的默认实现就是返回nil。
*/
- (nullable UICollectionViewLayoutAttributes *)initialLayoutAttributesForAppearingSupplementaryElementOfKind:(NSString *)elementKind atIndexPath:(NSIndexPath *)elementIndexPath;

/**
返回一个Supplementary View从Collection View中被移除时的结束布局信息。

这个方法在- (void)prepareForCollectionViewUpdates:(NSArray<UICollectionViewUpdateItem *> *)updateItems;方法后调
用，在- (void)finalizeCollectionViewUpdates;方法之前调用，在任何
一个Supplementary View被删除之前都会调用。
Collection View使用这个信息作为动画的结束位置。（动画的起始位置
是这个Supplementary View在Collection View中的当前位置。）如果
你返回nil，那么Supplementary View会使用它最后的属性作为动画的开始和结束位置。
这个方法的默认实现就是返回nil。
*/
- (nullable UICollectionViewLayoutAttributes *)finalLayoutAttributesForDisappearingSupplementaryElementOfKind:(NSString *)elementKind atIndexPath:(NSIndexPath *)elementIndexPath;

/**
返回一个Decoration View插入到Collection View中开始的布局信息。

这个方法在- (void)prepareForCollectionViewUpdates:(NSArray<UICollectionViewUpdateItem *> *)updateItems;方法后调
用，在- (void)finalizeCollectionViewUpdates;方法之前调用，在任何
一个Decoration View被插入之前都会调用。
你的实现需要返回描述了Decoration View的最初位置和状态的布局信
息。Collection View使用这个信息作为动画的开始位置。（动画的终点
位置是这个Decoration View在Collection View中的新位置。）如果你
返回nil，那么Decoration View会使用它最后的属性作为动画的开始和结束位置。
这个方法的默认实现就是返回nil。
*/
- (nullable UICollectionViewLayoutAttributes *)initialLayoutAttributesForAppearingDecorationElementOfKind:(NSString *)elementKind atIndexPath:(NSIndexPath *)decorationIndexPath;

/**
返回一个Decoration View从Collection View中被移除时的结束布局信息。

这个方法在- (void)prepareForCollectionViewUpdates:(NSArray<UICollectionViewUpdateItem *> *)updateItems;方法后调
用，在- (void)finalizeCollectionViewUpdates;方法之前调用，在任何
一个Decoration View被删除之前都会调用。
Collection View使用这个信息作为动画的结束位置。（动画的起始位置
是这个Decoration View在Collection View中的当前位置。）如果你返
回nil，那么Decoration View会使用它最后的属性作为动画的开始和结
束位置。
这个方法的默认实现就是返回nil。
*/
- (nullable UICollectionViewLayoutAttributes *)finalLayoutAttributesForDisappearingDecorationElementOfKind:(NSString *)elementKind atIndexPath:(NSIndexPath *)decorationIndexPath;


/**
返回将要移除的Supplementary View对应的索引组成的数组。如果你不想移除任何给定的类型的Supplementary View，请返回空数组。

无论你删除Collection View中的cell还是section，Collection View都会
调用这个方法。实现这个方法让你的布局对象有机会删除不再需要的
Supplementary View。
Collection View在调用- (void)prepareForCollectionViewUpdates:
(NSArray<UICollectionViewUpdateItem *> *)updateItems;和- 
(void)finalizeCollectionViewUpdates;方法之间调用这个方法。
*/
- (NSArray<NSIndexPath *> *)indexPathsToDeleteForSupplementaryViewOfKind:(NSString *)elementKind NS_AVAILABLE_IOS(7_0);
/**
返回将要移除的Decoration View对应的索引组成的数组。如果你不想移除任何给定的类型的Decoration View，请返回空数组。

无论你删除Collection View中的cell还是section，Collection View都会
调用这个方法。实现这个方法让你的布局对象有机会删除不再需要的Decoration View。
Collection View在调用- (void)prepareForCollectionViewUpdates:
(NSArray<UICollectionViewUpdateItem *> *)updateItems;和- 
(void)finalizeCollectionViewUpdates;方法之间调用这个方法。
*/
- (NSArray<NSIndexPath *> *)indexPathsToDeleteForDecorationViewOfKind:(NSString *)elementKind NS_AVAILABLE_IOS(7_0);

/**
返回你想要添加到布局中的Supplementary View对应的索引组成的数
组。如果你不想添加任何Supplementary View，请返回空数组。

无论你添加cell还是section到Collection View中，Collection View都会
调用这个方法。实现这个方法让你的布局对象有机会添加新的Supplementary View。
Collection View在调用- (void)prepareForCollectionViewUpdates:
(NSArray<UICollectionViewUpdateItem *> *)updateItems;和- 
(void)finalizeCollectionViewUpdates;方法之间调用这个方法。
*/
- (NSArray<NSIndexPath *> *)indexPathsToInsertForSupplementaryViewOfKind:(NSString *)elementKind NS_AVAILABLE_IOS(7_0);

/**
返回你想要添加的Decoration View对应的索引组成的数组。如果你不
想添加任何Decoration View，请返回空数组。

无论你添加cell还是section到Collection View中，Collection View都会
调用这个方法。实现这个方法让你的布局对象有机会添加新的Decoration View。
Collection View在调用- (void)prepareForCollectionViewUpdates:
(NSArray<UICollectionViewUpdateItem *> *)updateItems;和- 
(void)finalizeCollectionViewUpdates;方法之间调用这个方法。
*/
- (NSArray<NSIndexPath *> *)indexPathsToInsertForDecorationViewOfKind:(NSString *)elementKind NS_AVAILABLE_IOS(7_0);

@end
```

#### UICollectionViewLayout (UIReorderingSupportHooks)

```
/** ----------------- 响应UICollectionview 更新 -------------------- */
/**
根据item在UICollectionView中的位置获取该item的NSIndexPath.

第一个参数该item原来的index path，第二个参数是item在collection view中的位置。
 在item移动的过程中，该方法将UICollectionView中的location映射成
相应 NSIndexPaths。该方法的默认是显示，查找指定位置的已经存在
的cell，返回该cell的NSIndexPaths 。如果在相同的位置有多个cell，
该方法默认返回最上层的cell。
*/
- (NSIndexPath *)targetIndexPathForInteractivelyMovingItem:(NSIndexPath *)previousIndexPath withPosition:(CGPoint)position NS_AVAILABLE_IOS(9_0);

/**
当item在手势交互下移动时，通过该方法返回这个item布局的attributes 。

默认实现是，复制已存在的attributes，改变attributes两个值，一个是
中心点center；另一个是z轴的坐标值，设置成最大值。
所以该item在collection view的最上层。子类重载该方法，可以按照自
己的需求更改attributes，
首先需要调用super类获取attributes,然后自定义返回的数据结构。
*/
- (UICollectionViewLayoutAttributes *)layoutAttributesForInteractivelyMovingItemAtIndexPath:(NSIndexPath *)indexPath withTargetPosition:(CGPoint)position NS_AVAILABLE_IOS(9_0);

/** ------------------------无效化布局------------------------- */
/**
params
* targetIndexPaths NSArray - 正在移动的Item的当前索引值。
* targetPosition   CGPoint - Collection View坐标系内的点，是Item可能拖动的点。
* previousIndexPaths NSArray - 正在移动的Item的之前的索引值。
* previousPosition CGPoint - Collection View坐标系内的点。这个点是之前用来确定Item拖动点的。

返回一个上下文对象，这个对象标示了在布局中被交互移动的Item。

在交互移动一个或多个Item的过程中，布局对象使用这个方法来获取无
效上下文。这个方法的默认实现是创建一个invalidationContextClass提
供的类的对象，使用提供的信息填入这个对象，并且返回这个对象。如
果你想要对你的布局使用一个自定义无效上下文对象，重写这个方法，
并返回你的自定义的类的实例。
子类化可以重写这个方法，使用这个方法在返回之前来执行额外的对无
效上下文的配置。在你的自定义实现中调用super，以便父类可以执行
对这个对象的基本配置。
*/
- (UICollectionViewLayoutInvalidationContext *)invalidationContextForInteractivelyMovingItems:(NSArray<NSIndexPath *> *)targetIndexPaths withTargetPosition:(CGPoint)targetPosition previousIndexPaths:(NSArray<NSIndexPath *> *)previousIndexPaths previousPosition:(CGPoint)previousPosition NS_AVAILABLE_IOS(9_0);

/**
params
* indexPaths NSArray - Item的最终索引值。对于取消的交互，这个索引值对应了Item的最初索引值。
* previousIndexPaths NSArray - Item之前的索引值。这个参数包含了Collection View在一系列移动中报告的最后一组索引值。
* movementCancelled BOOL - 标示了移动交互是成功结束还是被取消了。

返回一个上下文对象，这个对象标示了被移动的Item。

当交互移动一个或多个Item结束时（移动成功或者被用户取消了移
动），布局对象使用这个方法来获取无效上下文。这个方法的默认实现
是创建一个invalidationContextClass提供的类的对象，使用提供的信息
填入这个对象，并且返回这个对象。如果你想要对你的布局使用一个自
定义无效上下文对象，重写这个方法，并返回你的自定义的类的实例。
子类化可以重写这个方法，使用这个方法在返回之前来执行额外的对无
效上下文的配置。在你的自定义实现中调用super，以便父类可以执行
对这个对象的基本配置。
*/
- (UICollectionViewLayoutInvalidationContext *)invalidationContextForEndingInteractiveMovementOfItemsToFinalIndexPaths:(NSArray<NSIndexPath *> *)indexPaths previousIndexPaths:(NSArray<NSIndexPath *> *)previousIndexPaths movementCancelled:(BOOL)movementCancelled NS_AVAILABLE_IOS(9_0);
```

