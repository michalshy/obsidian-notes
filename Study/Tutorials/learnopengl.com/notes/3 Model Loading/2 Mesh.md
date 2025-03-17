# Construct
When model is loaded with Assimp, it is represented as library data structure. Eventually, we should transform this data to format understandable by OpenGL.
A mesh should at least contain set of vertices, where each vertex has position vector, normal vector and texture coordinate vector. A mesh should also contain indices for indexed drawing and material data in form of textures.
```
struct Vertex 
{ 
	glm::vec3 Position; 
	glm::vec3 Normal; 
	glm::vec2 TexCoords; 
};
```
Next to vertex struct, we need Texture struct.
```
struct Texture 
{ 
	unsigned int id; 
	string type; 
};
```
we store id of the texture and its type (diffuse, specular).
Now we can construct Mesh class
```
class Mesh {
	public:
		// mesh data 
		vector<Vertex> vertices;
		vector<unsigned int> indices; 
		vector<Texture> textures; 
		Mesh(vector<Vertex> vertices, vector<unsigned int> indices, vector<Texture> textures); 
		void Draw(Shader &shader); 
	private: 
		// render data 
		unsigned int VAO, VBO, EBO; 
		void setupMesh();
};
```
The function content of the constructor is pretty straightforward. We simply set the class's public variables with the constructor's corresponding argument variables.
```
Mesh(vector<Vertex> vertices, vector<unsigned int> indices, vector<Texture> textures) 
{ 
	this->vertices = vertices; 
	this->indices = indices; 
	this->textures = textures; 
	setupMesh(); 
}
```
# Initialize
There is large lists of mesh data that can be used for rendering.
Setup of appropriate buffers is needed and specifying the vertex shader layout via vertex attribute pointers.
```
void setupMesh() 
{ 
	glGenVertexArrays(1, &VAO);
	glGenBuffers(1, &VBO); 
	glGenBuffers(1, &EBO); 
	glBindVertexArray(VAO); 
	glBindBuffer(GL_ARRAY_BUFFER, VBO); 
	glBufferData(GL_ARRAY_BUFFER, vertices.size() * sizeof(Vertex), &vertices[0], GL_STATIC_DRAW); 
	glBindBuffer(GL_ELEMENT_ARRAY_BUFFER, EBO);
	glBufferData(GL_ELEMENT_ARRAY_BUFFER, indices.size() * sizeof(unsigned int), &indices[0], GL_STATIC_DRAW); 
	// vertex positions
	glEnableVertexAttribArray(0); 
	glVertexAttribPointer(0, 3, GL_FLOAT, GL_FALSE, sizeof(Vertex), (void*)0); 
	// vertex normals 
	glEnableVertexAttribArray(1); 
	glVertexAttribPointer(1, 3, GL_FLOAT, GL_FALSE, sizeof(Vertex), (void*)offsetof(Vertex, Normal)); 
	// vertex texture coords 
	glEnableVertexAttribArray(2); 
	glVertexAttribPointer(2, 2, GL_FLOAT, GL_FALSE, sizeof(Vertex), (void*)offsetof(Vertex, TexCoords)); 
	glBindVertexArray(0); 
}
```
Structs are sequential in memory (e.g:)
```
Vertex vertex; 
vertex.Position = glm::vec3(0.2f, 0.4f, 0.6f); 
vertex.Normal = glm::vec3(0.0f, 1.0f, 0.0f); 
vertex.TexCoords = glm::vec2(1.0f, 0.0f); 
// = [0.2f, 0.4f, 0.6f, 0.0f, 1.0f, 0.0f, 1.0f, 0.0f];
```
Thanks to this useful property we can directly pass a pointer to a large list of Vertex structs as the buffer's data and they translate perfectly to what glBufferData expects as its argument:
`glBufferData(GL_ARRAY_BUFFER, vertices.size() * sizeof(Vertex), &vertices[0], GL_STATIC_DRAW);`
The last needed function is `Draw` function. Before rendering we have to bind appropriate textures.
However, this is somewhat difficult since we don't know from the start how many (if any) textures the mesh has and what type they may have. So how do we set the texture units and samplers in the shaders?
To solve the issue we're going to assume a certain naming convention: each diffuse texture is named `texture_diffuseN`, and each specular texture should be named `texture_specularN` where `N` is any number ranging from `1` to the maximum number of texture samplers allowed. Let's say we have 3 diffuse textures and 2 specular textures for a particular mesh, their texture samplers should then be called:
```
uniform sampler2D texture_diffuse1; 
uniform sampler2D texture_diffuse2; 
uniform sampler2D texture_diffuse3; 
uniform sampler2D texture_specular1; 
uniform sampler2D texture_specular2;
```
The resulting drawing code:
```
void Draw(Shader& shader)
{
	unsigned int diffuseNr = 1; 
	unsigned int specularNr = 1; 
	for(unsigned int i = 0; i < textures.size(); i++) 
	{ 
		glActiveTexture(GL_TEXTURE0 + i); 
		// activate proper texture unit before binding 
		// retrieve texture number (the N in diffuse_textureN) 
		string number; 
		string name = textures[i].type; 
		if(name == "texture_diffuse") 
			number = std::to_string(diffuseNr++); 
		else if(name == "texture_specular") 
			number = std::to_string(specularNr++);
		shader.setInt(("material." + name + number).c_str(), i);
		glBindTexture(GL_TEXTURE_2D, textures[i].id); 
	} 
	glActiveTexture(GL_TEXTURE0); 
	// draw mesh 
	glBindVertexArray(VAO); 
	glDrawElements(GL_TRIANGLES, indices.size(), GL_UNSIGNED_INT, 0);
	glBindVertexArray(0);
}
```
Note that we increment the diffuse and specular counters the moment we convert them to `string`. In C++ the increment call: `variable++` returns the variable as is and **then** increments the variable while `++variable` **first** increments the variable and **then** returns it. In our case the value passed to `std::string` is the original counter value. After that the value is incremented for the next round.