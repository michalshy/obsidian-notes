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
Point lights is a light source with a given position somewhere in a world that illuminates in all directions (e.g. torches, light bulbs), where light ray fade out over distance.
We want to add logic that handles fading out the light.
#### Attenuation
To reduce intensity of light over the distance a light ray travels is generally called *attenuation*. 
One way to reduce the light intensity over distance is to use linear equation. However, such a linear function tends to look fake.
In real life, lights are generally bright standing close by, but the brightness diminishes quickly at a distance; the remaining light intensity then slowly diminishes over distance. Other equation should be used then.
$$
\begin{equation} F_{att} = \frac{1.0}{K_c + K_l * d + K_q * d^2} \end{equation}
$$
*d* represents distance from fragment to light source.
Then to calculate, we specify 3 configurable terms:
- a constant term *K<sub>c</sub>*
- a linear term *K<sub>l</sub>*
- a quadratic term *K<sub>q</sub>*
The constant is usually kept at 1.0 which is mainly there to make sure denominator does not go lower than 1.0.
The linear is multiplied with distance, which reduces the intensity in linear fashion.
The quadratic is multiplied with quadrant of distance. Quadratic will be less significant when distance is low, but will get more important when distance grows.
![[Pasted image 20250313205007.png]]
#### Choosing the right values
![[Pasted image 20250313205120.png]]
Example parameters for distance of light to cover.
#### Implementing attenuation
To calculate point light, we need 3 extra parameters and object position, since it is important now.
```
struct Light {
	vec3 postion;
	vec3 ambient;
	vec3 diffuse;
	vec3 specular;
	float constant;
	float linear;
	float quadratic;
}
```
Then, terms have to be set in the application.
```
lightingShader.setFloat("light.constant", 1.0f); 
lightingShader.setFloat("light.linear", 0.09f);
lightingShader.setFloat("light.quadratic", 0.032f);
```
Now we calculate with usage of formula
```
float distance = length(light.position - FragPos);
float attenuation = 1.0 / (light.constant + light.linear * distance + light.quadratic * (distance * distance));
```
Then, we can include it in all 3 of Phong model components.
```
ambient *= attenuation; 
diffuse *= attenuation; 
specular *= attenuation;
```
We could leave the ambient component alone so ambient lighting is not decreased over distance, but if we were to use more than 1 light source all the ambient components will start to stack up. In that case we want to attenuate ambient lighting as well. Simply play around with what's best for your environment.
### Spotlight
Light which is located somewhere in the environment and instead of shooting light rays in all directions, only shoots them in a specific direction.
A good example of a spotlight is street lamp or a flashlight.
![[Pasted image 20250313220220.png]]
A spotlight is represented by a world-space position, a direction and a cutoff angle that specifies the radius of the spotlight.
- LightDir: vector from the fragment to the light source
- SpotDir: direction the spotlight aiming at
- Phi ϕ: the cutoff angle
- Theta θ: angle between LightDir and SpotDir. Should be smaller than Phi to be inside.

To implement spotlight, calculate dot product between LightDir and SpotDir vectors. Compare them with cutoff angle.
#### Flashlight
Spotlight located at the viewer's position and usually aimed straight ahead from player's perspective. Normal spotlight, but with position and direction continually updated based on the player's position and orientation.
Start from updating Light struct
```
struct Light {
	vec3 position;
	vec3 direction;
	float cutOff;
	...
}
```
Next, pass values to the shader
```
lightingShader.setVec3("light.position", camera.Position); 
lightingShader.setVec3("light.direction", camera.Front);
lightingShader.setFloat("light.cutOff", glm::cos(glm::radians(12.5f)));
```
As you can see we're not setting an angle for the cutoff value but calculate the cosine value based on an angle and pass the cosine result to the fragment shader. The reason for this is that in the fragment shader we're calculating the dot product between the `LightDir` and the `SpotDir` vector and the dot product returns a cosine value and not an angle; and we can't directly compare an angle with a cosine value. To get the angle in the shader we then have to calculate the inverse cosine of the dot product's result which is an expensive operation. So to save some performance we calculate the cosine value of a given cutoff angle beforehand and pass this result to the fragment shader. Since both angles are now represented as cosines, we can directly compare between them without expensive operations.
Now, calculate theta and compare with cutoff value.
```
float theta = dot(lightDir, normalize(-light.direction));
if(theta > light.cutOff)
{
	//do light calculations
}
else
	color = vec4(light.ambient * vec3(texture(material.diffuse, TexCoords)), 1.0);
```

#### Smooth/Soft edges
To create effect of smooth edges, spotlight has to have inner and outer cone.
To create outer cone we simply define another cosine value which represents the angle between SpotDir and the outer cone's vector.
If a fragment is between inner and outer cone it should calculate an intensity value between 0.0 and 1.0. If fragment is in inner cone, intensity should be 1.0, if it outside of outer cone, it should be 0.0
We can calculate such a value with following equation:
$$
\begin{equation} I = \frac{\theta - \gamma}{\epsilon} \end{equation}
$$
Epsilon is the cosine difference between  the inner (ϕ) and the outer cone (γ) (ϵ=ϕ−γ). Resulting value is intensity of the spotlight at the current fragment.
![[Pasted image 20250313224617.png]]
We do not need if statement, since clamped value will be between 0.0 and 1.0.
```
float theta = dot(lightDir, normalize(-light.direction)); 
float epsilon = light.cutOff - light.outerCutOff; 
float intensity = clamp((theta - light.outerCutOff) / epsilon, 0.0, 1.0);
... 
// we'll leave ambient unaffected so we always have a little light. 
diffuse *= intensity; 
specular *= intensity; 
...
```

