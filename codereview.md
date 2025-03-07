As part of this, we ask you to prepare a 15 minute technical presentation of a project you've worked on. Please prepare something that you are comfortable presenting and discussing with the interviewers, specifically focussing on the logic and  direction behind your decisions. This could be from a previous job, a personal project, or any example that you feel best demonstrates your skills. This will be followed by Q&A. 

## IQGeo Code Review Presentation 
<h2><a href="https://github.com/nafzigba/GameEngine_cpp/tree/master">Original Project Git Repo<a></h2>

### Bradley Nafziger
### 3/3/25

---

## Introduction
This presentation covers creating a custom shader pipeline for a 3D rendering engine using GLSL. It will go over implementation within an existing C++ codebase, benefits, common issues and debugging, and features. 

---

## Technology Overview
GLSL is a shader pipeline technology that comes built in with OpenGL. OpenGL is required for this to work as it creates binding points in the memory and allocates space for the shaders. It has default implementations of each step of the pipeline and it is not necessary to create custom shaders. You can manage coloring, surface normals, and lighting within the rendering engine portion of your code, however this will run on the CPU and have poor performance; whereas shaders run calculations on the GPU boosting performance. GPUs are specialized in performing geometric calculations and parallized tasks.
The three sections of this presentation will be Buffer Writing, Layouts, and Calculations.

---

## Buffer Writing
Warning this is C++ code, it is graphic and if you have a weak stomach I recommend looking away for this part. 
This is a sample of a layout connector that inputs different view model matrices into the GLSL pipeline. This can be eventually generalized once there are many layout blocks that you want to insert data into, for example if you want also material blocks, lighting, fog, etc...

<a href="https://www.khronos.org/opengl/wiki/OpenGL_Reference">OpenGL Functions Definitions</a> 
- <a href="https://www.khronos.org/opengl/wiki/OpenGL_Type">GLuint</a>:  unsigned binary integer, no need for memory signature to be negative
- <a href="https://www.khronos.org/opengl/wiki/GLAPI/glBindBuffer">glBindBuffer</a>: <a href="references.md">glBindBuffer(GLenum target​, GLuint buffer​);, glenum target in this instance the layout uniform block found in the glsl code. GLuint buffer is the id for the buffer to connect </a>
- <a href="https://www.khronos.org/opengl/wiki/OpenGL_Type">GLuint</a>:  unsigned binary integer, no need for memory signature to be negative

---

Here is the data we want to send into the pipeline, as you can see theres a projection matrix, view matrix and model matrix. 
```cpp
#include "SharedTransformations.h"
GLuint SharedTransformations::projectionLocation; // Byte offset of the projection matrix
glm::mat4 SharedTransformations::projectionMatrix; // Current projection matrix that is held in the buffer
GLuint SharedTransformations::viewLocation;
glm::mat4 SharedTransformations::viewMatrix;
GLuint SharedTransformations::modelLocation;
glm::mat4 SharedTransformations::modelMatrix;
GLuint SharedTransformations::normalModelLocation;
GLuint SharedTransformations::eyePositionLocation;
SharedUniformBlock SharedTransformations::projViewBlock(projectionViewBlockBindingPoint);
SharedUniformBlock SharedTransformations::worldEyeBlock(worldEyeBlockBindingPoint);
const std::string SharedTransformations::transformBlockName = "transformBlock";
const std::string SharedTransformations::eyeBlockName = "worldEyeBlock";
```
This following piece takes the above data, labels it, and packs it into an array to eventually slot into the buffer that is allocated in memory. 
What transformation matricies do will be covered later in the document.


