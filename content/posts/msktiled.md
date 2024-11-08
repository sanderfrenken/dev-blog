+++
title = 'MSKTiled: A Tiled map parser for SpriteKit'
date = 2024-11-07T00:04:03+02:00
draft = false
type = "post"
showTableOfContents = true
+++

## Tilemaps

In 2016 at WWDC, Apple announced a new addition to SpriteKit: Support for tiling.

Tiling is a technique often used in (2D) game development, where maps are created using tiles.

Using spritesheets, one can have a variety of for example 100 tiles in a single image. These tiles can include terrain-like tiles, building-like tiles including walls, etc.

{{< figure src="../../images/screenshot-tileset.png" width="400" title="Example spritesheet for a terrain">}}

Using these individual tiles one can create unique environments, all drawn using the same spritesheet as a source.

{{< figure src="../../images/screenshot-map.png" width="400" title="Example map; all the terrain is drawn using the above spritesheet">}}

In essence, one can see a tilemap as two-dimensional array where each item represents a specific tile.
As a basic example, one can think of a following way to represent this in code:

```
let mapArray = [[Tile]]() // This is the map definition
var xPosition = 0
var yPosition = 0
// Loop ever each tile in the map definition
for row in mapArray {
    for tile in row { 
        // Create the tile image based on the information in the array
        let tileImage = Image(from: tile)
        // Set the position of the tile in the to render scene
        tileImage.position = Point(x: xPosition*tileSize, y: yPosition*tileSize)
        // Render the tile
        renderImage(tileImage) 
        xPosition += 1
    }
    yPosition += 1 
}
```

This "solution" was already possible using SpriteKit using `SKNode` and `SKSpriteNode` instances.

However, with iOS 10 more extensive support for tiling was added to SpriteKit.

## SKTileMapNode

One of the drawbacks of the solution as posed above is that when one creates a map of for instace 100 tiles in width and 100 tiles in height, a total of 10.000 tiles would need to be individually rendered to a scene. 

Now, imagine that you also support layers of maps. 

In most cases maps are layered: There is a layer of terrain, where on top there can be a layer of flora, another layer with the walls of buildings, another layer of decorations like windows and doors to be rendered on top of the walls, etc.

This could easily make your game render > 20.000 nodes for each frame, only to draw the map itself. This can become quite expensive and hence impact performance of your game.

