Example of color definition
`glm::vec3 coral(1.f, 0.5f, .31f);`

![[Pasted image 20250308165903.png]]
Object does not have color, they reflect it. As you can see, sun light is white light (so collection of all colors). If we will shine it on object above, it will absorb most of colors and reflect some of them with varying intensity.
> Technically it's a bit more complicated, but we'll get to that in the PBR chapters.

When defining light in OpenGL, we specify source color, thus resulting color would be calculated as follow:
```
glm::vec3 lightColor(1.0f, 1.0f, 1.0f); 
glm::vec3 toyColor(1.0f, 0.5f, 0.31f); 
glm::vec3 result = lightColor * toyColor; // = (1.0f, 0.5f, 0.31f);
```
## Begin

Strip down vertex shader from previous chapters:
```
#version 330 core 
layout (location = 0) in vec3 aPos;

uniform mat4 model; 
uniform mat4 view; 
uniform mat4 projection; 
void main() 
{ 
	gl_Position = projection * view * model * vec4(aPos, 1.0); 
}
```
Creation of light source cube, and defining it parameters requires us to create special VAO for this object, since we want to differentiate cube object from light cube object.

```
unsigned int lightVAO;
glGenVertexArrays(1, &lightVAO);
glBindVertexArray(lightVAO);

glBindBuffer(GL_ARRAY_BUFFER, VBO);
glVertexAttribPointer(0, 3, GL_FLOAT, GL_FALSE, 3 * sizeof(float), (void*)0);
glEnableVertexAttribArray(0);
```
==Create fragment shader for both cube and light cube.
```
#version 330 core
out vec4 FragColor;

uniform vec3 objectColor;
uniform vec3 lightColor;

void main()
{
	FragColor = vec4(lightColor * objectColor, 1.0);
}
```
Set parameters.
```
// don't forget to use the corresponding shader program first (to set the uniform) 
lightingShader.use(); 
lightingShader.setVec3("objectColor", 1.0f, 0.5f, 0.31f); 
lightingShader.setVec3("lightColor", 1.0f, 1.0f, 1.0f);
```
We need to create second shader for light cube, since we don't want its color to be affected by any calculations. This will show us that  light cube looks really like the *source* of the color and light.

Shader for light cube.
```
#version 330 core 
out vec4 FragColor; 
void main() 
{ 
	FragColor = vec4(1.0); 
	// set all 4 vector values to 1.0 
}
```
Usually light cube is kept away from the scene, but in this example we will show it close to object so it is visible that its constantly white
`glm::vec3 lightPos(1.2f, 1.0f, 2.0f);`

We then translate light source cube to light source position and scale it down
```
model = glm::mat4(1.0f); 
model = glm::translate(model, lightPos); 
model = glm::scale(model, glm::vec3(0.2f));
```