```cpp
void SharedTransformations::setUniformBlockForShader(GLuint shaderProgram)
{
	std::vector<std::string> projViewMemberNames = { "modelMatrix", "normalModelMatrix", "viewMatrix", "projectionMatrix"};
	std::vector<GLint> uniformOffsets = projViewBlock.setUniformBlockForShader(shaderProgram, transformBlockName, projViewMemberNames);
	// Save locations
	modelLocation = uniformOffsets[0];
	normalModelLocation = uniformOffsets[1];
	viewLocation = uniformOffsets[2];
	projectionLocation = uniformOffsets[3];
	uniformOffsets.clear();
	std::vector<std::string> worldEyeMemberNames = { "worldEyePosition" };
	uniformOffsets = worldEyeBlock.setUniformBlockForShader(shaderProgram, eyeBlockName, worldEyeMemberNames);
	// Save location
	eyePositionLocation = uniformOffsets[0];
} // end setUniformBlockForShader
```
Here the glBindBuffer function is the workhorse of the code. For the view matrix you need to consider the world and projection blocks, but for modeling and projection matrices they are the same. 
```cpp
void SharedTransformations::setViewMatrix( glm::mat4 viewMatrix)
{
	if (projViewBlock.getSize() > 0  ) {
		glBindBuffer(GL_UNIFORM_BUFFER, projViewBlock.getBuffer());
		SharedTransformations::viewMatrix = viewMatrix;
		glBufferSubData(GL_UNIFORM_BUFFER, viewLocation, sizeof(glm::mat4), glm::value_ptr(viewMatrix));
	}

	if (worldEyeBlock.getSize() > 0 ) {
		glBindBuffer(GL_UNIFORM_BUFFER, worldEyeBlock.getBuffer());
		glm::vec3 viewPoint = vec3(glm::inverse(viewMatrix)[3]);
		glBufferSubData(GL_UNIFORM_BUFFER, eyePositionLocation, sizeof(glm::vec3), glm::value_ptr(viewPoint));
	}
	// Unbind the buffer. 
	glBindBuffer(GL_UNIFORM_BUFFER, 0);
} // end setViewMatrix
```

Accessor functions, projection and modeling functions are the same.


```cpp
glm::mat4 SharedTransformations::getViewMatrix()
{return viewMatrix;}
```

## Layouts
The uniforms are where all of the data that we bound in the buffers can be accessed. These need to have the same names.  

``` glsl
#version 330 core
layout(location = 0) in vec3 aPosition;
uniform mat4 modelMatrix;
uniform mat4 viewMatrix;
uniform mat4 projectionMatrix;
```
<h1>Vertex Shader</h1>
"layouts" are the entrance point for the data sent from the engine (glBindBuffer) to be processed by the shader.

```glsl
#version 450 core
layout(shared) uniform transformBlock
{
	mat4 modelMatrix;
	mat4 viewMatrix;
	mat4 projectionMatrix;
	mat4 normalModelMatrix;
};
layout (location = 0) in vec4 vertexPosition;
layout (location = 1) in vec3 normal;
layout (location = 2) in vec2 vertexTexCoord;
layout(location = 3) in vec3 aTangent;
layout(location = 4) in vec3 aBitangent;
```

Example syntax for the output data that is sent to the next step of the pipeline

```glsl
out vec3 worldPos;
out vec3 worldNorm;
out vec2 texCoord0;
out mat3 TBN;
```

Start of computations for the verticies, this section is for normal mapping.
<img style="width:150" src="https://external-content.duckduckgo.com/iu/?u=http%3A%2F%2F1.bp.blogspot.com%2F-l-q0YPQkQSk%2FULI1oM5NEpI%2FAAAAAAAAAVg%2FFM4btN0XKLE%2Fs1600%2FWell%2BPreserved%2BChesterfield%2B-%2B(Normal%2BMap_1).png&f=1&nofb=1&ipt=5aa479b3533cf147f82917fb0166e735dfd8946eeb6e0a59ed29ee062bbcb950&ipo=images"></img>

```glsl
void main()
{
	// Normal Mapping
	vec3 T = normalize(vec3(modelMatrix * vec4(aTangent, 0.0)));
	vec3 B = normalize(vec3(modelMatrix * vec4(aBitangent, 0.0)));
	vec3 N = normalize(vec3(modelMatrix * vec4(normal, 0.0)));

	TBN = (mat3(T, B, N));
```