To cater for this issue, Apple introduced [SKTileMapNode](https://developer.apple.com/documentation/spritekit/sktilemapnode). As always, I would encourage you to read the documentation before using it, but a short quote of the documentation:


> SKTileMapNode does the work of laying out predefined tiles in a grid of any size. Typically, you configure 9-slice images (tile groups) in Xcode's SpriteKit scene editor and paint the look of your tile map ahead of time versus configuring the tile map in code.


There is a lot to read in these 2 short sentences. First, its appearant that this `SKTileMapNode` is capably of drawing a layer of a map inside a single SKNode. In practice, this means that instead of your game requiring SpriteKit to draw 10.000 individual nodes for a layer, it will now render just one node.

Moreover, the quote also talks about Xcode's SpriteKit scene editor. So what is that?

## Tilemap editors

In order to create a tilemap, one would usually prefer to use a specific editor instead of drawing a map just using code. Use a tilemap editor, one can preview the map inside the editor and change it. You can see it as a Paint or Photoshop, geared specifically to create tilemaps.

### Tiled

One very popular tile editor is [Tiled](https://www.mapeditor.org). Tiled is used by many game developers to create tilemaps, and used in popular games like [Shovel Knight](https://www.yachtclubgames.com/games/shovel-knight-treasure-trove/). 

With Tiled, one can create tilemaps in various perspectives (isometric/ hexagonal/ orthogonal), animate tiles (for instance think of looping water tiles to create an illusion of movement), add objects and assign properties to tiles, layers and objects (and much more). 

You can import and use spritesheets on the fly, as you can see in the below example in the right lower corner.

{{< figure src="../../images/screenshot-tiled.png" width="800" title="A map I created in Tiled for Battledom">}}

Obviously, tilemaps as created by Tiled use their own formats (there is no standard format for tilemaps) and in essence Tiled and Apple have nothing to do with each other. Therefore, a map created in Tiled or any other third-party tilemap editor will not work out-of-the-box in a SpriteKit game. 

### Xcode tilemap editor

As a mean to create tilemaps for your SpriteKit games using an editor, Apple added support for creating tilemaps from Xcode. Using the scene editor, one can drag an `SKTileMapNode` into the scene and edit it the way you want it to look:

{{< figure src="../../images/screenshot-xcode-tiled.png" width="800" title="Example map in Xcode scene editor">}}

With the new SpriteKit tiling API's, a map layer is represented as a `SKTileMapNode`, and you populate each grid entry with an image.

An image is represented in the form of an [SKTileDefinition](https://developer.apple.com/documentation/spritekit/sktiledefinition). Basically, this is just the image to represent a specific tile, extended with the ability to assign custom properties to this tile in the form of [userdata](https://developer.apple.com/documentation/spritekit/sktiledefinition/1645813-userdata).

So in order to work with different tiles in the Xcode scene editor, you need to create different `SKTileDefinition` instances. These `SKTileDefinition` can be grouped into a set by using [SKTileGroup](https://developer.apple.com/documentation/spritekit/sktilegroup). These groups can be enriched with adjacency rules (see [SKTileGroupRule](https://developer.apple.com/documentation/spritekit/sktilegrouprule)), which will instruct the editor how to draw edges of larger areas when using the same tile.

For instance, below you will see a `SKTileGroup` as created in Xcode.

{{< figure src="../../images/screenshot-tilegroup.png" width="800" title="Example tilegroup in Xcode">}}

You see that this group contains 13 different images (tiles), with each image belonging to a certain rule (Up edge, center, Down edge etc.). You can enrich each individual tile with variations, and assign weights to them. This will make the `SKTileMapNode` pick a random variation of that tile based on the weights provided.

{{< figure src="../../images/screenshot-tilegroup-variant.png" width="800" title="Example tilegroup with a variant in Xcode">}}

In a nutshell, with iOS 10.0 SpriteKit was now enhanced with not only new API's to draw tilemaps in your game more efficiently than ever before, but also with a brandnew tilemap editor. 

It's good to note that all above API's are available in code as well, so you are not bound to use the Xcode tilemap editor per se, which might be a very good thing..

### Xcode tilemap editor vs Tiled

With the release of Xcode 8 and the support for tilemaps, it was time to dig in. I was using Tiled before for creating my tilemaps, but I had no mean to translate these tilemaps as defined in Tiled to work in a SpriteKit game using the new tiling API's. 

Ideally, I would now learn to use the new built-in tilemap editor and stick with Xcode for everything game development related!

One of the first things I noticed however was that the editor has less functionalities than Tiled. 
I could not create object layers to contain non-tile related information. There were no means to draw any custom shapes like poygons on the map, something that can be very useful for instance to create an efficient pathing navmesh.

But the biggest issue I faced was with creating the `SKTileGroup` definitions. 

In Xcode's tilemap editor, you can't use a spritesheet. Instead, you need to manually drag and drop each individual image to its destined tiledefinition. This also means that if you have a spritesheet, you need to manually crop the individual image and then drag it into Xcode. I have maps in my games that render more than 1.000 different tiles. That is unmanageable when not using spritesheets.

Unfortunately, we can conclude now that over the years since it's first release, nothing has significantly changed here. As of now, we are still not able to use spritesheets in Xcode's map editor. 

As I was building SpriteKit games, and definitely wanted to make use of the new tiling API's, I saw only one viable alternative: Building the logic to read Tiled maps in a `SpriteKit` game and render the maps using `SKTileMapNode` and related new classes. That way, I could make use of the highly performance SpriteKit tiling capabilities, while still being able to enjoy Tiled with all it's goodness.

## MSKTiled

When I started to develop my upcoming game Battledom, the most essential starting part was to develop a tool to make this bridge between Tiled and the SpriteKit tiling API's. 

There was already a framework that is capable of integrating Tiled maps into SpriteKit: [SKTiled](https://github.com/mfessenden/SKTiled).

This is a very nice framework with loads of features supporting different usecases as used in Tiled, but it doesn't use the SpriteKit tiling API's. Instead, it draws each tile as a separate SKSpriteNode, leading to [performance issues on large maps](https://github.com/mfessenden/SKTiled/issues/40).

However, it was a good starting point to investigate how I could develop my own solution.

The first thing to understand was the underlying datastructure that Tiled uses to persist a tilemap in.
By looking at the xml structure and altering the map in the Tiled editor to see the effects, at a certain moment I was able to think of a proper mapping between Tiled structure and the desired SpriteKit object's structure.

This is a Tiled map example (.tmx file):

```
<map version="1.10" tiledversion="1.11.0" orientation="orthogonal" renderorder="right-down" width="100" height="100" tilewidth="32" tileheight="32" infinite="0" nextlayerid="15" nextobjectid="489">
 <tileset firstgid="1" name="terrain" tilewidth="32" tileheight="32" tilecount="2048" columns="32">
  <image source="terrain.png" width="1024" height="2048"/>
 </tileset>
 <tileset firstgid="2049" name="plants" tilewidth="32" tileheight="32" tilecount="512" columns="16">
  <image source="plants.png" width="512" height="1024"/>
 </tileset>
 <layer id="1" name="ground_0" width="100" height="100">
  <data encoding="csv">
385,385,386,386,387,386,387,385,3852,322,..........
</data>
 </layer>
 <objectgroup id="3" name="bases">
  <object id="1" name="basePoint" x="1112" y="1152">
   <properties>
    <property name="range" type="int" value="6"/>
    <property name="type" type="int" value="0"/>
   </properties>
   <point/>
  </object>
 </objectgroup>
 <objectgroup id="14" name="polygonObstacles">
  <object id="133" x="1512" y="1096">
   <polygon points="0,0 240,0 240,248 -24,240"/>
  </object>
  <object id="134" x="1888" y="1216">
   <polygon points="0,0 240,0 240,248 -24,240"/>
  </object>
  <object id="135" x="1712" y="1544">
   <polygon points="0,0 240,0 240,248 -24,240"/>
  </object>
  <object id="136" x="1288" y="1792">
   <polygon points="0,0 240,0 240,248 -24,240"/>
  </object>
  <object id="137" x="2080" y="1824">
   <polygon points="0,0 240,0 240,248 -24,240"/>
  </object>
  <object id="138" x="1133.82" y="1232">
   <polygon points="0,0 298.182,0 298.182,296 -29.8182,286.452"/>
  </object>
 </objectgroup>
</map>
```

As you can see, there are `tileset` instances defined, which represent a spritesheet. The tiles they contain can be referenced as numbers. This you can see in the `layer` instances: Each layer is a layer in the map, has a width and height (expressed in tiles) and contains a `data` element. This data element is an array of integers, where each integer references to an entry in a tileset (a tile image).

Object layers are also clearly structured as `objectgroup` instances, which can contain `object` instances with (custom) properties and positional information. 

Using this information, I drafted some mappings:

- A layer in Tiled corresponds with an SKTileMapNode in SpriteKit.
- A single data element in Tiled corresponds to an `SKTileDefinition` in SpriteKit.
- Properties in Tiled can correspond to the [userdata](https://developer.apple.com/documentation/spritekit/sktiledefinition/1645813-userdata) of an SKTileDefinition in SpriteKit when the properties are put on an individual tile. 
- For objects and object layers, I needed to create my own custom classes and structs, as there is no such thing in SpriteKit.

Eventually, this mapping became more and more clear, and as a result MSKTiled became a working SPM package that offers the capability to turn Tiled maps into SKTileMapNode and related objects. 

As I use it for my own games at the moment, I only focussed on supporting orthogonal maps. It does not support isometric or hexagonal maps at the moment.

In addition to being just a bridge between Tiled and SpriteKit tiling API's, MSKTiled will also take care of rendering the Tiled maps to an SKScene subclass which you can inherit from in your own game.

This is what I called the `MSKTiledMapScene`, which you initialize by injecting the name of the tilemap to render and various other variables. 

Internally, MSKTiled parses the map, looks up the spritesheets, crops the desired tiles from the sheet and save it to the disk for caching purposes. These individual images are then used to programmatically create the SKTileDefinition instances which are used to create the SKTileMapNode. 

These are then layed out on the scene at the provided zpositions, and can be accessed programmatically using the normal SpriteKit API's.

The scene also provides means for A* pathfinding and camera behavior including zooming and scrolling constrained by the provided and calculated boundaries.

There is a lot more that can be told about the internals and the pro's and con's of the solution, and I would suggest you to use the [readme](https://github.com/sanderfrenken/MSKTiled/blob/main/README.md) for more technical information and usage.

I am using MSKTiled now in my upcoming game [Battledom](http://localhost:1313/projects/battledom/), and so far I am very happy with and actively maintaining it while doing so.

Do you have any questions, suggestions or remarks for this post? 

Please leave a message below!

{{< youtube 0y0pernS0Gc >}}