## Start
Previous chapters showed a lot about lighting in OpenGL.
Phong shading, materials, lighting maps and different type of light casters.
Here, we are going to combine all the knowledge by creating **fully lit scene with 6 active lights sources.**
To use more than one light source in the scene, lighting calculations will be encapsulated into *GLSL functions.*
Functions in GLSL are just like C-like functions.
When using multiple lights in a scene, approach is usually as follows:
- single color vector which represents the fragment's output color
- for each light, the light's contribution to the fragment is added to its output color vector
- each light in scene will calculate its individual impact and contribute that to the final output color
Structure of this workflow might look like this
```
out vec4 FragColor;
void main()
{
	//define an output color value
	vec3 output = vec3(0.0);
	//add the directional light
	output += someFunctionToCalculateDirectionalLight();
	//do the same for point lights
	for(int i = 0; i < nr_of_point_lights; i++)
	{
		output += someFunctionToCalculatePointLight();
	}
	//add other lights
	output += someFunctionToCalculateSpotLight();
	FragColor = vec4(output, 1.0);
}
```
## Directional light
First, struct has to be created
```
struct DirLight{
	vec3 direction;
	vec3 ambient;
	vec3 diffuse;
	vec3 specular;
};
```
then, we can pass dirLight uniform to the function with the following prototype:
`vec3 CalcDirLight(DirLight light, vec3 normal, vec3 viewDir);`
A word:
	Just like C and C++, when we want to call a function (in this case inside the main function) the function should be defined somewhere before the caller's line number. In this case we'd prefer to define the functions below the main function so this requirement doesn't hold. Therefore we declare the function's prototypes somewhere above the main function, just like we would in C.
Content of the function
```
vec3 CalcDirLight(DirLight light, vec3 normal, vec3 viewDir)
{
	vec3 lightDir = normalize(-light.direction);
	// diffuse shading
	float diff = max(dot(normal, lightDir), 0.0);
	// specular shading
	vec3 reflectDir = reflect(-lightDir, normal);
	float spec = pow(max(dot(viewDir, reflectDir), 0.0), material.shininess);
	//combine
	vec3 ambient = light.ambient * vec3(texture(material.diffuse, TexCoords));
	vec3 diffuse = light.diffuse * diff * vec3(texture(material.diffuse, TexCoords));
	vec3 specular = light.specular * spec * vec3(texture(material.specular, TexCoords));
	return (ambient+diffuse+specular);
}
```
## Point Light
First, struct has to be defined:
```
struct PointLight 
{ 
	vec3 position; 
	float constant; 
	float linear; 
	float quadratic; 
	vec3 ambient; 
	vec3 diffuse; 
	vec3 specular; 
}; 
#define NR_POINT_LIGHTS 4 
uniform PointLight pointLights[NR_POINT_LIGHTS];
```
Prototype of the function to calculate
`vec3 CalcPointLight(PointLight light, vec3 normal, vec3 fragPos, vec3 viewDir);`
Function takes all the data needed and returns vec3 that represents color contribution.
```
vec3 CalcPointLight(PointLight light, vec3 normal, vec3 fragPos, vec3 viewDir) {
	vec3 lightDir = normalize(light.position - fragPos);
	// diffuse shading 
	float diff = max(dot(normal, lightDir), 0.0); 
	// specular shading 
	vec3 reflectDir = reflect(-lightDir, normal); 
	float spec = pow(max(dot(viewDir, reflectDir), 0.0), material.shininess); 
	// attenuation 
	float distance = length(light.position - fragPos); 
	float attenuation = 1.0 / (light.constant + light.linear * distance + light.quadratic * (distance * distance)); 
	// combine results 
	vec3 ambient = light.ambient * vec3(texture(material.diffuse, TexCoords)); 
	vec3 diffuse = light.diffuse * diff * vec3(texture(material.diffuse, TexCoords)); 
	vec3 specular = light.specular * spec * vec3(texture(material.specular, TexCoords)); 
	ambient *= attenuation; 
	diffuse *= attenuation; 
	specular *= attenuation; 
	return (ambient + diffuse + specular);
}
```
## Put it together
Now, we can put those lights together in main
```
void main()
{
	//properties
	vec3 norm = normalize(Normal);
	vec3 viewDir = normalize(viewPos - FragPos);
	//directional
	vec3 result = CalcDirLight(dirLight, norm, viewDir);
	//point lights
	for(int i = 0; i < NR_POINT_LIGHTS; i++)
	{
		result += CalcPointLight(pointLights[i], norm, FragPos, viewDir);
	}
	//spot light
	//result += CalcSpotLight(spotLihgt, norm, FragPos, viewDir);
	FragColor = vec4(result, 1.0);
}
```
#### Positions of point lights
```
glm::vec3 pointLightPositions[] = { 
	glm::vec3( 0.7f, 0.2f, 2.0f), 
	glm::vec3( 2.3f, -3.3f, -4.0f), 
	glm::vec3(-4.0f, 2.0f, -12.0f), 
	glm::vec3( 0.0f, 0.0f, -3.0f) 
};
```