<img src="https://external-content.duckduckgo.com/iu/?u=https%3A%2F%2Fuser-images.githubusercontent.com%2F42164422%2F107613437-ab973700-6c8b-11eb-99a8-d08c99fc0d73.png&f=1&nofb=1&ipt=c5bb6888a1045901268c89128c78b4b7e1d0e9f48ab9d478a75898b55741f01c&ipo=images"></img>
In this section these are the transformations being bound to the shader program. These are all of the "out" variables we described above.
1. Transform the position of the vertex to clip coordinates (minus perspective division)
2. Transform the position of the vertex to world coords for lighting
3. Transform the normal to world coords for lighting
4. Pass through the texture coordinate

```glsl
	gl_Position = projectionMatrix * viewMatrix * modelMatrix * vertexPosition; //1
	worldPos = (modelMatrix * vertexPosition).xyz;//2
	worldNorm = normalize(mat3(normalModelMatrix) * normal);//3 
	texCoord0 = vertexTexCoord;//4

}
```
<h1>Fragment Shader</h1>
<h3>Layouts and Ins & Outs</h3>
The descriptor "in" describes data that flows into the shader program from a previous shader. The descriptor "out" describes data that flows out of the shader program into another. Notice that the main function that does calculations does not have a return function like you would expect from a "C"-like language, rather the final values at the end of the program are outputted from whatever value exists in the "out" variables.


``` glsl
// Targeting version 4.5 of GLSL. If the compiler does not support 4.5 it will cause an error.
#version 450 core
in vec3 worldPos;
in vec3 worldNorm;
in vec2 texCoord0;
in mat3 TBN;
out vec4 fragmentColor;
const float gamma = 2.2;
const int MaxLights = 16;

// Structure for holding general light properties
struct GeneralLight
{
	vec4 ambientColor, diffuseColor, specularColor, positionOrDirection;
	// Spotlight attributes
	vec3 spotDirection;		// the direction the cone of light is shinning    
	bool isSpot;				// 1 if the light is a spotlight  
	float spotCutoffCos;	// Cosine of the spot cutoff angle
	float spotExponent;		// For gradual falloff near cone edge
	// Attenuation coefficients
	float constant,linear,quadratic;
	bool enabled;// true if light is "on"

};
layout(shared) uniform LightBlock
{
	GeneralLight lights[MaxLights];
};
layout(shared) uniform worldEyeBlock
{
	vec3 worldEyePosition;
};
struct Material
{
	vec4 ambientMat, diffuseMat, specularMat, emmissiveMat;
	float specularExp;
	int textureMode;
	bool diffuseTextureEnabled, specularTextureEnabled, normalMapTextureEnabled;
};
layout(shared) uniform MaterialBlock
{
	Material object;
};
layout(binding = 0) uniform sampler2D diffuseSampler;
layout(binding = 1) uniform sampler2D specularSampler;
layout(binding = 2) uniform sampler2D normalMapSampler;
```
<h3>Computations</h3>

```glsl
void main()
{
	vec4 totalColor = object.emmissiveMat;
	vec4 ambientColor = object.ambientMat;
	vec4 diffuseColor = object.diffuseMat;
	vec4 specularColor = object.specularMat;
	vec3 fragWorldNormal = normalize(worldNorm);
```

Texture mapping: 

<img style="width:250" src="https://external-content.duckduckgo.com/iu/?u=https%3A%2F%2Fwww.infotech.edu.gr%2Fwp-content%2Fuploads%2F2018%2F11%2F3.3.jpg&f=1&nofb=1&ipt=fd40ae0c104063680438c674c31f37eecca79c7b82fbb6db639c9c7152b57bd9&ipo=images"></img>

```glsl
	if (object.normalMapTextureEnabled) {
		vec3 normal = texture(normalMapSampler, texCoord0).xyz;
		normal = normalize(normal * 2.0f - 1.0f);
		fragWorldNormal = normalize(TBN * normal);
	}
	if(object.textureMode != 0 && object.diffuseTextureEnabled) {
		diffuseColor = texture( diffuseSampler, texCoord0 );
		ambientColor = diffuseColor;
	 }
	 if(object.textureMode != 0 && object.specularTextureEnabled) {
		specularColor = texture( specularSampler, texCoord0 );
	 }
	 if(object.textureMode != 1) {
		 for(int i = 0; i < MaxLights ; i++) {
			 if(lights[i].enabled == true ) {
				vec3 lightVector;
				float attenuation;
				float spotCosine = 1.0;
				float fallOffFactor = 1.0f;
```

