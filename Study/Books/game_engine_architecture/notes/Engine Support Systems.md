# Order of subsystems start up and shut down
## Begin
Subsystems are usually initialized in some desired order.
In C++, we would want to create instances before main, although static created objects provide no order of creation. If one system would be dependent on another, we would have no control over creation and destruction, thus creating those systems like this:
```
class RenderManager
{
public:
	RenderManager()
	{
		//start up
	}
	~RenderManager()
	{
		//shut down
	}
}
```
would not work.

In C++, static variable declared in function will be defined on first call of this function. That is why we can use other approach.
```
class RenderManager
{
public:
	//Get the one and only instance
	static RenderManager& get()
	{
		static RenderManager sSingleton;
		return sSingleton;
	}
	RenderManager()
	{
		//start up other managers
		VideoManager::get();
		...
	}
	~RenderManager()
	{
		//shut down
	}
}
```

Singleton might also be allocated dynamically
```
static RenderManager& get()
{
	static RenderManager* gpSingleton = nullptr;
	if(gpSingleton == nullptr)
	{
		gpSingleton = new RenderManager();
	}
	ASSERT(gpSingleton);
	return *gpSingleton
	
}

```
Still, we have no control over destructions.
The design pattern is dangerous - programmer might not expect get to allocate some memory dynamically.
## Simple Approach that works

If programmer wants to stick with singleton -> make explicit functions to start up in classes, and make constructors do nothing.

```
class RenderManager()
{
public:
	RenderManager()
	{
		//do nothing
	}
	~RenderManager()
	{
		//do nothing
	}
	void startUp()
	{
		//start manager
	}
	void shutDown()
	{
		//shut manager
	}
}
class PhysicsManager { /* similar */ }
class AnimationManager { /* similar */ }
class MemoryManager { /* similar */ }

...

RenderManager gRenderManager;
PhysicsManager gPhysicsManager;
AnimationManager gAnimationManager;
MemoryManager gMemoryManager;

...

int main()
{

	gRenderManager.startUp();
	// start rest

	gSimulationManager.run()

	gRenderManager.shutDown();
	// shut everything (reverse order)

	return 0;
}
```

Pros:
- Simple and easy to implement
- Explicit
- Easy to debug and maintain
Cons:
- watch out for **ORDER**

# Memory management

## Affecting performance

- malloc and new are really **slow**
- if data is not presented in **small, contiguious blocks** it is inefficient

> Even the most efficient algorithm, coded with the utmost care, can be brought to its knees if the data upon which it operates is not laid out efficiently in memory.

## Optimization of dynamic memory allocation

Keep heap allocations to a minimum, and never allocate from the heap within a tight loop.

Providing custom allocator might be useful since it omits kernel calls.

### Stack allocator
In game engines used to load levels. Provide loading in stack-like fashion. Easy to implement
1. Allocate big chunk of memory (new, malloc or **global array of bytes**)
2. Store pointer to the top of the stack. All data below is used, all above is free.
3. Allocating moves pointer up.
4. Rolling should be performed to the point between two allocated blocks.

Example:
```
class StackAllocator
{
public:
	//stack marker: current top, 
	//you can only roll back to a marker, not to arbitrary locations
	typedef U32 Marker;
	
	//constructs with defined size
	explicit StackAllocator(U32 stackSize_bytes);

	 //allocates a new block of the given size from stack top
	 void* alloc(U32 size_bytes);

	//returns marker to current size top
	Marker getMarker();
	
	//rolls to previous marker
	void freeToMarker(Marker marker);

	//clears stack
	void clear();
}
```
![[Pasted image 20250307230702.png]]