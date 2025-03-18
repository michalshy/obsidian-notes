# Model class
Model will represent whole object. One mesh is only part of a model e.g. balcony, when model is whole house.
Model class
```
class Model 
{ 
public: 
	Model(char *path) 
	{ 
		loadModel(path); 
	} 
	void Draw(Shader &shader); 
private: 
	// model data 
	vector<Mesh> meshes; 
	string directory; 
	void loadModel(string path); 
	void processNode(aiNode *node, const aiScene *scene); 
	Mesh processMesh(aiMesh *mesh, const aiScene *scene); 
	vector<Texture> loadMaterialTextures(aiMaterial *mat, aiTextureType type, string typeName); 
};
```
The Draw function is nothing special and basically loops over each of the meshes to call their respective Draw
```
void Draw(Shader &shader) 
{ 
	for(unsigned int i = 0; i < meshes.size(); i++) 
		meshes[i].Draw(shader); 
}
```
# Import a 3D model into OpenGL
Include appropriate headers of assimp
```
#include <assimp/Importer.hpp> 
#include <assimp/scene.h> 
#include <assimp/postprocess.h>
```
Within loadModel, we use Assimp to load the model into a data structure of Assimp called a scene object.
Once we have that object, we can access all the data we need.
Assimp is abstracting from all the technical details of loading different file formats.
```
Assimp::Importer importer; 
const aiScene *scene = importer.ReadFile(path, aiProcess_Triangulate | aiProcess_FlipUVs);
```
`ReadFile` expects file path and couple post-processing options. By setting `aiProcess_Triangulate` we tell Assimp that if the model does not entirely consist of triangles, it should transform all the model's primitive shapes to triangles first. The `aiProcess_FlipUVs` flips the texture coordinates on the y-axis where necessary.
Few other useful options:
- `aiProcess_GenNormals`: creates normal vectors for each vertex if the model doesn't contain normal vectors.
- `aiProcess_SplitLargeMeshes`: splits large meshes into smaller sub-meshes which is useful if your rendering has a maximum number of vertices allowed and can only process smaller meshes.
- `aiProcess_OptimizeMeshes`: does the reverse by trying to join several meshes into one larger mesh, reducing drawing calls for optimization.
Complete loadModel function:
```
void loadModel(string path)
{
	Assimp::Importer import; 
	const aiScene *scene = import.ReadFile(path, aiProcess_Triangulate | aiProcess_FlipUVs); 
	if(!scene || scene->mFlags & AI_SCENE_FLAGS_INCOMPLETE || !scene->mRootNode) 
	{ 
		cout << "ERROR::ASSIMP::" << import.GetErrorString() << endl;
		return; 
	} 
	directory = path.substr(0, path.find_last_of('/')); 
	processNode(scene->mRootNode, scene);
}
```
After we load the model, we check if the scene and the root node of the scene are not null and check one of its flags to see if the returned data is incomplete. If any of these error conditions are met, we report the error retrieved from the importer's `GetErrorString` function and return. We also retrieve the directory path of the given file path.
If nothing went wrong, we want to process all of the scene's nodes. We pass the first node (root node) to the recursive processNode function. Because each node (possibly) contains a set of children we want to first process the node in question, and then continue processing all the node's children and so on. This fits a recursive structure, so we'll be defining a recursive function.
Each node contains a set of mesh indices where each index points to a specific mesh located in the scene object. We thus want to retrieve these mesh indices, retrieve each mesh, process each mesh, and then do this all again for each of the node's children nodes. The content of the processNode function is shown below:
```
void processNode(aiNode *node, const aiScene *scene) 
{ 
	// process all the node's meshes (if any) 
	for(unsigned int i = 0; i < node->mNumMeshes; i++) 
	{ 
		aiMesh *mesh = scene->mMeshes[node->mMeshes[i]];
		meshes.push_back(processMesh(mesh, scene)); 
	} 
	// then do the same for each of its children 
	for(unsigned int i = 0; i < node->mNumChildren; i++) 
	{ 
		processNode(node->mChildren[i], scene); 
	} 
}
```

> [!NOTE] IMPORTANT
> Careful reader may have noticed that we could forget about processing any of the nodes and simply loop through all of the scene's meshes directly, without doing all this complicated stuff with indices. The reason we're doing this is that the initial idea for using nodes like this is that it defines a parent-child relation between meshes. By recursively iterating through these relations, we can define certain meshes to be parents of other meshes.  
> An example use case for such a system is when you want to translate a car mesh and make sure that all its children (like an engine mesh, a steering wheel mesh, and its tire meshes) translate as well; such a system is easily created using parent-child relations.  
> Right now however we're not using such a system, but it is generally recommended to stick with this approach for whenever you want extra control over your mesh data. These node-like relations are after all defined by the artists who created the models.

# Assimp to mesh

