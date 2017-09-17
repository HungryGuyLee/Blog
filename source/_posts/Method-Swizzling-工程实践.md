title: Method Swizzling 工程实践
date: 2017-09-17 11:26:24
tags: iOS
---
在14年的时候看程序媛念茜的博客[Objective-C的hook方案（一）: Method Swizzling](http://blog.csdn.net/yiyaaixuexi/article/details/9374411)知道了OC中的黑科技Method Swizzling，但当时只是当做一个小知识点并不知道在实际工程中哪里能用到。而且对双刃剑及黑魔法这两个描述这项特性的关键字印象深刻，事后证明在没有了解清楚之前没有贸然在工程内使用是一个正确的选择。 </br></br>

本质就是交换SEL和IMP的映射关系,简单的理解就是截胡然后再回去。这样可以无侵入式的在原函数执行前执行一些自己想要的东西。比如ViewController中系统会自己调用`viewDidLoad`名字(SEL)用A代替，实现(IMP)用AI来代替。我自己写了一个函数叫`rock_viewDidLoad`，同理名字用B，实现用BI代替，原先的对应关系是A->AI B->BI 我通过swizzling把对应关系变成了A->BI B->AI那么当系统调用`viewDidLoad`时候其实是调用了`rock_viewDidLoad`，而`rock_viewDidLoad`的最后一句是`[self rock_viewDidLoad]`看似是递归但因为已经交换过了所以其实这句调用的是`viewDidLoad`
</br></br></br>如今看了一些实践的例子，决定在以下几个地方用 </br></br>
1.Appdelegate瘦身</br></br>
2.NSArray和NSDictionary的健壮性完善上</br></br>
~~3.使用友盟统计的数据埋点~~</br></br>
念茜的微博原理讲得很清楚了，但是我会补充几点方便理解</br></br>

先贴核心代码

    + (void)swizzleInstanceMethodWithClass:(Class)class 
                          originalSelector:(SEL)originalSelector 
                            swizzledMethod:(SEL)swizzledSelector {
        
        Method originalMethod = class_getInstanceMethod(class, originalSelector);
        Method swizzledMethod = class_getInstanceMethod(class, swizzledSelector);
    
        if (class_addMethod(class, originalSelector, method_getImplementation(swizzledMethod), method_getTypeEncoding(swizzledMethod))) {
            class_replaceMethod(class, swizzledSelector, method_getImplementation(originalMethod), method_getTypeEncoding(originalMethod));
        }else {
            method_exchangeImplementations(originalMethod, swizzledMethod);
        }
    }
    
    + (void)load {
        static dispatch_once_t onceToken;
        dispatch_once(&onceToken, ^{
        [self hook];
        });
    }
    
    + (void)hook {
        //使用swizzleInstanceMethodWithClass:(Class)class originalSelector:(SEL)originalSelector swizzledMethod:(SEL)swizzledSelector
        //按照实际需求进行hook
    }

<font size = 8>1.为什么在`+ (void)load`里调用？</br></font>
只要调用<font color=#7FFF00 size=10>一次</font>且确保 <font color=#7FFF00 size=10>一定</font> 而且在<font color=#7FFF00 size=10>早期</font>调用当然也<font color=#7FFF00 size=10>不能影响父类</font>，
那么满足这四个条件的非这个函数莫属。</br></br>
有人会问那为什么不在`+ (void)initialize`里调用呢？</br>
可以结合我提的四个条件去看这两篇文章</br>
[细说OC中的load和initialize方法](http://www.jianshu.com/p/d25f691f0b07)</br>
[Objective-C 深入理解 +load 和 +initialize](http://www.jianshu.com/p/872447c6dc3f)

引用第二篇文章的表格先给出结论

 |               | +load            | +initialize   |
 | -------------  |------------- | ----- |
 | 调用时机       | 被添加到 runtime 时  | 收到第一条消息前，可能永远不调用  |
 |调用顺序       | 父类->子类->分类       |   父类->子类  |
 | 调用次数  |1次?   |    多次  |
 |是否需要显式调用父类实现 |否 |否 |
 |是否沿用父类的实现 |否 |是 |
 |分类中的实现 |类和分类都执行 |覆盖类中的方法，只执行分类的实现 |

1.不能保证一次</br>
2&3.是懒加载如果不给此类或者此类子类发消息可能永远不会调用</br>4.会影响父类

<font size=8>2.为什么要使用`dispatch_once`</font></br>
因为要防止有人手动调用`+ (void)load`

<font size=8>3.为什么不直接使用`OBJC_EXPORT void method_exchangeImplementations(Method m1, Method m2) 
    OBJC_AVAILABLE(10.5, 2.0, 9.0, 1.0);`而是要多一层使用`OBJC_EXPORT BOOL class_addMethod(Class cls, SEL name, IMP imp, 
                                 const char *types) 
    OBJC_AVAILABLE(10.5, 2.0, 9.0, 1.0);`判断</font></br></br>

有没有这么一种情况呢？就是你想要交换的原函数在你想要交换的类里没有实现而是在父类里实现呢？那这样直接交换就会影响父类的其他子类，这显然不是我们想要的。</br>
然后再来看下官方给`OBJC_EXPORT BOOL class_addMethod(Class cls, SEL name, IMP imp, 
                                 const char *types) 
    OBJC_AVAILABLE(10.5, 2.0, 9.0, 1.0);`的注释</br></br>

    /** 
    * Adds a new method to a class with a given name and implementation.
    * 
     * @param cls The class to which to add a method.
     * @param name A selector that specifies the name of the method being added.
     * @param imp A function which is the implementation of the new method. The function must take at least two arguments—self and _cmd.
     * @param types An array of characters that describe the types of the arguments to the method. 
     * 
     * @return YES if the method was added successfully, otherwise NO 
    *  (for example, the class already contains a method implementation with that name).
    *
    * @note class_addMethod will add an override of a superclass's implementation, 
    *  but will not replace an existing implementation in this class. 
    *  To change an existing implementation, use method_setImplementation.
     */
     
没错如果这个函数返回YES说明子类没有实现这个方法,那么需要进行特殊处理。直接增加一个空的实现并且两步交换selector和IMP的映射关系。</br>

下面再具体说说</br>
1.Appdelegate瘦身</br></br>
可以hook`application:didFinishLaunchingWithOptions:`以及一系列生命周期的方法。将一系列第三方库的使用抽离原来的Appdelegate写在category里。需要注意的是`application:didFinishLaunchingWithOptions:`最后需要使用`return [self rock_application:application didFinishLaunchingWithOptions:launchOptions];`回到原先的函数里继续</br></br>
2.对`ObjectAtIndex` `replaceObjectAtIndex:withObject:``insertObject:atIndex:`涉及到下标的数组函数进行hook加入越界判断防止崩溃</br>
对字典`setObject:forKey:`hook对键、值为空的情况进行处理

谢谢大家
    


