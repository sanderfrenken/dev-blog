+++
title = 'Back in action. On Battledom, pathfinding and AI tools'
date = 2025-03-19T00:04:03+02:00
draft = false
type = "post"
showTableOfContents = true
+++

## Back to blog

It has been almost four months since my last post, which is way too long ago.
Not that I have been sitting still. Obviously as most of you can also relate to, December is a fully packed month with little free time. Especially for us now with two little children that have long holidays, December was a lost cause when it comes to game development.

But in January and February I picked up the pace again with development of Battledom. I am currently in an early beta phase and have about 150 active testers. I am still very eager to get more feedback, so if you are interested please join via [TestFlight](https://testflight.apple.com/join/IsXcGtGR).

I added a lot of new functionality and content over the last months and spend time revisiting a couple of mechanics. Most importantly, I needed to revise the pathfinding algorithm for reasons I will clarify further in this post.

Moreover, I also made a new trailer for Battledom that you can find here:

{{< youtube ZecfzsuEKZw >}}

## Promo material

Even though the new trailer is just 30 seconds of material, it took way longer to create. There are many steps in this process, and as having proper material is quite essential for your game's success, you want to do them all right. Creating screenshots, videos and other promo material is definitely not my strongest point, but I have to admit I am happy enough with how this trailer turned out. I have some learnings from this process which I like to share as well. 

### Tools and theory

First of, you need to capture the screens and audio from some device. You can either use a physical device for this, or record from your simulator. Personally, I used to the simulator as I could then also directly record the audio using QuickTime. Plus, I also didn't have an iPad at my disposal to do a physical recording. I preferred to use an iPad because of the screen ratio.

The screen itself can be recored by hitting `CMD+R` while focussing on the simulator. 
The recorded file can also be saved as a GIF right away, which might be an interesting feature in some cases as well.

So recording video and sound can be done from your Apple development device without too much hassle. If you do have an iPad at your disposal, it might be worth trying that out for screen capturing as it could have better performance than your Mac simulator, and potentially a higher resolution.

But then there is more. You probably will write some form of script of what you like to show in your trailer. And this an important part. An ideal trailer has a length between 30-90 seconds, so you dont have much time to explain or build up a story line. Moreover, an App Preview in the AppStore has a max length of 30 seconds, so its already a good exercsie to make your trailer not much longer than those 30 seconds either.

In a trailer, you wish to capture the attention of the viewer immediately. As such, you could opt for starting the trailer in the middle of the action, then tune the action a bit lower, and in the end "go out with a bang". Its also essential to show the name of the game and communicate clearly the gameloop. Don't show things your game doesn't deliver.

I found some nice guidelines and explanations on how to make nice trailers on YouTube. I can really recommend watching the following video:

{{< youtube 4CSYA9R70R8 >}}=

With the theory absorbed and the recording tools identified, its time to create the raw content!

### Developer tools

You probably will notice soon enough when recording your material that you need to enforce certain behaviors. For instance, you might want to hide certain UI elements to create a cleaner interface for the trailer. Moreover, you dont want to be limited when creating/ setting up the recording scenes by the normal game loop. For instance, in Battledom you can only create units if you have enough resources. But while I was creating content for the trailer, I didn't wanted to be bothered by this.

Usually you already encounter this challenge when developing the game in general. You want an easy way to spawn units, jump to certain scenes, enable/ disable certain UI elements, etc.
Therefore, it's advisable to facilitate this from an early stage of development.

What I currently do is to define a set of variables (booleans) that I can toggle in Debug mode. I put them inside compiler flags, which will guarantuee that toggling these variables will only impact Debug builds.

``` Swift
#if DEBUG
let showStatistics = true
let showFogOfWar = true
let showPathing = true
let forceTutorialsShown = false
let forceAllWeaponsUnlocked = false
let forceImmediateWin = false
let forceItemLocked = false
let hideUI = false
let infiniteEndlessResources = true
#else
let showStatistics = false
let showFogOfWar = true
let showPathing = false
let forceTutorialsShown = false
let forceAllWeaponsUnlocked = false
let forceImmediateWin = false
let forceItemLocked = false
let hideUI = false
let infiniteEndlessResources = false
#endif
```

Inside your code you can these use these booleans to steer certain behaviors. Of course there are many other ways to facilitate this. You could also inject the "Debug" code directly within compiler flags inside the code where it is needed and omit the booleans, but personally I find these booleans a bit more readable and the ease of toggling is worth a lot.

You could even add an hidden entrypoint into your game from where you can live toggle these settings. I am not sure if I would advise this though, as obviously some of these toggles can be quite game-breaking.

### Editing

Once you have gathered all the right audio and video material, there comes the final step: Editing the videos and images. For this there is also a plethora of tools available, where the most accessible one is most likely iMovie, as that is already installed on your Mac if you have one. However, even though iMovie is definitely an easy to use tool, the downside is that it is quite limited. Transitions and animations, the basics are there, but I couldn't produce what I aimed for with just iMovie.

After some searching and trials, I came across [Canva](https://www.canva.com). Canva has a proper web editor and supports creating videos and images in many different formats. There are options to create your own branding using default fonts and colors you pick that you can then apply to all your projects. There are lots of transitions and animations available, which you can customize easily. 

Moreover, as with most tools nowadays, there are many AI features you can use to generate specific content, or alter existing content. I didn't dig into this, but for sure I can say that Canva does deliver a lot of nice features and it is very easy to use.

## Pathfinding on grids

Now on to a very different topic; Pathfinding. Pathfinding has been on my radar on and off over the last year of development of Battledom. In Battledom I make use of normal grid layed out maps, where most scenes I made so far consisted of approx. 150*150 tiles. 

I wrote about MSKTiled earlier in another [blogpost](https://sanderfrenken.github.io/dev-blog/msktiled/). But in a nutshell my workflow is as follows. I design my maps using a very popular tile editor being [Tiled](https://www.mapeditor.org). Such a map can consist of many layers, but for this example I use a simple map with just two layers. A base layer, which holds the ground (in this case, grass), and an obstacle layer. The obstacle layer has many empty tiles as well. The filled tiles are water and graves. These tiles should not be traversable in the game.

{{< figure src="../../images/screenshot-tiled-simple-map.png" width="800" title="Simple map; Just two layers being a base and an obstacle layer">}}

{{< figure src="../../images/screenshot-tiled-simple-map-obstacles.png" width="800" title="When hiding the base layer, you can easily see how the obstacle layer is setup">}}

Using [MSKTiled](https://github.com/sanderfrenken/MSKTiled/blob/main/README.md), an open source Swift package I developed, one can load Tiled maps into SpriteKit's `SKTileMapNode` instances and access the related types. As such, one can very efficiently render large maps, while using the extensive toolset of Tiled.

### GKGridGraph

There are several means to accomplish pathfinding on maps. For grid based games, A* is quite a commonly used algorithm. Variants of it also exists like [Jump point search](https://en.wikipedia.org/wiki/Jump_point_search) which is more efficient algorithm but can yield different paths compared to A*. Alternatively one can also rely on mesh graphs for pathfinding. This is a technique where one defines polygon shapes that cover the non-traversable area's of a graph. The algorithm will then aim to avoid these area's, and will travel freely over the remaining available space. From that perspective its a very different approach compare to A*, as with A* the movement is restricted to the grid.

Apple's [GameplayKit](https://developer.apple.com/documentation/gameplaykit) is an extensive toolbox for game development, offering things like state machines, utilities for the entity-component-system, noise-generators and much more. It also includes pathfinding capabilities, which are formed around [GKGraph](https://developer.apple.com/documentation/gameplaykit/gkgraph) and related types. GameplayKit offers the following options:

- Use the GKGridGraph and GKGridGraphNode classes to describe game worlds that constrain movement to an integer grid, such as tactical role-playing games.

- Use the GKObstacleGraph or GKMeshGraph class to describe 2D game worlds that allow continuous movement in open spaces that are interrupted by impassable obstacles (GKPolygonObstacle objects). Obstacle graphs automatically generate nodes containing 2D point information (GKGraphNode2D objects), and you can also add your own such nodes representing locations of interest.

With the setup of maps I use for Battledom in mind, I did implement support for pathfinding into MSKTiled using [GKGridGraph](https://developer.apple.com/documentation/gameplaykit/gkgridgraph). 

I have setup a repo [here]() that contains the Tiled map as shown above that uses MSKTiled to render the map and demonstrate the pathfinding. If you run the project, you should see this:

{{< figure src="../../images/screenshot-pathfinding-demo.png" width="800" title="Rendered map. The red crosses are layed over the obstacles to clarify non-traversable tiles">}}

When you tap the screen on a non-obstacle tile, you will see a red circle indicator showing where you tapped. This will be startpoint of the path. Tap again on another non-obstacle tile, and the path from start to end will be calculated and visualized. The path calculated using `GKGridGraph` is visualized with red circles.

{{< figure src="../../images/screenshot-pathfinding-demo-path.png" width="800" title="Rendered path. The red circles visualize the path as calculated using GKGridGraph">}}

The code to achieve this all is quite simple. 
Inside the GameScene, I invoke:

``` Swift
updatePathGraphUsing(layer: obstacleLayer, diagonalsAllowed: true)
```

This will instruct MSKTiled to construct the path graph using the obstacle layer (which is the layer named "obstacles"). That implementation is as follows:

``` Swift
public func updatePathGraphUsing(layer: SKTileMapNode, diagonalsAllowed: Bool) {
    pathGraph = GKGridGraph(fromGridStartingAt: vector_int2(0, 0),
                            width: Int32(layer.numberOfColumns),
                            height: Int32(layer.numberOfRows),
                            diagonalsAllowed: diagonalsAllowed)
    guard let pathGraph else {
        return
    }
    for column in 0..<layer.numberOfColumns {
        for row in 0..<layer.numberOfRows {
            if layer.tileGroup(atColumn: column, row: row) != nil {
                let node = pathGraph.node(atGridPosition: vector_int2(Int32(column), Int32(row)))!
                node.removeConnections(to: node.connectedNodes, bidirectional: false)
            }
        }
    }
}
```

It creates a `GKGridGraph` using the same size as the tile map. Then, each tile that has a tileGroup (which functionally drills down to has a tile; IE an obstacle) will be removed from the graph.

Now, this pathGraph can be used to find paths from any valid tile in the grid to another valid tile in the grid as follows:

``` Swift
public func getPath(fromTile: MSKTiledTile, toTile: MSKTiledTile) -> [MSKTiledTile]? {
    if !isValidTile(tile: fromTile) {
        log(logLevel: .warning, message: "Invalid tile provided as start for path")
    } else if !isValidTile(tile: toTile) {
        log(logLevel: .warning, message: "Invalid tile provided as end for path")
    }
    guard let pathGraph = pathGraph else {
        log(logLevel: .warning, message: "Pathgraph has not been initialized yet")
        return nil
    }
    guard let startNode = pathGraph.node(atGridPosition: vector_int2(Int32(fromTile.column), Int32(fromTile.row))) else {
        log(logLevel: .warning, message: "Invalid start position for finding a path")
        return nil
    }
    guard let endNode = pathGraph.node(atGridPosition: vector_int2(Int32(toTile.column), Int32(toTile.row))) else {
        log(logLevel: .warning, message: "Invalid end position for finding a path")
        return nil
    }
    let foundPath = pathGraph.findPath(from: startNode, to: endNode)
    if foundPath.isEmpty {
        log(logLevel: .warning, message: "Path could not be determined")
        return nil
    }
    ...
}

public func isValidTile(tile: MSKTiledTile) -> Bool {
    if tile.row < 0 || tile.column < 0 {
        return false
    }
    return tile.row <= baseTileMapNode.numberOfRows-1 || tile.column <= baseTileMapNode.numberOfColumns-1
}
```

This is quite straight forward. In your console, you will also see some print statements. When the path is calculated, you will see for instance:

`time to find GKGridGraph path: 97.513916`

This means that calculating the path has taken almost 100 milliseconds, which is a lot.
In Battledom I might most of the times use somewhat smaller maps, with less obstacles. 
But still I often encountered path calculations that take more than 50 milliseconds. 

This is very problematic, as it is much more than the approx. 16 milliseconds you get when you want to support a steady 60 FPS. 

I don't know why this `GKGridGraph` is so inefficient. I can only hypothesize that since `GKGridGraph` is a subclass of `GKGraph` which also to some extent needs to support the other types of graphs like `GKObstacleGraph` or `GKMeshGraph`, there is a lot of abstraction going on and therefore the actual A* implementation is much different from the usual vanilla implementation. On smaller grids `GKGridGraph` might still be usable, but with larger grids I needed to look for alternatives.

### Vanilla A* using DeepSeek


## Wrap up

There is a lot more that can be told about the internals and the pro's and con's of the solution, and I would suggest you to use the [readme](https://github.com/sanderfrenken/MSKTiled/blob/main/README.md) for more technical information and usage.

I am using MSKTiled now in my upcoming game [Battledom](https://sanderfrenken.github.io/dev-blog/projects/battledom/), and so far I am very happy with and actively maintaining it while doing so.

Do you have any questions, suggestions or remarks for this post? 

Please leave a message below!

{{< youtube ZecfzsuEKZw >}}