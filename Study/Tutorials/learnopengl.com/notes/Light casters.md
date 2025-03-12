A light source that casts light upon objects is called **light caster**.
There are three type of light casters to be discussed
- directional light
- point light
- spot light
### Directional Light
When a light source is far away, the rays coming from it are close to parallel to each other.
It looks like all the light rays are coming from same direction, no matter where object where the object/viewer is. When light source is *infinitely* far away, it is called a **directional light**. 
Example of this is sun.

Since all the light rays are parallel, position of the object does not matter.
**Because the light's direction vector stays the same, the lighting calculations will be similar for all objects in the scene.**
This light can be modeled by defining light direction vector instead of position vector. Calculations remain mostly the same.
```
struct Light
{
	//vec3 position;
	vec3 direction;
	vec3 ambient;
	vec3 diffuse;
	vec3 specular;
};
[...]
void main()
{
	vec3 lightDir = normalize(-light.direction);
	[...]
}
```
Negation of light direction is present since to this point light direction was direction **from fragment** toward light source, and generally preferred way to specify directional light is as global direction **from the light source.** After negation it is pointing **towards light source**. 
**All the next calculations are performed in similar way as in previous chapters.**

Revisit *Coordinate system* chapter and implement directional light.
```
for(unsigned int i = 0; i < 10; i++)
{
	glm::mat4 model = glm::mat4(1.0f);
	model = glm::translate(model, cubePosition[i]);
	float angle = 20.f * i;
	model = glm::rotate(model, glm::radians(angle), glm::vec3(1.f, 0.3f, 0.5f));
	lightShader.setMat4("model", model);

	glDrawArrays(GL_TRIANGLES, 0, 36);
}

```
Specify direction
`lightShader.setVec3("light.direction", -0.2f, -1.f, -0.3f);`


> [!NOTE] vec4 in light
> Specify light as a vector4. If it is position vector, set *w* to 1, if it is direction vector, set *w* to 0. We do not want to translate directions.

### Point lights
