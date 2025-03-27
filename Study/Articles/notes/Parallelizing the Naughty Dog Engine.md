- allow jobifying all code
- job can yield to another job in the middle of execution
	- player update with kick and wait for ray cast
- easy API
- no memory management
- one simple way to synchronize/chain jobs
- performance < ease-of-use of the API

# Fiber
- "partial thread"
	- user provided stack space
	- small context containing state of fiber and saved registers
- executed by a thread
- cooperative multi-threading
- minimal overhead

# Architecture of ND
- 6 worker threads
	- each locked to a CPU core
- A thread is the execution unit, the fiber is the context
- Job always executes within the context of a fiber
- Atomic counters for synchronization
- Fibers
	- 160 fibers (128 x 64 KiB stach, 32 x 512 KiB stack)
- 3 global job queues (Low/Normal/High)
	- No job stealing
![[Pasted image 20250327203429.png]]