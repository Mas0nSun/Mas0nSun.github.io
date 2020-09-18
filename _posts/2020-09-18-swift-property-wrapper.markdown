---
layout: post
title:  "Swift Property Wrapper"
date:   2020-09-18 11:55:00 +0800
categories: Swift
---

## Property Wrapper

刚接触`SwiftUI`时, 看到这些东西

`@State, @Binding, @Published, @ObservedObject, @Environment`

是不是一脸懵逼, 但是用起来的时候却额外的丝滑, 比如当我们想要动态的更新View的布局时可以这么写:

```swift
struct ContentView: View {
    // 1
    @State var isSearching: Bool = false

    var body: some View {
        // 2
        if isSearching {
            Text("Hello, world!")
                .padding()
        } else {
            Button("Search") {
                // 3
                isSearching = true
            }
        }
    }
}
```

- 1: 用@State修饰一个属性, 当我们更改它时`ContentView`会更新布局

- 2: 添加判断条件, 如果`isSearching == true`那么展示`Text`, 否则展示`Button`

- 3: 当点击按钮时更改`isSearching`的值

是不是非常丝滑? 我很长一段时间都把这些东西当作关键字来理解, 因为我只需要知道他们各自的作用就好了, 直到我看到官方的文章才了解什么是

[Property Wrapper](https://github.com/apple/swift-evolution/blob/master/proposals/0258-property-wrappers.md) 🌚, 其他的`Property Wrapper`这里不在举例, 我们重点讨论`Property Wrapper`是什么.



官方给出的解释是: 有一些属性实现模式会重复出现, 比如`lazy`, `@NSCopying`, 但他们不想每次都用硬编码的模式将这个关键字添加到编译器中, 所以提出了`Property Wrapper`机制.

比如下面懒加载的实现:

```swift
class Person {
    lazy var name: String = "Mas0n"
}
```

等同于

```swift
class Person {
    private var _name: String?
    var name: String {
        get {
            if let value = _name { return value }
            let initialValue = "Mas0n"
            _name = initialValue
            return initialValue
        }
        set {
            _name = newValue
        }
    }
}
```

苹果之前将`lazy`放到了`Swift`的语言中, 这样有很多缺点, 比如会让`Swift`和编译器变得复杂, 并且缺少灵活性, 比如当有更多的像`lazy`这样的变种实现, 他们不想把这些硬编码全都放到`Swift`中, 比如想要实现一个可以重制`lazy`属性的`lazyReset`.



因此Apple提出了`Property Wrapper`的解决方案:

我可以自己实现一个`lazy`关键字通过`Property Wrapper`

```swift
// 1
@propertyWrapper
enum Lazy<Value> {
    case uninitialized(() -> Value)
    case initialized(Value)

    // 2
    init(wrappedValue: @autoclosure @escaping () -> Value) {
        self = .uninitialized(wrappedValue)
    }

    var wrappedValue: Value {
        mutating get {
            // 3
            switch self {
            case .uninitialized(let initializer):
                let value = initializer()
                self = .initialized(value)
                return value
            case .initialized(let value):
                return value
            }
        }
        set {
            // 4
            self = .initialized(newValue)
        }
    }
}
```

- 1: 用`@propertyWrapper`声明一个struct/enum/class, 需要注意的是我们必须实现一个非`static`类型的变量`wrappedValue`
- 2: 实现`init`方法, 这里采用`@autoclosure`将传入的参数自动包装成闭包的形式, 为了当调用时才去获取真正的值(延迟执行), 想要了解更详细的内容可以看这里[@autoclosure](https://docs.swift.org/swift-book/LanguageGuide/Closures.html)
- 3: 重写`get`方法, 当我们外部调用被`@Lazy`声明的属性时, 实际上是调用的`wrappedValue`的`get`方法. 所以这里做判断, 当`self`是没有初始化的时候会去调用初始化时传入了`@autoclosure`的值
- 4: `set`方法会直接初始化被`@Lazy`声明的属性

使用:

```swift
@Lazy var name = "Mas0n"
// or
@Lazy(wrappedValue: "Mas0n") var name
```

- 这里需要说明一下, 当使用`@propertyWrapper`声明属性时, 我们无论是调用`name`的`set`还是`get`方法, 其实都是在调用`wrappedValue`的`set`和`get`, `_name`的类型才是真正的`enum Lazy`, 因此也可以通过`_name.wrappedValue`来访问`name`的值

这样我们自定义实现的`Property Wrapper`就完成了, 它的效果等同于`Swift`语言中的`lazy`, 是不是很方便呢? 这样当我们想要再次实现一个类似于`lazy`的关键字的时候, 就不必再去为`Swift`添加硬编码了. 

`Property Wrapper`还提供了可选的`projectedValue`属性, 类似于`wrappedValue`, 当我们实现了`projectedValue`时, 我们可以通过`$`在外部快速的访问一个`projectedValue`



所以就诞生了一开始`SwiftUI`中的`@State, @Binding, @Published, @ObservedObject, @Environment`, 他们本质山都是Apple用`Property Wrapper`定义的`struct/enum/class`

我们可以推测一下`@State`的实现:

```swift
@propertyWrapper
struct State<Value>: DynamicViewContent {
    init(wrappedValue: Value) { }
    var wrappedValue: Value { }
    var projectedValue: Binding<Value> { }
}
```

用法:

```swift
@State var name: String
// 1
_name = State(wrappedValue: "Mas0n")
// 2
name = "Mas0n"
// 3
$name.wrappedValue = "Mas0n"
```

- 1: 为结构体`State`类型的`name`初始化
- 2: 访问`name`的`wrappedValue`
- 3: 访问`name`的`projectedValue`



### 总结

其实`Property Wrapper`的作用就是可以帮助我们封装模版代码, 简化重复的代码, 并且提供了`wrappedValue`和`projectedValue`方便我们访问自定义的`get`和`set`实现.

以上文章参考苹果官方的资料[SE-0258](https://github.com/apple/swift-evolution/blob/master/proposals/0258-property-wrappers.md)

更详细的内容可以去链接里查看
