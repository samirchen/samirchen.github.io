---
layout: post
title: Swift 中的闭包捕获语义
description: 与 Objective-C 的闭包相比，Swift 的闭包有着不同的捕获语义，本文借助一个例子来简要说明。
category: blog
tag: iOS, Objective-C, Swift, Closures Capture
---

本文内容是节选了 [Closures Capture Semantics, Part 1: Catch them all!][3] 的一个示例略作修改而来。

基于 Swift 3 语法的示例代码如下：

ViewController.swift


```swift
import UIKit

// MARK: - Pokemon Demo
class Pokemon: CustomDebugStringConvertible {
    let name: String
    var debugDescription: String {
        return "<Pokemon \(name)>"
    }
    init(name: String) {
        self.name = name
    }
    deinit {
        print("\(self) escaped!")
    }
}

func delay(seconds: Int, closure: @escaping ()->()) {
    let time = DispatchTime.now() + .seconds(seconds)
    DispatchQueue.main.asyncAfter(deadline: time) {
        print("🕑")
        closure()
    }
}

func demo() {
    var pokemon = Pokemon(name: "Mew")
    print("➡️ Initial pokemon is \(pokemon)")
    
    delay(seconds: 1) { [capturedPokemon = pokemon] in
        print("closure 1 — pokemon captured at creation time: \(capturedPokemon)")
        print("closure 1 — variable evaluated at execution time: \(pokemon)")
        pokemon = Pokemon(name: "Pikachu")
        print("closure 1 - pokemon has been now set to \(pokemon)")
    }
    
    pokemon = Pokemon(name: "Mewtwo")
    print("🔄 pokemon changed to \(pokemon)")
    
    delay(seconds: 2) { [capturedPokemon = pokemon] in
        print("closure 2 — pokemon captured at creation time: \(capturedPokemon)")
        print("closure 2 — variable evaluated at execution time: \(pokemon)")
        pokemon = Pokemon(name: "Charizard")
        print("closure 2 - value has been now set to \(pokemon)")
    }
}

// MARK: - ViewController
class ViewController: UIViewController {

    override func viewDidLoad() {
        super.viewDidLoad()
        
        demo()
    }

}
```

看看上面的代码，自己想想打印的结果会是什么？


下面是打印的结果：

```
➡️ Initial pokemon is <Pokemon Mew>
🔄 pokemon changed to <Pokemon Mewtwo>
🕑
closure 1 — pokemon captured at creation time: <Pokemon Mew>
closure 1 — variable evaluated at execution time: <Pokemon Mewtwo>
closure 1 - pokemon has been now set to <Pokemon Pikachu>
<Pokemon Mew> escaped!
🕑
closure 2 — pokemon captured at creation time: <Pokemon Mewtwo>
closure 2 — variable evaluated at execution time: <Pokemon Pikachu>
<Pokemon Pikachu> escaped!
closure 2 - value has been now set to <Pokemon Charizard>
<Pokemon Mewtwo> escaped!
<Pokemon Charizard> escaped!
```

跟你预想的是否一致？

下面是对上述代码的解释：

1. 开始将 `pokemon` 指向对象 `Mew`。
2. 创建闭包 1 并且它的新本地变量 `capturedPokemon` 捕获了 `pokemon` 指向对象的值（此时 `pokemon` 的值为 `Mew`）。闭包同时也捕获了 `pokemon` 这个引用，因为 `capturedPokemon` 和 `pokemeon` 都会在闭包中使用。
3. 然后将 `pokemon` 指向对象 `Mewtwo`。
4. 创建闭包 2，它的新本地变量 `capturedPokemon` 捕获了 `pokemon` 指向对象的值（此刻 `pokemon` 的值为 `Mewtwo`）。闭包同时也捕获了 `pokemon` 这个引用，因为 `capturedPokemon` 和 `pokemeon` 都会在闭包中使用。
5. 此时，demo() 函数执行完成。
6. 过了 1 秒钟后，GCD 开始执行第一个闭包：
	- 6.1 它的打印结果为 `Mew`，即第 2 步创建闭包时通过 `capturedPokemon` 变量捕获到的值。
	- 6.2 闭包同时根据所捕获 `pokemon` 的引用，找出它所指向对象的当前值，即 `Mewtwo`（这个是在第 5 步 demo() 函数执行完成时的 `pokemon` 指向对象的值）。
	- 6.3 然后将变量 `pokemon` 指向对象的值改为 `Pikachu`（闭包捕获的是变量 `pokemon` 的引用，所以 demo() 函数中的 `pokemon` 变量与闭包中进行赋值操作的 `pokemon` 变量是相同的引用）。
	- 6.4 当闭包执行完成被 GCD 释放后，没有人再强引用 `Mew` 这个对象了，因此它会释放掉。但是第二个闭包的 `capturedPokemon` 依然捕获着 `Mewtwo` 这个对象，并且第二个闭包也捕获了 `pokemon` 这个引用，此刻它的指向对象的值为 `Pikachu`。
7. 又过了 1 秒钟，GCD 开始执行第二个闭包：
	- 7.1 它的打印结果为 `Mewtwo`，即步骤 4 中第二个闭包创建时由 `capturedPokemon` 变量捕获到的值。
	- 7.2 它也会根据所捕获的 `pokemon` 这个引用，找出它指向对象的当前值，即 `Pikachu`（因为在第一个闭包中修改了它）。
	- 7.3 最后，将 `pokemon` 指向对象 `Charizard`，由于 `Pikachu` 此前只被 `pokemon` 变量强引用，而此时 `pokemon` 已不再指向它了，所以也会立即被释放。。
	- 7.4 当闭包执行完毕被 GCD 释放后，本地变量 `capturedPokemon` 脱离了作用域，所以它捕获并持有的对象 `Mewtwo` 会被释放，同时指向 `pokemon` 变量的强引用也会消失，`Charizard` 也会被释放。

在这里做一下闭包捕获语义的总结：

- 在 Swift 闭包中使用的所有外部变量，闭包会自动捕获这些变量的引用。这就像是在 Objective-C 中外部变量都用 `__block` 修饰了。
- 在闭包执行时，闭包会根据这些变量引用得到它当前指向的对象的具体值。
- 我们可以在闭包内部修改外部变量的值，因为闭包捕获的是变量的引用而不是引用指向的值，当然这种情况下这个外部变量要声明为 `var`，而不能是 `let`。
- 如果你想要捕获闭包创建时变量对应的值，可以使用带中括号的捕获列表，在其中将变量对应的值存储到闭包的本地常量中。



[SamirChen]: http://www.samirchen.com "SamirChen"
[1]: {{ page.url }} ({{ page.title }})
[2]: http://www.samirchen.com/closures-capture-cemantics-in-swift
[3]: http://alisoftware.github.io/swift/closures/2016/07/25/closure-capture-1/