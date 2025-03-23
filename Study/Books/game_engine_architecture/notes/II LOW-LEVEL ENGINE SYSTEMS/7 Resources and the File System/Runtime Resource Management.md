### Responsibilities of the RRM
- Ensures that only one copy of each unique resource exists in memory at any given time
- Manages the lifetime of each resource
- Loads needed resources and unloads unnecessary resources
- Handles loading of composite resources (resource made out of other resources)
- Maintains referential integrity - when loading composite resource, the resource manager must ensure that all necessary sub-resources are loaded and it must patch in all of the cross references properly. 
- Manages the memory usage of loaded resources
- Permits custom processing to be performed on a resource after it has been loaded, on a per-resource-type basis. This process is sometimes known as logging in or load-initializing the resource
- Usually provides a single unified interface through which wide variety of resources types can be managed. Ideally extensible!
- Handles streaming
### Resource file and directory organization
Usually game engine resource files are contained within tree like directory organization. This system is designed to benefit users, resource manager does not care where is the resource in this hierarchy. 
Other engines packs multiple resources together in single file, as ZIP or some other composite file.
When loading data from files, the three biggest costs are seek times (i.e., moving the read head to the correct place on the physical media), the time required to open each individual file, and the time to read the data from the file into memory.
Primary benefits of ZIP format:
- Open format
- The virtual files within ZIP archive "remember" their relative paths - This means that a ZIP archive “looks like” a raw file system for most intents and purposes.
- ZIP archives may be compressed - speed up load time, reduces amount of taken disk space.
- ZIP archives are modular - One particularly elegant application of this idea is in product localization. All of the assets that need to be localized (such as audio clips containing dialogue and textures that contain words or region-specific symbols) can be placed in a single ZIP file, and then different versions of this ZIP file can be generated, one for each language or region. To run the game for a particular region, the engine simply loads the corresponding version of the ZIP archive
### Resource File Formats
Resources have different formats.
> Sometimes a single file format can be used to house many different types of assets. For example, the Granny SDK by Rad Game Tools (http://www. radgametools.com) implements a flexible open file format that can be used to store 3D mesh data, skeletal hierarchies and skeletal animation data. (In fact the Granny file format can be easily repurposed to store virtually any kind of data imaginable.)

Many game engine programmers rolls their own format for different reasons. Also, many game engine endeavor to do as much offline processing as possible.
### Resource GUIDs
Every resource need *Globally unique identifier (GUID).* Most common choice is to store it as filepath string or 32bit hash. 
However, a file system path is by no means the only choice for a resource GUID. Some engines use a less-intuitive type of GUID, such as a 128-bit hash code, perhaps assigned by a tool that guarantees uniqueness.
In other engines, using a file system path as a resource identifier is infeasible.
In Unreal, a resource GUID is formed by concatenating the (unique) name of the package file with the in-package path of the resource in question.
### The resource registry
To ensure one copy of resource is loaded at any given time, resource manager maintain some kind of registry (dictionary with key-value).
When a resource is requested by the game, the resource manager looks up the resource by its GUID within the resource registry. If the resource can be found, a pointer to it is simply returned. If the resource cannot be found, it can either be loaded automatically or a failure code can be returned.
Loading resource file if it was not found has some serious problems (but some engines do it). It is slow operation, involves locating and opening a file on a disk, reading potentially large data block and possibly performing post-load initialization.
Two alternative approaches:
- Resource loading might be disallowed completely during active gameplay. They are loaded prior to gameplay (loading screen).
- Streaming resources.
### Resource lifetime
Lifetime is defined as the time period between resource is first loaded into memory and when this memory is reclaimed for another purposed.
Resource manager should manage resources lifetimes.
Each resource has its own lifetime:
- Infinite lifetime - global assets, global resources. Player meshes, materials, textures, core animations etc.
- Lifetime that matches particular game level
- Lifetime that is shorter than duration of the level in which they are found - assets for cinematic at the beginning of level
- Lifetime "as be played" - music, ambient sound and effects are streamed as they play. Each byte persists in memory for a tiny fraction of second. Such assets are typically loaded in chunks of a size that matches the underlying hardware’s requirements. For example, a music track might be read in 4 KiB chunks, because that might be the buffer size used by the low-level sound system
The question of when to unload a resource and reclaim its memory is not so easily answered. The problem is that many resources are shared across multiple levels. We don’t want to unload a resource when level X is done, only to immediately reload it because level Y needs the same resource. 
One solution to this problem is to reference-count the resources. 
Whenever a new game level needs to be loaded, the list of all resources used by that level is traversed, and the reference count for each resource is incremented by one. Next, we traverse the resources of any unneeded levels and decrement their reference counts by one; any resource whose reference count drops to zero is unloaded. Finally, we run through the list of all resources whose reference count just went from zero to one and load those assets into memory.
![[AllocationTable.png]]
### Memory Management for resources
The destination of every resource is not always the same. Certain types of resources must reside in video RAM. Typical examples are textures, vertex buffers, index buffers and shader code. Most other resources can reside in main RAM, but different kinds of resources might need to reside within different address range.. For example, a resource that is loaded and stays resident for the entire game (global resources) might be loaded into one region of memory, while resources that are loaded and unloaded frequently might go somewhere else.
**The design of a game engine’s memory allocation subsystem is usually closely tied to that of its resource manager. Sometimes we will design the resource manager to take best advantage of the types of memory allocators we have available, or vice versa—we may design our memory allocators to suit the needs of the resource manager.**
As mentioned, fragmentation should be avoided.
##### Heap-Based Resource Allocation
One approach is to ignore fragmentation issues and use general-purpose heap allocator. On system with support of virtual memory allocation, physical memory will be fragmented, but OS ability to map noncontiguous pages of physical RAM into contiguous virtual memory space helps to mitigate some effects of fragmentation.
Fragmentation then becomes problem on console.
##### Stack-Based Resource Allocation
Stack allocator can be used to load resources if the following two conditions are met:
- game is linear and level-centric (player watches loading screen, plays level, watches another loading screen, plays another level)
- each level fits fully into memory

When the game first starts up, the global resources are allocated first. The top of the stack is then marked, so that we can free back to this position later. To load a level, we simply allocate its resources on the top of the stack. When the level is complete, we can simply set the stack top back to the marker we took earlier, thereby freeing all of the level’s resources in one fell swoop without disturbing the global resources. This process can be repeated for any number of levels, without ever fragmenting memory.
A double-ended stack allocator can be used to augment this approach. Two stacks are defined within a single large memory block. One grows up from the bottom of the memory area, while the other grows down from the top. As long as the two stacks never overlap, the stacks can trade memory resources back and forth naturally—something that wouldn’t be possible if each stack resided in its own fixed size block.
> On Hydro Thunder, Midway used a double-ended stack allocator. The lower stack was used for persistent data loads, while the upper was used for temporary allocations that were freed every frame. Another way a doubleended stack allocator can be used is to ping-pong level loads. Such an approach was used at Bionic Games, Inc. for one of their projects. The basic idea is to load a compressed version of level B into the upper stack, while the currently active level A resides (in uncompressed form) in the lower stack. To switch from level A to level B, we simply free level A’s resources (by clearing the lower stack) and then decompress level B from the upper stack into the lower stack. Decompression is generally much faster than loading data from disk, so this approach effectively eliminates the load time that would otherwise be experienced by the player between levels.

##### Pool-Based resource allocation
Common technique that support streaming is to load resource data in equally sized chunks. Chunks are the same size, so we can use *pool allocator*. 
Large contiguous data structures must be avoided in favor of data structures that are either small enough to fit within a single chunk or do not require contiguous RAM to work properly.
Each chunk in the pool is typically associated with a particular game level. (One simple way to do this is to give each level a linked list of its chunks.) This allows the engine to manage the lifetimes of each chunk appropriately, even when multiple levels with different life spans are in memory concurrently.
![[ChunkAllocation.png]]
Big drawback is wasted space. Because usually not every data piece matches chunk size, there is unused space in chunk left.
##### Resource Chunk Allocators
One way to limit the effects of wasted chunk memory is to set up a special memory allocator that can utilize the unused portions of chunks.
A resource chunk allocator is not particularly difficult to implement. We need only maintain a linked list of all chunks that contain unused memory, along with the locations and sizes of each free block. We can then allocate from these free blocks in any way we see fit. For example, we might manage the linked list of free blocks using a general-purpose heap allocator. Or we might map a small stack allocator onto each free block; whenever a request for memory comes in, we could then scan the free blocks for one whose stack has enough free RAM and then use that stack to satisfy the request.
**What will happen if chunk with allocated free space will get freed?**
A simple solution to this problem is to only use our free-chunk allocator for memory requests whose lifetimes match the lifetime of the level with which a particular chunk is associated. In other words, we should only allocate memory out of level A’s chunks for data that is associated exclusively with level A and only allocate from B’s chunks memory that is used exclusively by level B.
##### Sectioned Resource Files
Another useful idea related to chunky resources is the concept of file sections. Typical resource file might contain between one and four sections. 
One section might contain data that is destined for main RAM, while another section might contain video RAM data. Another section could contain temporary data that is needed during the loading process but is discarded once the resource has been completely loaded. Yet another section might contain debugging information. This debug data could be loaded when running the game in debug mode, but not loaded at all in the final production build of the game.
Example: http://www.radgametools.com
### Composite Resources and Referential Integrity
In general, a game’s resource database can be represented by a directed graph of interdependent data objects.
Cross-reference between data objects can be *internal* (reference between two objects withing a single file) or *external* (reference to an object in a different file). 
**This distinction is important since internal and external cross references are usually implemented differently.** 
When visualizing a game's resource database, we can draw dotted lines surrounding individual files to make the internal/external distinction clear - any edge of the graph that crosses a dotted line file boundary is external reference.
![[ResourcesReferences.png]]
### Handling Cross-References between Resources
Cross references between resources is challenging.
In C++, cross-ref between two data objects is usually implemented via pointer or reference.
Pointers are just memory addresses—they lose their meaning when taken out of the context of the running application. In fact, memory addresses can and do change even between runs of the same application. When storing data to a disk file, we cannot use pointers to describe inter-object dependencies.
##### GUIDs as cross-references
Good approach is to store each cross-reference as a string or hash code containing the unique id of the referenced object. This implies that every resource which might be cross-referenced must have globally unique identifier or *GUID.*
To make this work, the runtime resource manager maintains a global resource look-up table. Whenever resource is loaded into memory, a pointer to that object is stored in the table with its GUID as the look-up key. After all resource objects have been loaded and their entries added to the table, we can make a pass over all of the objects and convert all of their cross-references into pointers, by looking up the address of each object in global resource look-up table via GUIDs.
##### Pointer Fix-Up tables
![[Pasted image 20250323214410.png]]
Other approach is to data into binary objects and change pointers into file offsets.
To store group of objects into a binary file, we need to visit each node once in an arbitrary order and write each object's memory image into file sequentially. 
During the process of writing binary file image, each pointer gets converted into an offset and store in place of the pointer.
**We can simply overwrite the pointers with their offsets, because the offsets never require more bits to store than the original pointers. In effect, an offset is the binary file equivalent of a pointer in memory.**
Conversion into pointers will be needed later, when file is loaded into memory. It is called *pointer fix-up*.
When the file’s binary image is loaded, the objects contained in the image retain their contiguous layout, so it is trivial to convert an offset into a pointer. We merely add the offset to the address of the file image as a whole.
```
u8* convertOffsetToPointer(u32 objectOffset, u8* pAddressOfFileImage)
{
	u8* pObject = pAddressOfFileImage + objectOffset;
	return pObject;
}
```

![[Pasted image 20250323215358.png]]
*Pointer fix up^*
**How to find all of the pointers that require conversion?**
This problem is solved at the time the binary file is written. The code that writes out the images of the data objects has knowledge of the data types and classes being written. It has knowledge of the locations of all the pointers within each object. The locations of the pointers are stored into a simple table known as a pointer fix-up table. This table is written into the binary file along with the binary images of all the objects. Later, when the file is loaded into RAM again, the table can be consulted in order to find and fix up every pointer. The table itself is just a list of offsets within the file—each offset represents a single pointer that requires fixing up.
##### Storing C++ objects as binary images: Constructors
Important step when loading C++ objects from a binary file is to ensure to call their constructors. 
There are two common solutions.
First, you can simply decide not to support C++ (decide on PODS or POD).
Second, you can save off a table containing the offsets of all non-PODS objects in your binary image along with some indication of which class each object is an instance of.
Later, you can iterate and call proper constructor via new keyword.
```
void* pObject = ConvertOffsetToPointer(objectOffset, pAddressOfFileImage);
::new (pObject) ClassName;
```
##### Handling external references
Presented above solutions can work for internal references.
In this simple case, you can load the binary image into memory and then apply the pointer fix-ups to resolve all the cross-references.
To successfully represent an external cross-reference, we must specify not only the offset or GUID of the data object in question, but also the path to the resource file in which the referenced object resides.
Key to load multi-file composite is to load all the interdependent files first.
This can be done by loading one resource file and then scanning through its table of cross-references and loading any externally referenced files that have not already been loaded. As we load each data object into RAM, we can add the object’s address to the master look-up table. Once all of the interdependent files have been loaded and all of the objects are present in RAM, we can make a final pass to fix up all of the pointers using the master look-up table to convert GUIDs or file offsets into real addresses.
### Post-Load initialization
Ideally, each and every resource would be prepared by offline tool, so that it is ready for use the moment it has been loaded into memory.
Many types of resources though, needs at least some "massaging".
This will be called *post-load initialization.*
Comes in two variables:
- In some cases, post-load initialization is an unavoidable step. For example, on a PC, the vertices and indices that describe a 3D mesh are loaded into main RAM, but they must be transferred into video RAM before they can be rendered. This can only be accomplished at runtime, by creating a Direct X vertex buffer or index buffer, locking it, copying or reading the data into the buffer and then unlocking it.
- In other cases, the processing done during post-load initialization is avoidable (i.e., could be moved into the tools), but is done for convenience or expedience. For example, a programmer might want to add the calculation of accurate arc lengths to our engine’s spline library. Rather than spend the time to modify the tools to generate the arc length data, the programmer might simply calculate it at runtime during post-load initialization. **Later, when the calculations are perfected, this code can be moved into the tools, thereby avoiding the cost of doing the calculations at runtime.**
