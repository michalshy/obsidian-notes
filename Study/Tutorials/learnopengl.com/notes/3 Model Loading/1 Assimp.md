*Open Asset Import Library* is able to import dozens of different file formats.
When importing a model via Assimp it loads the entire model into a _scene_ object that contains all the data of the imported model/scene. Assimp then has a collection of nodes where each node contains indices to data stored in the scene object where each node can have any number of children. A (simplistic) model of Assimp's structure is shown below:
![[Pasted image 20250317175702.png]]
Features:
- All the data of the scene/model are contained in the *Scene* object like all the materials and meshes.
- The Root node of the scene may contain children nodes and could have a set of indices that point to mesh data in the scene object's `mMeshes` array. Scene's `mMeshes` array actually contains *Mesh* objects, the values in `mMeshes` array are only indices for scene's meshes array.
- A mesh object contains all the relevant data required for rendering, think of vertex positions, normal vectors, texture coordinates, faces, and the materials.
- A mesh contains several faces. *A face* represents a render primitive (triangle, square, point). Face contains indices of the vertices that form a primitive.
- Mesh also links to a Material object that hosts several functions to retrieve the material properties of an object.

**Mesh:**
When modeling objects, artists generally do not create entire model out of single shape. Usually, each model has several sub-models/shapes that it consist of. Each of those shape is called **mesh**. 

https://github.com/assimp/assimp/blob/master/Build.md