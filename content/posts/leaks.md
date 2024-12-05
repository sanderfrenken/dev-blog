+++
title = 'About leaks when using SpriteKit'
date = 2024-12-03T00:04:03+02:00
draft = false
type = "post"
showTableOfContents = true
+++

## Efficiency

As with any computer program you are writing, memory management is essential. You don't want to keep references to instances that you don't need anymore, occupying memory and maybe computational power lost to the void. 

At the same time, many applications won't require much memory or processing power, especially taken into account the hardware of current generation phones and tablets. 

For example, running a somewhat more complex CRUD application on an iPhone 11 consumes about 40%CPU (one core, for 40% occupied) at it's highest point (out of 6 cores, so there is a total CPU of 600% in total available) and 20MB of working memory (of the available 4GB).

In such a case, even though inefficient memory usage is still as waste of energy, you might never notice any real issues when using the application (I don't say that you should not care about your footprint when developing a CRUD application though, you always should).

{{< figure src="../../images/screenshot-crud-cpu.png" width="800" title="Example of CPU usage of a CRUD application">}}

{{< figure src="../../images/screenshot-crud-ram.png" width="800" title="Example of RAM usage of a CRUD application">}}

*It's important to note that the above screenshots are taken from a simulator. That obviously has an impact on the data shown: there is no phone available at the moment with 32GB of memory or a 12 core CPU. For the purpose of this demo using the simulator suffices, but when assessing your app's performance/ memory usage, it's essential to use a real physical device.*

When making games however, you most likely require a lot more from your hardware, making it even more essential to manage your memory and processing power in the best way possible.

In this post I will talk about how to assess your app's performance, some easy to make errors, and how to fix those.

The examples used in this post most likely are already known to Swift developers that have been working with iOS/ OSX on non SpriteKit apps as well. However, those new to Swift dipping into SpriteKit as the first Swift experience might be unaware of the issues introduced in the examples. 

In any case, I hope this post is informative for both audiences.

## Tooling

There is a lot more to Xcode than you might be aware of. Next to it being an IDE, there is a variety of tooling available that allows you to inspect your application. For example debugging view hierarchies, memory graphs and a plethora of instruments to profile your application on different levels are just some of these capabilities. 

Inside Instruments, part of the Developer Tools, you find a variety of templates which can be used to inspect certain aspects of your application. It's incredibly powerful, but also hard to interpret and not always very well documented in my opinion.

{{< figure src="../../images/screenshot-dt-instruments.png" width="400" title="Here you can find Instruments in Xcode">}}

Instruments and all it's templates deserves its own blog post, which I will cover in later posts. I will cover the File Activity template in this blog though.

### Debug navigator

An easy to use and to interpret overview can be found in the left side bar of the Xcode IDE. It's called the Debug navigator.
When you are debugging your application this is where you will find the current threads and their current call stack.

{{< figure src="../../images/screenshot-debug-navigator-call-stack.png" width="800" title="The Debug navigator in Xcode">}}

If you are not debugging your application, you can of course still navigate to the Debug navigator, and it will show you some statistics of your running application.

{{< figure src="../../images/screenshot-debug-navigator.png" width="800" title="The Debug navigator focussed on the current CPU profiling">}}

As you can see in this example, we are looking at an overview of the current CPU usage by our running application. Especially the graph showing the CPU usage over time is interesting: Even though the application might not be using any significant CPU at this moment in time, there have been spikes which might require your attention. 

Scrolling further down you will see CPU usage per thread, which could be on interest as well if you are running multithreaded logic. For instance, if you try to spread heavily computational tasks over multiple threads, you would like to see an even distribution of CPU consumption per thread, and ideally the threads not waiting for one another.

You can also inspect your memory consumption, disk operations, networking and GPU performance using the Debug navigator.

### Debug memory graph

A very useful tool available from Xcode's main editor panel is the Debug memory graph functionality. At any moment in time, you can use this tool to assess the current memory graph. This allows you to assess all the memory your application is currently allocating. 

You can request the memory graph when your app is running by tapping the green-circled icon as shown in the below image.

{{< figure src="../../images/screenshot-memory-graph.png" width="800" title="The memory graph determined for the current application">}}

What you can see very nicely in this overview is not only which allocations are made, but also what references are existing between your objects in memory. On the top right of the graph view you will also find yellow warnings signs, which warn you about leaked objects. 

Leaks are objects that occupy memory which are no longer needed but can not be released.

These leaks can occur at certain points in your code, over and over again. This would eventually cause so much memory to be consumed, that the OS decides to terminate your program. Before doing so, the OS will always warn you to release as much memory as possible by invoking the method [didReceiveMemoryWarning()](https://developer.apple.com/documentation/uikit/uiviewcontroller/didreceivememorywarning()). 

But eventually, if you don't resolve the leak, your applications is destined to be terminated (this is also one of the causes of [software aging](https://en.wikipedia.org/wiki/Software_aging#Memory_leaks)).

### Debug view hierarchy

Finally, also available from Xcode's main editor panel is the Debug view hierarchy functionality.

{{< figure src="../../images/screenshot-view-hierarchy.png" width="800" title="The view hierarchy visualized from the current application's state">}}

This allows you to inspect the views that are currently rendered. This can be handy to identify everything that is drawn on the screen, and potentially spot duplicated views or invisible views that should even be there in the first place. When building games however the hierarchy can quickly become very crowded, but it might be helpful at some occasions.

## Demo project

Now, for the sake of this blog post, I created a demo project that can be found [here](https://github.com/sanderfrenken/spritekit-demo-leaks). It's probably convenient to check it out and run it as well, so that you can also see it in action.

It's a very basic project, where there is a main scene (`GameScene`), from where you can navigate to a couple of demo scenes and from there back to to the main scene.


### Capturing self in closure

For this leak example, we will look at the scene `DemoSelfInClosureScene`. This scene is rendering a label and a couple of circles. It's also running an action, where it invokes a closure every second. In the closure it's calling an internal method named `printHello()`, in which the rendered label is made transparant and subsequently fades back in. The circles are only added to increase the memory footprint of the scene, making the issue more easily to visualize.

``` Swift
import SpriteKit

class DemoSelfInClosureScene: BaseScene {

    let label = SKLabelNode(text: "Hello world")


    override func didMove(to view: SKView) {
        super.didMove(to: view)

        addCircles()

        label.alpha = 0
        label.fontSize = 30
        addChild(label)

        let actions: [SKAction] = [
            .run {
                self.printHello()
            },
            .wait(forDuration: 1)
        ]
        run(.repeatForever(.sequence(actions)))
    }

    private func printHello() {
        label.alpha = 0
        label.run(.fadeIn(withDuration: 0.5))
    }

    override func touchesBegan(_ touches: Set<UITouch>, with event: UIEvent?) {
        super.touchesBegan(touches, with: event)
    }
}
```

Run the app, and then navigate to the scene by tapping `leakSelf`. You should see a flickering label with the text `Hello world`. Now, navigate back to the previous scene and then again back to the `leakSelf` scene. Repeat this for a couple of times.

While doing the above, open the Debug navigator as mentioned earlier, and select the memory tab. Look at the overtime graph, and what you should notice is an increasing use of memory.

{{< figure src="../../images/screenshot-memory-increase.png" width="800" title="The memory graph is showing an increase in memory consumption over time.">}}

This steplike pattern is a clear indicator something is wrong in our code. There seems to be memory consumed each time the `DemoSelfInClosureScene` scene is rendered, which is not released when we return to main scene.

At this point, it's wise to have a look at the memory graph. Open the memory graph and this is what we will see:

{{< figure src="../../images/screenshot-memory-increase-graph.png" width="800" title="The memory graph, where I selected one of the leaked instance and highlighted the referencing allocation.">}}

The first thing to look at is the left panel. Here you see the currently allocated objects:

- 1 instance of `AppDelegate` as expected
- 1 instance of `GameViewController` as expected
- 0 instances of `GameScene` (you might see 1, depending on which scene you were at when you created the memory graph)
- 7(!) instances of `DemoSelfInClosureScene` (you might see another number, depending on how often you opened and returned from the scene you were at when you created the memory graph)

This is clear indication that the `DemoSelfInClosureScene` scene is not freed from memory when you move away from it. You might see the yellow warning sign next to one or more allocated instances of this scene in the memory graph. Looking at the overall leaks detected, we see a large amount of allocations that are not released:

{{< figure src="../../images/screenshot-memory-increase-warnings.png" width="800" title="All the leaked allocations detected.">}}

There are many leaked objects, including SKLabelNode, SKShapeNode and CGPath instances. These are all related as we know to our `DemoSelfInClosureScene` (the CGPath instances are related to our SKShapeNode instances which we initiate as a circle shape.)

In the next screenshot, I selected one of the leaked instances of `DemoSelfInClosureScene`. Looking at the graph drawn for this instance, you are able to assess the reference that is keeping this instance from being released from memory.

{{< figure src="../../images/screenshot-memory-increase-graph-closure.png" width="800" title="A closure is still referencing this instance of our DemoSelfInClosureScene.">}}

It's here that you can see that a closure (`Closure context`) is capturing the scene, preventing it from being released.

There is only one closure in our code, and it's located at the `SKAction.run` invocation here:

``` Swift
let actions: [SKAction] = [
    .run {
        self.printHello()
    },
    .wait(forDuration: 1)
]
```

The problem with this code is that we are capturing self in the closure, creating a strong reference cycle. The problem is now that when you move away from the `DemoSelfInClosureScene`, the closure still captures self, preventing self from be released from memory. 

Luckily we can easily fix this by changing the code to: 

``` Swift
let actions: [SKAction] = [
    .run { [weak self] in
        self?.printHello()
    },
    .wait(forDuration: 1)
]
```

By creating a weak reference here, the closure will not retain self anymore. 

Perform the change, and you should not see the memory build-up anymore when navigating back and forth to the `DemoSelfInClosureScene` and no increasing amounts of `DemoSelfInClosureScene` instances.

{{< figure src="../../images/screenshot-memory-increase-graph-fixed.png" width="800" title="The memory graph, now healthy!">}}

*You could instead of using the memory graph to inspect this issue also made use of the Leaks template in Instruments. I will cover that in another post later.*

### Circular references

For another leak example, we will look at the scene `DemoSelfInPropertyScene`. This scene contains a `Spawner` node and again draws a couple of circles, only added to increase the memory footprint of the scene. 

The `Spawner` is a class that the scene will use to delegate the spawning of a label to. 

You can see the spawner as an object that is responsible for rendering certain nodes to the scene (parentScene) it is initialized with. When the scene is initialized, so is the spawner. 

The scene will inject it self into the spawner, and then instructs the spawner to render the label.

``` Swift
import SpriteKit

@MainActor
final class Spawner {

    let parentScene: SKScene

    init(parentScene: SKScene) {
        self.parentScene = parentScene
    }
    
    required init?(coder aDecoder: NSCoder) {
        fatalError("init(coder:) has not been implemented")
    }

    func addLabel() {
        let node = SKLabelNode(text: "Hello, World!")
        node.alpha = 0
        parentScene.addChild(node)
        node.run(.repeatForever(.sequence([.fadeIn(withDuration: 0.5), .fadeOut(withDuration: 0.5)])))
    }
}
```

``` Swift
import SpriteKit

class DemoSelfInPropertyScene: BaseScene {

    var spawner: Spawner?

    override func didMove(to view: SKView) {
        super.didMove(to: view)

        addCircles()
        
        spawner = Spawner(parentScene: self)
        guard let spawner else { return }
        spawner.addLabel()
    }

    override func touchesBegan(_ touches: Set<UITouch>, with event: UIEvent?) {
        super.touchesBegan(touches, with: event)
    }
}

```

Run the app, and then navigate to the scene by tapping `leakProperty`. You should see a flickering label with the text `Hello world` now. Now, navigate back to the previous scene and back to the `leakProperty` scene. Repeat this for a couple of times.

Repeating the steps as done in the previous exercise, you will notice an increase in memory consumption again when moving back and forth to this scene, visualized by the steplike increase in the memory graph.

{{< figure src="../../images/screenshot-memory-increase-property.png" width="800" title="The memory graph, again with the steplike shape">}}

In this example, we again use the Debug memory graph to see what is causing this steady increase in memory consumption. 

Looking at the graph, we see that after a couple of times of going back and forth, the amount of instances of `DemoSelfInPropertyScene` increases at the same rate. 

Moreover, we see the same increase in instances of the `Spawner` type.

{{< figure src="../../images/screenshot-memory-increase-graph-property.png" width="800" title="The memory graph after a couple of going back and forths">}}

If we zoom in to one of the leaked instances of `DemoSelfInPropertyScene`, we see its being retained by a strong reference made by `Spawner`, and vice versa: `Spawner` is retaining `DemoSelfInPropertyScene` as well (see above screenshot). This is a called a circular reference, and it happens when an amount of objects are strongly referencing each other.

We can break this however by creating a weak reference to the scene inside `Spawner` instead of strong reference.
Instead of:

``` Swift
let parentScene: SKScene
```

We use:

``` Swift
weak var parentScene: SKScene?
```

This will allow the parentScene to deallocate when it needs to, as `Spawner` will not reference it strongly.
Once you made this change, you will see no more memory build-up overtime.

*In Swift delegation implementations, you are advised to do the same: always reference a delegate using a weak reference, to prevent strong reference cycles as explained here in the [Swift documentation](https://docs.swift.org/swift-book/documentation/the-swift-programming-language/protocols/#Delegation).*

### Open file descriptors

In this final example, I like to point out something I came across recently.

With SpriteKit, one can easily make nodes play certain sounds using the action [playSoundFileNamed(_:waitForCompletion:)](https://developer.apple.com/documentation/spritekit/skaction/1417664-playsoundfilenamed).

This is an action that will open the file requested and the node will run an action that plays this sound. 
For instance, we can look at a simplified version of the `DemoPlaySoundEffectScene`. 
You see that a simple SKNode instance can run an action to play the sound:

``` Swift
import SpriteKit

class DemoPlaySoundEffectScene: BaseScene {

    let soundNode = SKNode()

    override func didMove(to view: SKView) {
        super.didMove(to: view)
        addChild(soundNode)
    }

    override func touchesBegan(_ touches: Set<UITouch>, with event: UIEvent?) {
        super.touchesBegan(touches, with: event)
        soundNode.run(.playSoundFileNamed("steelHitSteelLight1", waitForCompletion: false))
    }
}
```

There is a slight issue with this API however. It can be a bit costly to invoke `playSoundFileNamed` the first time, as it will need to open and read the file on disk and play it. As this is happening on the main actor, in practice you may encounter short framedrops when playing a sound if it wasn't cached before.

One way to work around this would be to cache the actions up front, so that they can be played directly from memory:

``` Swift
import SpriteKit

class DemoPlaySoundEffectScene: BaseScene {

    let soundNode = SKNode()

    private var audioActions = [SKAction]()

    override func didMove(to view: SKView) {
        super.didMove(to: view)
        for index in 1...8 {
            audioActions.append(.playSoundFileNamed("steelHitSteelLight\(index)", waitForCompletion: false))
        }
        addChild(soundNode)
    }

    override func touchesBegan(_ touches: Set<UITouch>, with event: UIEvent?) {
        super.touchesBegan(touches, with: event)
        guard let audioAction = audioActions.randomElement() else { return }
        soundNode.run(audioAction)
    }
}
```

I have been doing this in my game Battledom, and there are many sound files that I have in there, at one point more than 256. When I reached that limit, I suddenly got runtime crashes in my game when I tried to open a file from the bundle, stating there were too many open files. 

It wasn't clear to me immediately what was going on (and the role of the sound effects in this issue), so I used the File Activity template in Instruments to profile my application in order to determine what was causing this large amount of open files.

{{< figure src="../../images/screenshot-file-activity.png" width="800" title="The File Activity template in Instruments">}}

For this demo, I reproduce the issue on a smaller scale inside `DemoPlaySoundEffectScene` so you can follow along. In Xcode, open Instruments and then select the File Activity template.

Then, select your run destination and application (demo-leaks) on the top of the screen, hit the red record button. 

{{< figure src="../../images/screenshot-file-activity-input.png" width="800" title="Setup the File Activity recording">}}

Now, the app will start and you will see the start scene. Tap the `soundEffect` button and when in the new scene, tap a few times on the screen to hear the sounds being played. Then, hit the record stop button and wait a bit while Xcode is analyzing the data streams.  

The result once there shows a couple of tracks. For this demo, we select the track `Filesystem Activity`. Then, in the selector above the tracings, you can select `File Descriptor History` instead of `Filesystem Statistics`. You will now see an overview of all files that were opened and closed during the lifetime of the recording.

{{< figure src="../../images/screenshot-file-activity-output.png" width="800" title="Output of the File Activity recording">}}

If you scroll through the list (you can sort on Path first), you will notice that the sound effects were indeed opened at one moment in time (Create File Descriptor). However, you will also see they were never closed (there is no Close File Descriptor). 

That means that as long as you keep the sounds in memory, the files remain open. I am not even sure if this might be a bug in SpriteKit. If you have any thoughts on this, please leave a comment below!

NB, I did find an old SO post [here](https://stackoverflow.com/questions/27748818/playsoundfilenamed-is-leaving-the-sound-files-open-using-up-file-descriptors-an) that describes a similar problem. Getting the feeling this truly is a bug in SpriteKit.

## Wrap up

I hope you enjoyed and maybe learned something from this post. There is a lot more to tell about leaks and related problems, especially with regard to the tooling that you can use to assess it. 

If you are interested in the origin of reference cycles and want to understand why they occur more thoroughly, I advise you to read this [article on the Swift docs](https://docs.swift.org/swift-book/documentation/the-swift-programming-language/automaticreferencecounting/). There is a good explanation on how automatic reference counting works, which lies at the base of Swift's garbage collection.

I will explore instruments further in future posts, demonstrating their usage by examples.

But for now I would like to emphasize the following learnings:

- Memory allocations and CPU usage are essential to assess during your development process. Do it frequently to avoid large refactors later on.

- Avoid capturing self strongly in closures

- Avoid strong circular reference cycles by using `weak var`

- To assess memory issues, you can use the Memory graph and the Debug navigator, there is no need to resort to Instruments immediately

- Open file descriptors can be assessed by using the File Template in Xcode instruments

- Be cautious when caching many `playSoundFileNamed(_:waitForCompletion:)` action instances 

Next time, I will write about playing sounds in SpriteKit.

Do you have any questions, suggestions or remarks for this post? 

Please leave a message below!

{{< youtube 0y0pernS0Gc >}}