OpenGL is not familiar with camera, we can simulate one by moving all objects, giving the **ilusion**.
### Camera / View space
View matrix transforms all the world coordinates into view coordinates.
To define camera we need: 
- position in the world space
- direction it is looking at
- vector pointing to the right
- vector pointing up
#### 1. Camera position
vector in a world space
```
glm::vec3 cameraPos = glm::vec3(0.f, 0.f, 3.f);
```
**Positive z axis is going to the user direction (from the screen), we want to move it backwards, add to Z**
#### 2. Camera direction
Assume pointing to 0,0,0 in the world space.
Subtracting two vectors produces direction.
Subtracting camera position from origin makes this direction.

We want view matrix coordinate system to have positive Z, by default camera points toward negative Z, so we have to negate direction vector -> that is why we are switching subtraction

```
glm::vec3 cameraTarget = glm::vec3(0.f,0.f,0.f);
glm::vec3 cameraDirection = glm::normalize(cameraPos - cameraTarget);
```
> The name _direction_ vector is not the best chosen name, since it is actually pointing in the reverse direction of what it is targeting.

#### 3. Right axis
Provide up vector in the world space and make cross product with camera direction.
```
glm::vec3 up = glm::vec3(0.f, 1.f, 0.f);
glm::vec3 cameraRight = glm::normalize(glm::cross(up, cameraDirection));
```
#### 4. Up axis
`glm::vec3 cameraUp = glm::cross(cameraDirection, cameraRight);`

#### 5. LookAt Matrix
Used to transform any world coords into view coords.

$$
LookAt = 
\begin{bmatrix}
Rx & Ry & Rz & 0 \\
Ux & Uy & Uz & 0\\
Dx & Dy & Dz & 0\\
0 & 0 & 0  & 1\\
\end{bmatrix}
\cdot
\begin{bmatrix}
1 & 0 & 0 & -Px \\
0 & 1 & 0 & -Pu\\
0 & 0 & 1 & -Pz\\
0 & 0 & 0  & 1\\
\end{bmatrix}
$$

R is right vector, U is up, D is direction and P is the cameras position vector.
Rotation (left) and translation (right) matrices parts are inverted.
We want to rotate and translate the world in the opposite direction. 

**Creates view matrix that *looks* at given target.**

GLM will create it for us:
```
glm::mat4 view;
view = glm::lookAt(glm::vec3(0.f, 0.f, 3.f),
					glm::vec3(0.f, 0.f, 0.f),
					glm::vec3(0.f, 1.f, 0.f));
```

The glm::LookAt function requires a position, target and up vector respectively. This example creates a view matrix that is the same as the one we created in the previous chapter.

## Exercise 1 - Rotating camera at one point (0,0,0)

```
const float radius = 10.0f;
float camX = sin(glfwGetTime()) * radius;
float camZ = cos(glfwGetTime()) * radius;
glm::mat4 view;
view = glm::lookAt(glm::vec3(camX, 0.f, camZ), glm::vec3(0.f, 0.f, 0.f), glm::vec3(0.f, 1.f, 0.f));
```

#### 6. Walk around

Setup camera system
```
glm::vec3 cameraPos = glm::vec3(0.f, 0.f, 3.f);
glm::vec3 cameraFront = glm::vec3(0.f, 0.f, -1.f);
glm::vec3 cameraUp = glm::vec3(0.f, 1.f, 3.f);

view = glm::lookAt(cameraPos, cameraPos + cameraFront, cameraUp);
```
Additional input check
```
if (glfwGetKey(window, GLFW_KEY_W) == GLFW_PRESS)
	cameraPos += cameraSpeed * cameraFront; 
if (glfwGetKey(window, GLFW_KEY_S) == GLFW_PRESS) 
	cameraPos -= cameraSpeed * cameraFront; 
if (glfwGetKey(window, GLFW_KEY_A) == GLFW_PRESS) 
	cameraPos -= glm::normalize(glm::cross(cameraFront, cameraUp)) * cameraSpeed; if (glfwGetKey(window, GLFW_KEY_D) == GLFW_PRESS) 
	cameraPos += glm::normalize(glm::cross(cameraFront, cameraUp)) * cameraSpeed;
```

