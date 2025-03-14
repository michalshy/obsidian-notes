## Define material struct
```
#version 330 core
struct Material {
	vec3 ambient;
	vec3 diffuse;
	vec3 specular;
	float shininess;
};

uniform Material material;
```
We create this struct in the fragment shader.
Simulating real world materials: http://devernay.free.fr/cours/opengl/materials.html

## Setting material
We have to change lighting now so it complies with the new material properties.
We have to access them with uniform created above.
```
void main()
{
	vec3 ambient = lightColor * material.ambient;
	
	vec3 norm = normalize(Normal);
	vec3 lightDir = normalize(lightPos - FragPos);
	float diff = max(dot(norm, lightDir), 0.0);
	vec3 diffuse = lightColor * (diff * material.diffuse);

	 vec3 viewDir = normalize(viewPos - FragPos);
	 vec3 reflectDir = reflect(-lightDir, norm);
	 float spec = pow(max(dot(viewDir, reflectDir), 0.0), material.shininess);
	 
	 vec3 result = ambient + diffuse + spec;
	 FragColor = vec4(result, 1.0);
}
```

## Accessing material
```
lightingShader.setVec3("material.ambient", 1.0f, 0.5f, 0.31f);
...
```
No difference, struct just acts as an alias inside shader.

## Light properties
Light sources have different intensities as well. Default, they are set on max (1.0). In previous lessons, we fixed that by adding strength to ambient, diffuse and specular. We can do similar thing, but with a component as well as Material.

```
struct Light {
	vec3 position;
	
	vec3 ambient;
	vec3 diffuse;
	vec3 specular;
}

uniform Light light;
```
Now, lets update fragment shader
```
vec3 ambient = light.ambient * material.ambient;
vec3 diffuse = light.diffuse * (diff * material.diffuse);
vec3 specular = light.specular * (spec * material.specular);
```
And set them in app
```
lightingShader.setVec3("light.ambient", 0.2f, 0.2f, 0.2f); 
lightingShader.setVec3("light.diffuse", 0.5f, 0.5f, 0.5f); // darken diffuse light a bit
lightingShader.setVec3("light.specular", 1.0f, 1.0f, 1.0f);
```