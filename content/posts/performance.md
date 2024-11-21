+++
title = 'About performance and caching'
date = 2024-11-20T00:04:03+02:00
draft = false
type = "post"
showTableOfContents = true
+++

With Battledom nearing a state where I expect it's foundations to be done, I lately spend some extra time on profiling and performance testing the game.

As Battledom can become computational quite heavy at moments when large maps are rendered while moving around hundreds of units, the topic of optimization has been crossing my path (maybe too) often. 

But in general, with any app or game you develop, you will most likely at one moment in time face some form of performance issues.

{{< figure src="../../images/screenshot-performance-units.png" width="800" title="Example of a more intense battle in Battledom">}}

Some of the performance issues I encountered were about loading content and framedrops when loading textures. I will give some examples of these issues and enhancements I did in this post.

## Performance

The first thing that I find very important to note is that performance testing of your game should be done on a real device. Testing on a simulator is ofcourse convenient and very suitable for feature development and tracking down memory leaks, but performance is very dependent on the hardware in use. 

Since the Xcode simulator is in essence an app running on your OSX machine emulating an iPhone/ iPad, its nowhere near a proper representation of a real physical device. A thorough explanation about the differences can be read [here on the Apple Developer documentation](https://developer.apple.com/documentation/xcode/testing-in-simulator-versus-testing-on-hardware-devices), but an overview from there:


> - General: there are differences in processing, graphics, and network speed..

> - Display: there are differences in resolution and color.

> - System: there are differences in how the system handles backgrounding your app.

> - Hardware: Simulator doesn’t support some hardware features.

> - API: Simulator doesn’t support some APIs.

In specific interest when using SpriteKit, is this final mention:

> - Metal: there are differences between the device GPU and your computer GPU.

As [SpriteKit](https://developer.apple.com/documentation/spritekit) is leveraging Metal for it's rendering capabilities, this is an important note. 
You can find more in-depth information about running Metal apps in the simulator [here](https://developer.apple.com/documentation/metal/developing_metal_apps_that_run_in_simulator).

Now in practice you might encounter that your game runs smoother on your physical device compared to a simulator, but I also have encountered it the other way around (less often though). It's very dependent on the tasks being performed and the hardware in use. 

But as a general remark: when you are trying to assess your game's performance, make sure to use a real physical device, and if possible a variety. Moreover, make sure to include lower-end devices like devices released a couple of years ago. 

There is a huge [difference](https://www.apple.com/iphone/compare/?modelList=iphone-11-pro,iphone-16-pro) between the computing capabilities of for instance an iPhone 16 Pro (released 2024; 6-core GPU; A18 Pro-processor) and an iPhone 11 Pro (released 2019; 4-core GPU; A13 Pro-processor). 

{{< figure src="../../images/screenshot-performance-11-16.png" width="800" title="Comparing iPhone 11 Pro with iPhone 16 Pro">}}

Taken this into account, I usually do most extensive performance tests on my older iPhone device (iPhone 11).

Now let's look at some examples of performance issues I encountered.

## Maps

In Battledom I make use of maps that are layed out using tiles. Most maps are about 100\*100 tiles, and maybe an occassional larger map of 200\*200  tiles. I won't make them much larger, as they become too large to navigate in the timespan I expect one to finish the level. 

When developing the map renderer (of which most code is available at GitHub: [MSKTiled](https://sanderfrenken.github.io/dev-blog/posts/msktiled/)), I therefore needed to make sure that it would be capable of efficiently drawing maps in this larger size of 200\*200 tiles. 

During development however, I try to explore the limits of the game's internals, and stretch them as far as possible. This means that even though I might not exceed maps of 200\*200 tiles, I do try to render maps at least twice that size. 

With regard to MSKTiled, I am able to render at maps of 1000\*1000 tiles smoothly. This made me confident that the library would be performing sufficient to be used in Battledom.

{{< figure src="../../images/screenshot-large-map.png" width="800" title="Large map example including calculated path from A to B">}}

At the same time it is important to notice that even though this individual component of the game seems to function and perform properly, it might behave differently when combined with all other parts of the game once integrated. 

But overall, when making a game with some form of complexity and heavy computational work, it does make life easier to modularize your systems as this allows you to individually assess their performance (aside from the argument that modularisation can clean up your code significantly when applied properly). 

With [Swift Package Manager](https://developer.apple.com/documentation/xcode/swift-packages) this has become very convenient.

### Caching

When looking at the capability to render large maps rendering, caching is an important subject. 

When using MSKTiled, parsing tilemaps the first time is expensive as the package will need to create the individual tiles from the tilesheets referenced in the map. 

This means that for example the tiles in the sheet below will need be cropped to it's individual tiles on the device at runtime when creating the map. 

{{< figure src="../../images/screenshot-tileset.png" width="800" title="Example spritesheet for a terrain">}}

SpriteKit provides an API to do this, which is [SKTexture(rectIn:)](https://developer.apple.com/documentation/spritekit/sktexture/1520425-init). This API allows you to create a texture from a portion of another texture. 

The result, an [SKTexture](https://developer.apple.com/documentation/spritekit/sktexture) it self, can then be used to draw new nodes from. One could use this texture directly to create the tiles. 

But in this example, we instead will write it as an image to the disk, to reload the texture from that image subsequently. This might seem a bit strange, and I agree it is. The motivation for this workaround can be read [here](https://github.com/mfessenden/SKTiled/issues/40#issuecomment-1172284072). Note, that I recently found a way to prevent this workaround, but I leave that for another time and place.

All in all, this is an expensive operation, especially if we are talking about a map that might contain 1000's of different tiles. In such an example, constructing (basically cropping 1000's of images) the needed tiles can easily take up to 3 seconds.

To resolve this, I implemented a first layer of caching: When an individual tile is created for the first time in the app's lifecycle on the device, I create the [UIImage](https://developer.apple.com/documentation/uikit/uiimage/1624114-init) from the cropped texture's [CoreGraphic image](https://developer.apple.com/documentation/spritekit/sktexture/1519755-cgimage). 

Subsequently, this UIImage is written to the app container document's directory for future usage.
Everytime a tile is attempted to be loaded by MSKTiled, it will first query the document's directory if the image is already there. 

In code, this looks likes this:

```
var texture: SKTexture?
// retrieve the tile definition
let tileDefinition = getTile(tileSheet: tileSheet, tileId: tileId)

// first check the file system (cache) for existence
if let imageData = try? Data(contentsOf: getCacheFileUrl(tileName: tileDefinition.name)) {
    if let image = UIImage(data: imageData) {
        texture = SKTexture.init(image: image)
        log(logLevel: .debug, message: "Cache hit for image \(tileDefinition.name)")
    }
}
// if not present in cache, create the tile and save it to the cache
if texture == nil {
    // load the parent tilesheet
    let sourceTexture = getTexture(name: tileDefinition.source)

    // create the cropping instructions
    let row = tileDefinition.row
    let column = tileDefinition.column

    // crop the parent texture to the tile texture
    let tileTexture = SKTexture(rect: CGRect(x: tileDefinition.width*CGFloat(column)/sourceTexture.size().width, 
                                            y: 1-(tileDefinition.height*CGFloat(row+1)/sourceTexture.size().height),
                                            width: tileSize.width/sourceTexture.size().width,
                                            height: tileSize.height/sourceTexture.size().height),
                                in: sourceTexture)

    // save the tile texture image to the file system
    let uimg = UIImage(cgImage: tileTexture.cgImage())
    if let data = uimg.pngData() {
        try? data.write(to: getCacheFileUrl(tileName: tileName))
    }
    texture = SKTexture(image: uimg)
}
```

Now we can reference to this form of caching as `static caching`. These images are persisted on the device, and will remain a valid cache object also after the app or system has been rebooted.

On the other hand, we can also make use of `runtime caching`. 

Taking the example of a large map with multiple layers of 200\*200 tiles, even with static caching as described above loading a map can still take up to a few seconds. Usually you would like to avoid such long loading times. Or at least, you want the user not to notice their existence.

In order to improve this, one can implement runtime caching. With MSKTiled, I created a method to parse a map and store the individual components that would be used to construct a scene from it. 

Parsing the map contains the most expensive operations, where in essence its creating the individual layers and objects represented as SpriteKit objects. Inside this logic resides the parsing of the map and it's layers and creating the cropped tiles (either from cache or fresh).

The parsed map can be used later at any time by injecting it into a scene, which will use the individual components for rendering the scene itself.

Below a simplified version of the code in use. This class, [MSKTiledMapParser](https://github.com/sanderfrenken/MSKTiled/blob/main/Sources/MSKTiled/MSKTiledMapParser.swift) implements the `XMLParserDelegate`, where the actual parsing is done by the implemented protocol methods (which I left out in the below code example).

```
public func loadTilemap(filename: String,
                        allowTileImagesCache: Bool = true,
                        checkBundleForTileImages: Bool = false,
                        addingCustomTileGroups: [SKTileGroup]? = nil) {
parser.delegate = self
// inside parse is where the expensive work happens: looping over all the layers and tiles
// creating the tiles, caching them if needed, constructing the layers as much as possible
parser.parse()
// now that the parser has parsed the map, it already has loaded most of the map information needed for rendering into memory
}
```

Moreover, we can dispatch the `loadTilemap` invocation to a non-main thread, thereby doing the work while the user will not even notice it. 

Once finished, we can use this MSKTiledMapParser instance at any subsequent moment we need it to render a scene. In Battledom, at app startup I start doing this work right away:

```
func application(_ application: UIApplication, didFinishLaunchingWithOptions
                     launchOptions: [UIApplication.LaunchOptionsKey: Any]?) -> Bool {
        printInDebug(NSHomeDirectory())
        loadSavedData()
        cacheScenes()
        return true
    }

private func cacheScenes() {
    if allowSceneCaching {
        cachedSceneHelper.cacheScene(sceneType: .overworld)
        cachedSceneHelper.cacheScene(sceneType: .start)
        cachedSceneHelper.cacheScene(sceneType: .village)
        cachedSceneHelper.cacheScene(sceneType: .woodlands)
        cachedSceneHelper.cacheScene(sceneType: .farm)
    }
}
```

The AppDelegate will start caching all necessary immediately. 

During the app's launch sequence, the user is presented with an animated scene that presents the game's name and some other details, something which you see in most other games. 

Obviously, these scenes or screens, also known as [splashscreens](https://en.wikipedia.org/wiki/Splash_screen), exist for a reason: to cache/ load content upfront while still creating some form of engagement with the user. 

Now that we have the parser which already processed and stored most of its data, we need to use it in a scene that we present. This can be done by invoking the method `getTileMapNodes` [from the parser](https://github.com/sanderfrenken/MSKTiled/blob/70475e6e2303ea67057c44c1d339b9b9fee22e6d/Sources/MSKTiled/MSKTiledMapParser.swift#L49) below that returns the SpriteKit objects from it:

```
@MainActor
public func getTileMapNodes() -> (layers: [SKTileMapNode],
                                  tileGroups: [SKTileGroup],
                                  tiledObjectGroups: [MSKTiledObjectGroup]?) {
    let tileMapNodeLayers: [SKTileMapNode] = layers.map { layer in
        let tileSet = layer.tileSet
        let rawLayer = layer.rawLayer
        let layerData = layer.layerData
        let tileMapNode = SKTileMapNode(tileSet: layer.tileSet, 
                                        columns: Int(mapSize.width), 
                                        rows: Int(mapSize.height), 
                                        tileSize: tileSize)
        for tileId in layerData {
            if !hasValidTileData(tileId: tileId) {
                continue
            }
            let tileGroup = getTileGroup(tileId: tileId)
            tileMapNode.setTileGroup(tileGroup, forColumn: column, row: row)
        }
        tileMapNode.name = rawLayer.name
        if rawLayer.invisible {
            tileMapNode.alpha = 0
        }
        return tileMapNode
    }
    return (tileMapNodeLayers, tileGroups, tiledObjectGroups)
}
```

Note here, that this method is annoted with `@MainActor`. 

This is an important aspect as well. Initialisation and usage of any SKNode subclass should be done on the main thread in SpriteKit. This means that as the above method is initializing instances of SKTileMapNode, this needs to happen on the MainActor (potentially blocking the UI).

This is something to really look out for: even though you might cache something up to front, always verify you are not blocking the UI while creating the cache. And when this can not be avoided, try to do it at moments when your user will not be bothered with it, for example during a loading screen.

## Texture caching

Another example of caching is texture caching. Even thoug Apple did implement some form of texture caching already, you might need more aggressive texture caching. According to the [documentation](https://developer.apple.com/documentation/spritekit/sktexture/), out of the box texture caching works as follows:

> After a texture is loaded into the graphics hardware memory, it stays in memory until the referencing SKTexture object is deleted. This means that between levels (or in a dynamic game), you may need to make sure a texture object is deleted. Delete a SKTexture object by removing any strong references to it, including: 

> - All texture references from SKSpriteNode and SKEffectNode objects in your game

> - Any strong references to the texture in your own code

> - An SKTextureAtlas object that was used to create the texture object

This does mean that while an SKTexture is still referenced from somewhere, any subsequent usage of that same texture can be facilitated by loading it from memory. However, once you removed all references to that texture, its deallocated. This can be a problem if there are textures of which you know that will be requested often but are shortlived in usage. 

What I see is that when loading larger textures into memory, you can experience framedrops.

For those occassions I make of a texture helper. This TextureHelper is a singleton which holds a dictionary of textures. Simplified this looks like this:

```
final class TextureHelper {
    private var cachedTextures = [String: SKTexture]()

    func getCacheableTexture(imageName: String, setInCache: Bool = true) -> SKTexture {
        if let texture = cachedTextures[imageName] {
            return texture
        }
        let texture = SKTexture(imageNamed: imageName)
        if setInCache {
            cachedTextures[imageName] = texture
        }
        return texture
    }
```

Now you can use your TextureHelper instance from anywhere in your code in order to retrieve a texture from it's cache (or load it newly and cache it subsequently):

```
let myTexture = textureHelper.getCacheableTexture(texture: "my_texture")
```

It's an easy way to retain a strong reference to a loaded texture in memory, inhibiting the texture from being deallocated. Use it wisely though, as you don't want to increase your memory usage too far.

## Cache invalidation


### Static cache 

As you might know, any cache can get corrupted. This could happen for instance in our static caching as described above. 

We create individual images from tilesheets and store them to the device. But what will happen if in a subsequent app update, I decide to rename a tilesheet? Currently, I use the tilesheet name among other information to define the name of the cached file. 
Renaming a tilesheet will definitely break some of the desired behavior in this case. 

Therefore, I would always advise you to introduce means that can invalidate your built up cache. In the example, clearing the cache would be as simple as removing all images stored in the cache directory in use. 

That way, it will need to be built up again from scratch. It's up to you decide when to do these actions, but you should try to avoid it as much as possible (otherwise, why would you cache). 

Additionally, you could create a mean for a user to do this themselves via a settings menu for example. That way, you can troubleshoot without the need for an app update when things go wrong.

### Runtime cache

When doing runtime caching, you might store too much information in your RAM and the OS might start to complain that your app uses too much memory. 

You are warned about this in your AppDelegate's method `applicationDidReceiveMemoryWarning` which you can hook into and respond by taking the right actions. In Battledom, I use this to clear up all textures that I have been caching since the app started:

```
func applicationDidReceiveMemoryWarning(_ application: UIApplication) {
    printInDebug("applicationDidReceiveMemoryWarning")
    textureHelper.clearCharacterTextures()
    textureHelper.clearStaticTextures()
}
```

## Wrap up

I hope you enjoyed and maybe learned something from this post. There is a lot more to tell about performance, like the tooling that you can use to assess it. 

This post is mostly about loading performance. Runtime performance is just as important, but is much more dependent on the tasks that are being performed. 

I will do more about this in future posts, looking at specific tasks that impact performance (like pathfinding) and how you could fix them.

But for now I would like to emphasize the following learnings:

- Performance testing should be done on physical devices

- Try to performance test on a variety of devices, and include low-end devices as much as possible

- When optimizing for performance, try to stay performant well beyond the needs of your game

- Modularization can help you isolating certain pieces of code and performance test them individually

- There are ways to cache certain actions (static caching) like cropping textures and storing them to the filesystem, which will benefit performance even after the game has been rebooted

- Runtime caching can be very valuable to prevent reloading of textures or other data into memory that are requested often

- If you need to create cache up front, try to dispatch the work away from the main thread as much as possible to avoid blocking the UI.

- When using runtime caching make sure to be able to respond to memory warning by clearing this cache

- When using static caching make sure to not create a situation where you are using a corrupted cache, and provide means to escape from this corrupted cache in case it does happen.

Next time, I will write about memory leaks and Xcode Instruments.

Do you have any questions, suggestions or remarks for this post? 

Please leave a message below!

{{< youtube 0y0pernS0Gc >}}