Directional: Normalize the light vector (points towards a light source that is an "infinite" distance away and has no position.
- No attenuation for directional lights

```glsl
				if(lights[i].positionOrDirection.w < 0.98f) { 
                    
					lightVector = normalize(lights[i].positionOrDirection.xyz);
					// No attenuation for directional lights
					attenuation = 1.0f;
				}
```

Positional:

```glsl
				else { 
                    // Positional
					// Calculate the light vector					
					lightVector = normalize(lights[i].positionOrDirection.xyz - worldPos);
					// Calculate the distance to the light source
					float distanceToLight = distance(lights[i].positionOrDirection.xyz, worldPos);
					// Calculate the attenuation to weakend the light with distance
					attenuation = 1.0f / (lights[i].constant + lights[i].linear *distanceToLight + lights[i].quadratic * distanceToLight * distanceToLight);
					if (lights[i].isSpot == true) {
						// Calculate the spot cosine used to determine if in the spotlight cone
						spotCosine = dot(-lightVector, normalize(lights[i].spotDirection));
						// Caclate the fallOffFactor used to "blur" the edges of the spotlight cone
						fallOffFactor = pow(spotCosine, lights[i].spotExponent);  
						//fallOffFactor = clamp( 1.0f - (1.0f - spotCosine) / (1.0f - lights[i].spotCutoffCos), 0.0f, 1.0f);
					}
				}
```

Spot light, same as positional lighting, except there is a cutoff angle that light doesnt shine outside of.

```glsl
				// Is it a spot light and are we in the cone?
				if ( lights[i].isSpot == false || (lights[i].isSpot == true && spotCosine >= lights[i].spotCutoffCos) ) {
					// Ambient reflection
					totalColor += attenuation * fallOffFactor * ambientColor * lights[i].ambientColor;
					// Diffuse reflection
					totalColor += attenuation * fallOffFactor * max(dot(fragWorldNormal, lightVector), 0.0f) * diffuseColor * lights[i].diffuseColor;
					// Specular reflection
					vec3 viewVector = normalize(worldEyePosition - worldPos);
					// Phong
					vec3 reflection = normalize(reflect(-lightVector, fragWorldNormal));
					totalColor += attenuation * fallOffFactor * pow(max(dot(reflection, viewVector),0.0f),object.specularExp) * specularColor * lights[i].specularColor;
				}
			}
		}
	}
	else {
		totalColor = diffuseColor;
	}
	fragmentColor = totalColor;
}
```

---

## Output and Visualization

<a src="https://www.youtube.com/watch?v=677e8DsGWVQ">Demo Video</a>

---

## Common Errors and Debugging

Objects appear black and do not have any texture or lighting data:

Final frag color does not have values properly set between 0 & 1 

Do not see anything in the scene:

Camera is pointing the wrong direction.

---
## Further Optimization

Having branching if statements, combined with for loops within this code reduces efficiency, as each if statment needs to be processed per pixel. A large improvement can be made by pre computing the light value of a pixel beforehand, as there are 16 lights sent into the shader and the for loop must process it 16 times if there are that many lights in the scene.




---

## Conclusion

> In this presentation, we covered GLSL transformations and how to convert vertex positions from object space to clip space. Understanding this pipeline is essential for rendering 3D graphics.

---

## References

Include any links or resources you used for the presentation.

- [[OpenGL Transformation Pipeline](https://www.khronos.org/opengl/wiki/Vertex_Post-Processing)](https://www.khronos.org/opengl/wiki/Vertex_Post-Processing)
- [[GLSL Official Documentation](https://www.khronos.org/opengl/wiki/Core_Language_(GLSL))](https://www.khronos.org/opengl/wiki/Core_Language_(GLSL))

---

## Q&A

Please add any comments on how this could better be presented or any tough points that require more explanation.

---