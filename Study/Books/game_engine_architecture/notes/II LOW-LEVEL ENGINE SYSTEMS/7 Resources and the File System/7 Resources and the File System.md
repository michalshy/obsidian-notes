Games are multimedia experiences. Game engine therefore has to load and manage all kind of media - texture bitmaps, 3D mesh data, animations, audio clips, collision and physics data etc.
Also, engine needs to ensure that only one copy of each media file is loaded.
**Most game engines employ some kind of resource manager.**
Every resource manager make use of the file system. Game engines usually wrap the native file system in an engine-specific API, due to:
- Engine might be cross platform, which allows to platform specific file system implementation
- Provided system calls and native functions might not be sufficient for engine needs, for example many engines support file streaming (the ability to load data on the fly)
- Console game engines also need to provide access to variety of removable and non-removable media, the differences between various kinds of media might be hidden behind API
# [[7.1 File System]]
# [[7.2 The Resource Manager]]