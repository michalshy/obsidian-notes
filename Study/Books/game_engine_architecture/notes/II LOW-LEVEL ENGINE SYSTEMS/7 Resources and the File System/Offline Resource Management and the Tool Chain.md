### Revision Control for Assets
Small teams can keep files sitting loose around on shared network drive with an ad hoc directory structure.
For bigger projects, usually some code revision control system is used like Perforce.
Some teams provide wrappers around VSC to flatten the learning curve for their artists.
#### Dealing with data size
Usually the biggest problem in resource control system is the size of the assets. They are relatively big, comparing to source files.
There are various approaches.
Some teams turn to commercial revision control system which handles big data well.
Other teams use standard VSC, but it is slowing down operations (pulling, merging, pushing) of that VSC.
Some teams build systems on top of VSC's or build their own tools.
Naughty Dog use a  tool that makes use of UNIX symbolic links. As long as file is not edited, on drive, engine has link to the asset on shared network drive.
When file is checked out for editing, local copy is taking place of asset on drive, rather than symbolic link, and when editing is done, the local copy becomes new master copy. Later, local copy turns into symlink.
### The resource database
Most assets are not used by the engine in their original format. They need to pass through some kind of asset conditioning pipeline, whose job it is to convert the assets into the binary format needed by the engine.
Each resource has some *metadata* which describes, how this asset should be processed.  When compressing bitmap, we need to choose compression type, when exporting animation, we need to know what range of frames in Maya should be exported.
To manage all of this metadata, we need some kind of database. If we are making a very small game, this database, might be housed in the brains of the developers.
Is there other option?
Every professional game team has some kind of semiautomated resource pipeline, and the data that drive the pipeline is stored in some kind of resource database.
The resource database takes vastly different forms in game engines.
Some engines provide metadata embedded into source assets themselves (Maya).
Other, provides small files accompanied with resource file, which describes how it should be processed (Unity). 
Still other engines encode their resource building metadata in a set of XML files, perhaps wrapped in some kind of custom graphical user interface. Some engines employ a true relational database, such as Microsoft Access, MySQL or conceivably even a heavyweight database like Oracle.
No matter form, resource database should provide following functionality:
- ability to work with multiple type of resources
- ability to create new resources
- ability to delete resources
- ability to inspect and modify existing resource
- ability to move resource from one location to another on-disk
- ability to cross-reference other resources (e.g. the material used by a mesh, or the collection of animations needed by level 17), **those cross references usually drive both the resource building process and the loading process at runtime**
- ability to maintain *referential integrity* of all cross-references within the database and to do so in the face of all common operations such as deleteing or moving
- ability to maintain revision history
- ability to search or query in various ways
### Some good resource database designs
Here are some designs that have worked well in the experience of the book author.
##### Unreal Engine 4
In Unreal Engine, resource database is managed by UnrealEd. It is tool responsible for everything from resource metadata management to asset creation to level layout and more.
UnrealEd is part of engine.
This permits assets to be viewed as how they will appear in game. The game can be run from UnrealEd to visualize the assets in natural surroundings.
Having a single, unified and reasonably consistent interface for creating and managing all types of resources is a big win. This is especially true considering that the resource data in most other game engines is fragmented across countless inconsistent and often cryptic tools. Just being able to find any resource easily in UnrealEd is a big plus.
Unreal can be less error-prone. It checks validity of assets early in development,.
Drawbacks are resource data stored in a small number of large package files. These files are binary, it is not easy to version control them.
Referential integrity is good in UnrealEd, there are small problems. When object is moved or renamed, all cross-references are passed to dummy object which remap old resource to new one. Those dummy objects hang, accumulate and can later cause object.
By far the most user-friendly.
##### Naughty Dog's Engine
They stored resources metadata in a MySQL database. Custom interface was written to manage the contents. 
Original database did not provide history, a good way to roll back bad changes or ability to edit same resource by multiple users.
That's why they moved to XML file-based asset database, managed under Perforce.
The asset conditioning pipeline consists of a set of exporters, compilers and linkers, all run from command line. There are 2 types of resource which can be all kinds of data objects. These types are actors (skeletons, meshes, materials and/or animations) and scenes (static back-ground meshes, materials and textures, level-layout info).
These command-line tools query the database to determine exactly how to build the actor or level in question. This includes information on how to export the assets from DCC tools like Maya and Photoshop, how to process the data, and how to package it into binary .pak files that can be loaded by the game engine. This is much simpler than in many engines, where resources have to be exported manually by the artists—a time-consuming, tedious and error-prone task.
Benefits:
- Granular resources - Resources can be managed in terms of logical entities in game - meshes, materials, skeletons etc.
- The necessary features - Naughty Dog implemented only necessary features
- Obvious mapping to source files - user can quickly see which .ma/.psd file is mapped to which source file
- Easy to change how DCC data is exported and processed - just click on the resource and change properties in database GUI
- Easy to build assets - just type command line command, dependency system will cover rest
Drawbacks:
- Lack of visualization tool - The only way to preview an asset is to load it into the game or the model/animation viewer (which is really just a special mode of the game itself).
- The tools aren't fully integrated - Naughty Dog uses one tool to lay out levels, another to manage the majority of resources in the resource database, and a third to set up materials and shaders (this is not part of the resource database front end). Building the assets is done on the command line.
##### OGRE's Resource Manager System
In OGRE, a simple, consistent interface is used to load virtually any kind of resource. And the system has been designed with extensibility in mind. Any programmer can quite easily implement a resource manager for a brand new kind of asset and integrate it easily into OGRE’s resource framework.
One of the drawbacks of OGRE’s resource manager is that it is a runtimeonly solution. OGRE lacks any kind of offline resource database.
In summary, OGRE’s runtime resource manager is powerful and welldesigned. But, OGRE would benefit a great deal from an equally powerful and modern resource database and asset conditioning pipeline on the tools side.
##### Microsoft's XNA
Although it was retired by Microsoft in 2014, it’s still a good resource for learning about game engines. XNA’s resource management system is unique, in that it leverages the project management and build systems of the Visual Studio IDE to manage and build the assets in the game as well. XNA’s game development tool, Game Studio Express, is just a plug-in to Visual Studio Express.
### The Asset Conditioning Pipeline
Asset Conditioning Pipeline, Resource Conditioning Pipeline or simply tool chain is used to change native format of Maya, ZBrush, Photoshop etc to format suitable for engine.
Usually, changing involve 3 steps:
- Exporters - We need some exporter to get data out of original program like Maya with reasonable format to reading in later steps. It is usually done by providing plugin in program itself (in Maya, Zbrush or whatever).
- Resource compilers - We often have to “massage” the raw data exported from a DCC application in various ways in order to make them gameready. For example, we might need to rearrange a mesh’s triangles into strips, or compress a texture bitmap, or calculate the arc lengths of the segments of a Catmull-Rom spline. Not all types of resources need to be compiled—some might be game-ready immediately upon being exported.
- Resource linkers - Multiple resources have to be linked in one package. For example, when building 3D model, we might have to combine data from multiple mesh files, material files, skeleton file and multiple animation files. 
##### Resource Dependencies and Build Rules
Resource dependencies have an impact on the order in which assets should be processed. For example, we should process skeleton prior to animations.
Build dependencies revolve not only around changes to the assets themselves, but also around changes to data formats.
Every asset conditioning pipeline requires a set of rules that describe the interdependencies between the assets, and some kind of build tool that can use this information to ensure that the proper assets are built, in the proper order, when a source asset is modified. Some game teams roll their own build system. Others use an established tool, such as make. Whatever solution is selected, teams should treat their build dependency system with utmost care.