Translating an `aiMesh` object to a mesh object of our own is not too difficult. All we need to do, is access each of the mesh's relevant properties and store them in our own object. The general structure of the processMesh function then becomes:
```
Mesh processMesh(aiMesh *mesh, const aiScene *scene) 
{ 
	vector<Vertex> vertices; 
	vector<unsigned int> indices; 
	vector<Texture> textures; 
	for(unsigned int i = 0; i < mesh->mNumVertices; i++) 
	{ 
		Vertex vertex; 
		// process vertex positions, normals and texture coordinates 
		[...] 
		vertices.push_back(vertex); 
	} 
	// process indices 
	[...] 
	// process material 
	if(mesh->mMaterialIndex >= 0) 
	{ 
		[...] 
	} 
	return Mesh(vertices, indices, textures); 
}
```
Processing a mesh is a 3-part process: retrieve all the vertex data, retrieve the mesh's indices, and finally retrieve the relevant material data. The processed data is stored in one of the `3` vectors and from those a Mesh is created and returned to the function's caller.
Retrieving the vertex data is pretty simple: we define a Vertex struct that we add to the vertices array after each loop iteration. We loop for as much vertices there exist within the mesh (retrieved via `mesh->mNumVertices`). Within the iteration we want to fill this struct with all the relevant data. For vertex positions this is done as follows:
```
glm::vec3 vector; 
vector.x = mesh->mVertices[i].x; 
vector.y = mesh->mVertices[i].y; 
vector.z = mesh->mVertices[i].z; 
vertex.Position = vector;
```
Similar for normals
```
vector.x = mesh->mNormals[i].x;
vector.y = mesh->mNormals[i].y; 
vector.z = mesh->mNormals[i].z; 
vertex.Normal = vector;
```
Texture coordinates are roughly the same, but Assimp allows a model to have up to 8 different texture coordinates per vertex. We're not going to use 8, we only care about the first set of texture coordinates. We'll also want to check if the mesh actually contains texture coordinates (which may not be always the case):
```
if(mesh->mTextureCoords[0]) // does the mesh contain texture coordinates? 
{ 
	glm::vec2 vec; 
	vec.x = mesh->mTextureCoords[0][i].x; 
	vec.y = mesh->mTextureCoords[0][i].y; 
	vertex.TexCoords = vec; 
} 
else 
	vertex.TexCoords = glm::vec2(0.0f, 0.0f);
```
# Indices
A face contains the indices of the vertices we need to draw. So now we have to iterate and add them to our vector 
```
for(unsigned int i = 0; i < mesh->mNumFaces; i++) 
{ 
	aiFace face = mesh->mFaces[i]; 
	for(unsigned int j = 0; j < face.mNumIndices; j++) 
		indices.push_back(face.mIndices[j]); 
}
```
# Material
Similar to nodes, a mesh only contains an index to a material object. To retrieve the material of a mesh, we need to index the scene's mMaterials array. The mesh's material index is set in its mMaterialIndex property, which we can also query to check if the mesh contains a material or not:
```
if(mesh->mMaterialIndex >= 0) 
{ 
	aiMaterial *material = scene->mMaterials[mesh->mMaterialIndex];
	vector<Texture> diffuseMaps = loadMaterialTextures(material, aiTextureType_DIFFUSE, "texture_diffuse"); 
	textures.insert(textures.end(), diffuseMaps.begin(), diffuseMaps.end()); 
	vector<Texture> specularMaps = loadMaterialTextures(material, aiTextureType_SPECULAR, "texture_specular");
	textures.insert(textures.end(), specularMaps.begin(), specularMaps.end());
}
```
The loadMaterialTextures function iterates over all the texture locations of the given texture type, retrieves the texture's file location and then loads and generates the texture and stores the information in a Vertex struct. It looks like this:
```
vector<Texture> loadMaterialTextures(aiMaterial *mat, aiTextureType type, string typeName) 
{ 
	vector<Texture> textures; 
	for(unsigned int i = 0; i < mat->GetTextureCount(type); i++) 
	{ 
		aiString str; 
		mat->GetTexture(type, i, &str); 
		Texture texture; 
		texture.id = TextureFromFile(str.C_Str(), directory); 
		texture.type = typeName; 
		texture.path = str; 
		textures.push_back(texture); 
	} 
	return textures; 
}
```
We first check the amount of textures stored in the material via its GetTextureCount function that expects one of the texture types we've given. We retrieve each of the texture's file locations via the GetTexture function that stores the result in an `aiString`. We then use another helper function called TextureFromFile that loads a texture (with `stb_image.h`) for us and returns the texture's ID. You can check the complete code listing at the end for its content if you're not sure how such a function is written.
# An optimization
There is optimization to make.. Most scenes re-use several of textures. Loading of texture is not cheap operation.
So, we will store textures globally.
```
struct Texture 
{ 
	unsigned int id; 
	string type; 
	string path; // we store the path of the texture to compare with other textures 
};
```
Then we store all the loaded textures in another vector declared at the top of the model's class file as a private variable:
`vector<Texture> textures_loaded;`
In the loadMaterialTextures function, we want to compare the texture path with all the textures in the textures_loaded vector to see if the current texture path equals any of those. If so, we skip the texture loading/generation part and simply use the located texture struct as the mesh's texture. The (updated) function is shown below:

```
vector<Texture> loadMaterialTextures(aiMaterial *mat, aiTextureType type, string typeName) 
{ 
	vector<Texture> textures; 
	for(unsigned int i = 0; i < mat->GetTextureCount(type); i++) 
	{ 
		aiString str; 
		mat->GetTexture(type, i, &str); 
		bool skip = false; 
		for(unsigned int j = 0; j < textures_loaded.size(); j++) 
		{ 
			if(std::strcmp(textures_loaded[j].path.data(), str.C_Str()) == 0) 
			{ 
				textures.push_back(textures_loaded[j]); 
				skip = true; 
				break; 
			} 
		} 
		if(!skip) 
		{ 
			// if texture hasn't been loaded already, load it Texture texture; 
			texture.id = TextureFromFile(str.C_Str(), directory);
			texture.type = typeName; 
			texture.path = str.C_Str(); 
			textures.push_back(texture);
			textures_loaded.push_back(texture); 
			// add to loaded textures 
		} 
	} 
	return textures; 
}
```

For better understanding, read:
https://learnopengl.com/code_viewer_gh.php?code=src/3.model_loading/1.model_loading/model_loading.cpp

EOC