> Note that we normalize the resulting _right_ vector. If we wouldn't normalize this vector, the resulting cross product may return differently sized vectors based on the cameraFront variable. If we would not normalize the vector we would move slow or fast based on the camera's orientation instead of at a consistent movement speed.

#### 7. Delta time
```
float deltaTime = 0.0f; // Time between current frame and last frame 
float lastFrame = 0.0f; // Time of last frame
```
Each frame calculation
```
float currentFrame = glfwGetTime();
deltaTime = currentFrame - lastFrame;
lastFrame = currentFrame;
```
At the beginning of input processing provide
```
float cameraSpeed = 2.5f * deltaTime;
```
#### 8. Look around!

**EULER ANGLES:**
**pitch, yaw and roll**

![[Pasted image 20250306222629.png]]

pitch - how much looking up or down
yaw - magnitude looking left or right
roll - how much we roll

For camera **lets use yaw and pitch, ommit roll**

![[Pasted image 20250306223231.png]]
**assume hypotenuse = 1**, look down from Y axis

```
glm::vec3 direction;
direction.x = cos(glm::radians(yaw));
direction.z = sin(glm::radians(yaw));
```
![[Pasted image 20250306223639.png]]

```
direction.y = sin(glm::radians(pitch));
```
We have to include cos(pitch), it influences result:
```
direction.x = cos(glm::radians(yaw)) * cos(glm::radians(pitch)); 
direction.y = sin(glm::radians(pitch)); 
direction.z = sin(glm::radians(yaw)) * cos(glm::radians(pitch));
```
To make sure the camera points towards the negative z-axis by default we can give the `yaw` a default value of a 90 degree clockwise rotation. Positive degrees rotate counter-clockwise so we set the default `yaw` value to:
```
yaw = -90.0f;
```

#### 8. Mouse Input

Capture mouse
```
glfwSetInputMode(window, GLFW_CURSOR, GLFW_CURSOR_DISABLED);
```
We need to create mouse callback
```
void mouse_callback(GLFWwindow* window, double xpos, double ypos);

glfwSetCursorPosCallback(window, mouse_callback);
```
Here are some steps before being able to calculate camera's dir vector:
1. Calculate mouse offset since last frame
2. Add the offset values to the camera's yaw and pitch
3. Add some constraints to minimum/maximum pitch values
4. Calculate the direction vector

Initialize mousePos
`float lastX = 400, lastY = 300;`
then calculate offset
```
float xoffset = xpos - lastX;
float yoffset = ypos - lastY;

lastX = xpos;
lastY = ypos;

const float sens = 0.1f;
xoffset *= sens;
yoffset *= sens;

yaw += xoffset; 
pitch += yoffset;
```
Add constraints:
```
if(pitch > 89.0f) pitch = 89.0f; 
if(pitch < -89.0f) pitch = -89.0f;
```
Calculate direction
```
...
cameraFront = glm::normalize(direction);
```
Ommit first big jump of mouse
```
if (firstMouse) // initially set to true 
{ 
	lastX = xpos; 
	lastY = ypos; 
	firstMouse = false; 
}
```

#### Quick zoom
```
void scroll_callback(GLFWwindow* window, double xoffset, double yoffset) 
{
	fov -= (float)yoffset; 
	if (fov < 1.0f) 
		fov = 1.0f; 
	if (fov > 45.0f) 
		fov = 45.0f; 
}
```
`projection = glm::perspective(glm::radians(fov), 800.0f / 600.0f, 0.1f, 100.0f);`
`glfwSetScrollCallback(window, scroll_callback);`


# CHAPTER FINISHED