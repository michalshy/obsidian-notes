Phong lighting model consist of 3 components: ambient, diffuse and specular lighting.
![[Pasted image 20250308191439.png]]
- ==Ambient lighting ==- light of the world, stars, moon, sun, light which occurs on whole object.
- ==Diffuse lighting== - simulates directional impact a light object has on an object. The more a part of an object faces light source, the brighter it becomes.
- ==Specular lighting== - simulates the bright spot of light on shiny objects. It is more inclined to color of light than to color of object.
## Ambient light
> Light usually does not come from a single light source, but from many light sources scattered all around us, even when they're not immediately visible. One of the properties of light is that it can scatter and bounce in many directions, reaching spots that aren't directly visible; light can thus _reflect_ on other surfaces and have an indirect impact on the lighting of an object. Algorithms that take this into consideration are called global illumination algorithms, but these are complicated and expensive to calculate.
==To start with simplistic model, global illumination model named **ambient lighting** will be introduced. 
To include it, final result color has to be multiplied by some constant (light) color.==
```
void main()
{
	float ambientStrength = 0.1;
	vec3 ambient = ambientStrength * lightColor;
	vec3 result = ambient * objectColor;
	FragColor = vec4(result, 1.0);
}
```
## Diffuse lighting

Gives the object more brightness the closer its fragments are to the light rays of light source.
![[Pasted image 20250308193218.png]]

If the light ray is perpendicular to the object's surface the light has greatest impact. To measure angle we use normal vectors. Angle between them may be easily calculated with dot product.
**THE LOWER THE ANGLE BETWEEN VECTORS, DOT PRODUCT IS CLOSER TO 1, at 90 degrees dot product is equal to 0.**

> [!NOTE] REMEMBER
> Note that to get (only) the cosine of the angle between both vectors we will work with _unit vectors_ (vectors of length `1`) so we need to make sure all the vectors are normalized, otherwise the dot product returns more than just the cosine

To calculate diffuse lighting we need:
- normal vector to the vertex surface
- directed light ray: direction vector between the light's position and fragment position
### Normal vectors
Since vertex has no surface (it is single point) we use surrounding vertices to calculate normal vector. Usually it is done for all vertices by calculating cross product.
**In this example updated vertices array were provided**.
#### updating vertex shader of cube
```
#version 330 core
layout (location = 0) in vec3 aPos;
layout (location = 1) in vec3 aNormal;
```
> [!NOTE] Important
> It may look inefficient using vertex data that is not completely used by the lamp shader, but the vertex data is already stored in the GPU's memory from the container object so we don't have to store new data into the GPU's memory. This actually makes it more efficient compared to allocating a new VBO specifically for the lamp.
Now we need to forward our Normal to fragment shader.
```
out vec3 Normal;
void main()
{
	gl_Position = projection * view * model * vec4(aPos, 1.0);
	Normal = aNormal;
}
```
in fragment shader
`in vec3 Normal`
### Calculating the diffuse color
Since the light position is static we can declare it as an uniform.
`uniform vec3 lightPos;`
And then update uniform in render loop (or outside if it does not change per frame).
`lightingShader.setVec3("lightPos", lightPos);`
To get fragment's position in the world space, we need to convert. To do this, we have to multiply vertex position by model matrix. Lets do it then.
```
out vec3 FragPos;
out vec3 Normal;
void main()
{
	FragPos = vec3(model * vec4(aPos, 1.0));
	Normal = aNormal;
	gl_Position = projection * view * model * vec4(FragPos, 1.0);
}
```
in fragment shader:
`in vec3 FragPos`

### Calculate position vectors
```
vec3 norm = normalize(Normal);
vec3 lightDir = normalize(lightPos - FragPos);
```
> [!NOTE] Uniting of vectors!
> When calculating lighting we usually do not care about the magnitude of a vector or their position; we only care about their direction. Because we only care about their direction almost all the calculations are done with unit vectors since it simplifies most calculations (like the dot product). So when doing lighting calculations, make sure you always normalize the relevant vectors to ensure they're actual unit vectors. Forgetting to normalize a vector is a popular mistake.

Next, we can finally calculate impact
```
float diff = max(dot(norm, lightDir), 0.0);
vec3 diffuse = diff * lightColor;
```

**! LIGHTING FOR NEGATIVE COLORS IS NOT DEFINED **

### Finally...
Now, that we have both ambient and diffuse, we add both colors to each other
```
vec3 result = (ambient + diffuse) * objectColor;
FragColor = vec4(result, 1.0);
```

![[Pasted image 20250308210035.png]]
Effect after this part.
## Disclaimer
We **transformerd** everything in fragment shader to world space for calculations, but not normal vector. Why not?
Normal vectors are only directions, they do not have homogeneous coordinate (w, 4th component). So if we want to multiply normal vector with model matrix we want to remove translation part of matrix (upper left 3x3) or set the w component to 0 and multiply with 4x4.
When model matrix performing non-uniform scale, it changes normal vector in a way that it is no longer normal to the surface **!!!!**.

If we want to fix it, we have to use special matrix called normal matrix. It cuts off wrong scaling.

> [!NOTE] Definition of normal matrix
> the transpose of the inverse of the upper-left 33 part of the model matrix

We can provide above definition in a *vertex shader*
`Normal = mat3(transpose(inverse(model))) * aNormal;`

> ==Inversing matrices is a costly operation for shaders, so wherever possible try to avoid doing inverse operations since they have to be done on each vertex of your scene. For learning purposes this is fine, but for an efficient application you'll likely want to calculate the normal matrix on the CPU and send it to the shaders via a uniform before drawing (just like the model matrix).==

## Specular Lighting
Similar to diffuse, specular is based on the light's direction and normal vector but **also on the view direction**, so from what direction the player looks at the fragment. This lighting is based on the reflective properties of surfaces. It is strongest wherever we would see the light reflected on the surface.
![[Pasted image 20250308215714.png]]
We calculate a reflection vector by reflecting light ray around normal vector.
Then we calculate the angular distance between reflection vector and view direction.
The smaller the angle, the stronger the impact of the specular light.
**The view vector is the one extra variable we need for specular lighting which we calculate using the viewer's world space position and fragment's position.**

> We chose to do the lighting calculations in world space, but most people tend to prefer doing lighting in view space. An advantage of view space is that the viewer's position is always at `(0,0,0)` so you already got the position of the viewer for free. However, I find calculating lighting in world space more intuitive for learning purposes. If you still want to calculate lighting in view space you want to transform all the relevant vectors with the view matrix as well (don't forget to change the normal matrix too).

To pass view position, we just need to pass camera position vector to the shader.
`uniform vec3 viewPos;`
then set it:
`lighting.setVec3("viewPos", cam.Position);`

Next, we declare specular intensity so the highlight is not too bright:
`float specularStrength = 0.5;`
Next, calculation of view direction vector and the corresponding reflect vector along normal axis is possible:
```
vec3 viewDir = normalize(viewPos - FragPos);
vec3 reflectDir = reflect(-lightDir, norm);
```
We negate lightDir, since before we calculated lightPos - fragPos, so the pointing of vector is from fragment to light source and should be from fragment position to light source position, thus nagation.

Finally, we can calculate specular:
```
float spec = pow(max(dot(viewDir, reflectDir), 0.0), 32);
vec3 specular = specularStrength * spec * lightColor;
```
Power value is the shininess of the object. 
Then, we can sum results
```
vec3 result = (ambient + diffuse + specular) * objectColor; 
FragColor = vec4(result, 1.0);
```

**END OF THE CHAPTER**