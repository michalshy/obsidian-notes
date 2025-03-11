Previous chapter showed us difference in materials for objects.
The thing is, usually one object has a lot of materials, not just one.

# Extending system with diffuse and specular maps.
These allow us to influence the diffuse (and indirectly the ambient component since they should be the same anyways) and the specular component of an object with much more precision.
## Diffuse maps
Actually, diffuse map is different name for the texture. It represents colors of an object for each fragment.
This time however we store the texture as a `sampler2D` inside the Material struct. We replace the earlier defined `vec3` diffuse color vector with the diffuse map.
> [!NOTE] Remember
> Keep in mind that `sampler2D` is a so called opaque type which means we can't instantiate these types, but only define them as uniforms. If the struct would be instantiated other than as a uniform (like a function parameter) GLSL could throw strange errors; the same thus applies to any struct holding such opaque types.

We also does not need any ambient color, since from now on it will be controlled by light.
```
struct Material {
	sampler2D diffuse;
	vec3 specular;
	float shininess;
};

in vec2 TexCoords;
```

> [!NOTE] For stubborns
> If you're a bit stubborn and still want to set the ambient colors to a different value (other than the diffuse value) you can keep the ambient `vec3`, but then the ambient colors would still remain the same for the entire object. To get different ambient values for each fragment you'd have to use another texture for ambient values alone.

Next, we can sample from the texture to retrieve the fragment's diffuse color;
`vec3 diffuse = light.diffuse * diff * vec3(texture(material.diffuse, TexCoords));`
Also, set ambient color similary.
`vec3 ambient = light.ambient * vec3(texture(material.diffuse, TexCoords));`
 **To get it working we do need to update the vertex data with texture coordinates, transfer them as vertex attributes to the fragment shader, load the texture, and bind the texture to the appropriate texture unit.**
Update vertex data: https://learnopengl.com/code_viewer.php?code=lighting/vertex_data_textures

We need to update vertex shader as well to pass TexCoords.
```
#version 330 core
layout (location = 0) aPos;
layout (location = 1) aNormal;
layout (location = 2) aTexCoords;

...

out vec2 TexCoords;
void main()
{
	...
	TexCoords = aTexCoords;
}
```

Before usage, we need to update pointers of VAO's and load texture.
We have to assign right texture before rendering and bind the container texture to this texture unit.
```
lightingShader.setInt("material.diffuse", 0); 
... 
glActiveTexture(GL_TEXTURE0); 
glBindTexture(GL_TEXTURE_2D, diffuseMap);
```

**FUNCTION FOR LOADING TEXTURES**
```
unsigned int LightLightingMaps::load_texture(char const * path)
{
	unsigned int textureID;
	glGenTextures(1, &textureID);

	int width, height, nrChannels;
	unsigned char* data;
	data = stbi_load("tex\\container2.png", &width, &height, &nrChannels, 0);
	if (data)
	{
		GLenum format;
		if (nrChannels == 1)
			format = GL_RED;
		else if (nrChannels == 3)
			format = GL_RGB;
		else if (nrChannels == 4)
			format = GL_RGBA;

		glBindTexture(GL_TEXTURE_2D, textureID);
		glTexImage2D(GL_TEXTURE_2D, 0, format, width, height, 0, format, GL_UNSIGNED_BYTE, data);
		glGenerateMipmap(GL_TEXTURE_2D);

		glTexParameteri(GL_TEXTURE_2D, GL_TEXTURE_WRAP_S, GL_REPEAT);
		glTexParameteri(GL_TEXTURE_2D, GL_TEXTURE_WRAP_T, GL_REPEAT);
		glTexParameteri(GL_TEXTURE_2D, GL_TEXTURE_MIN_FILTER, GL_LINEAR_MIPMAP_LINEAR);
		glTexParameteri(GL_TEXTURE_2D, GL_TEXTURE_MAG_FILTER, GL_LINEAR);
	}
	else
	{
		std::cout << "Failed to load tex" << std::endl;
	}
	stbi_image_free(data);
	return textureID;
}
```
## Specular maps
Provided cube is really shiny everywhere, when it should obviously be more shinier on the edges (steel parts) than on the wood.
Texture map might be use for specular highlights. Define white and black texture that defines the specular intensities of each part. Example:
![[Pasted image 20250311210720.png]]
Each pixel of the specular map can be displayed as a color vector where black is vec3(0) and white vec3(1). In the fragment shader we sample the texture and multiply with light's specular intensity.
> Technically wood also has specular highlights although with a much lower shininess value (more light scattering) and less impact, but for learning purposes we can just pretend wood doesn't have any reaction to specular light.

We have to load texture now, and bind it to appropriate texture unit before rendering:
```
lightingShader.setInt("material.specular", 1); 
... 
glActiveTexture(GL_TEXTURE1); 
glBindTexture(GL_TEXTURE_2D, specularMap);
```
Then update material
```
struct Material 
{ 
	sampler2D diffuse; 
	sampler2D specular; 
	float shininess; 
};
```
Lastly, we have to sample the specular map:
```
vec3 ambient = light.ambient * vec3(texture(material.diffuse, TexCoords)); 
vec3 diffuse = light.diffuse * diff * vec3(texture(material.diffuse, TexCoords));
vec3 specular = light.specular * spec * vec3(texture(material.specular, TexCoords)); 
FragColor = vec4(ambient + diffuse + specular, 1.0);
```
By using a specular map we can specify with enormous detail what parts of an object have _shiny_ properties and we can even control the corresponding intensity. Specular maps give us an added layer of control over lighting on top of the diffuse map.
> [!NOTE] Color of specular highlight
> If you don't want to be too mainstream you could also use actual colors in the specular map to not only set the specular intensity of each fragment, but also the color of the specular highlight. Realistically however, the color of the specular highlight is mostly determined by the light source itself so it wouldn't generate realistic visuals (that's why the images are usually black and white: we only care about the intensity).

