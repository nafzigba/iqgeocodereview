As part of this, we ask you to prepare a 15 minute technical presentation of a project you've worked on. Please prepare something that you are comfortable presenting and discussing with the interviewers, specifically focussing on the logic and  direction behind your decisions. This could be from a previous job, a personal project, or any example that you feel best demonstrates your skills. This will be followed by Q&A. 

## IQGeo Code Review Presentation 
<h2><a href="https://github.com/nafzigba/GameEngine_cpp/tree/master">Original Project Git Repo<a></h2>

### Bradley Nafziger
### 3/3/25

---

## Introduction
This presentation covers creating a custom shader pipeline for a 3D rendering engine using GLSL. It will go over implementation within an existing C++ codebase, benefits, common issues and debugging, and features. 

---

## Code Overview
GLSL is a shader pipeline technology that comes built in with OpenGL. OpenGL is required for this to work as it creates binding points in the memory and allocates space for the shaders. It has default implementations of each step of the pipeline and it is not necessary to create custom shaders. You can manage coloring, surface normals, and lighting within the rendering engine portion of your code, however this will run on the CPU and have poor performance; whereas shaders run calculations on the GPU boosting performance. 
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
void main() {
    gl_Position = projectionMatrix * viewMatrix * modelMatrix * vec4(aPosition, 1.0);
}
```
// Targeting version 4.5 of GLSL. If the compiler does not support 4.5 it will cause an error.
#version 450 core

layout(shared) uniform transformBlock
{
	mat4 modelMatrix;
	mat4 viewMatrix;
	mat4 projectionMatrix;
	mat4 normalModelMatrix;
};

out vec3 worldPos;
out vec3 worldNorm;
out vec2 texCoord0;
out mat3 TBN;

layout (location = 0) in vec4 vertexPosition;
layout (location = 1) in vec3 normal;
layout (location = 2) in vec2 vertexTexCoord;
layout(location = 3) in vec3 aTangent;
layout(location = 4) in vec3 aBitangent;

void main()
{
	// Normal Mapping
	vec3 T = normalize(vec3(modelMatrix * vec4(aTangent, 0.0)));
	vec3 B = normalize(vec3(modelMatrix * vec4(aBitangent, 0.0)));
	vec3 N = normalize(vec3(modelMatrix * vec4(normal, 0.0)));

	TBN = (mat3(T, B, N));

	// Transform the position of the vertex to clip 
	// coordinates (minus perspective division)
	gl_Position = projectionMatrix * viewMatrix * modelMatrix * vertexPosition;

	// Transform the position of the vertex to world 
	// coords for lighting
	worldPos = (modelMatrix * vertexPosition).xyz;

	// Transform the normal to world coords for lighting
	worldNorm = normalize(mat3(normalModelMatrix) * normal); 
	//worldNorm = normalize(normalModelMatrix * normal);
	
	// Pass through the texture coordinate
	texCoord0 = vertexTexCoord;

}

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
	vec4 ambientColor;		// ambient color of the light
	vec4 diffuseColor;		// diffuse color of the light
	vec4 specularColor;		// specular color of the light

	// Either the position or direction
	// if w = 0 then the light is directional and direction specified 
	// is the negative of the direction the light is shinning
	// if w = 1 then the light is positional
	vec4 positionOrDirection;

	// Spotlight attributes
	vec3 spotDirection;		// the direction the cone of light is shinning    
	bool isSpot;				// 1 if the light is a spotlight  
	float spotCutoffCos;	// Cosine of the spot cutoff angle
	float spotExponent;		// For gradual falloff near cone edge

	// Attenuation coefficients
	float constant;
	float linear;
	float quadratic;

	bool enabled;			// true if light is "on"

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
	vec4 ambientMat;
	vec4 diffuseMat;
	vec4 specularMat;
	vec4 emmissiveMat;
	float specularExp;
	int textureMode;
	bool diffuseTextureEnabled;
	bool specularTextureEnabled;
	bool normalMapTextureEnabled;
};

layout(shared) uniform MaterialBlock
{
	Material object;
};

//layout(shared) uniform FogBlock
//{
//	vec4 fogColor;
//	float fogEnd;
//	float fogStart;
//	float fogDensity;
//	unsigned int fogMode; // 0 no fog, 1 linear, 2 exponential, 3 exponential two
//}

layout(binding = 0) uniform sampler2D diffuseSampler;
layout(binding = 1) uniform sampler2D specularSampler;
layout(binding = 2) uniform sampler2D normalMapSampler;

void main()
{
	vec4 totalColor = object.emmissiveMat;

	vec4 ambientColor = object.ambientMat;

	vec4 diffuseColor = object.diffuseMat;

	vec4 specularColor = object.specularMat;

	vec3 fragWorldNormal = normalize(worldNorm);

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

				if(lights[i].positionOrDirection.w < 0.98f) { // Directional
					
					// Normalize the light vector (points towards a light source that is an 
					// "infinite" distance away and has no potition
					lightVector = normalize(lights[i].positionOrDirection.xyz);

					// No attenuation for directional lights
					attenuation = 1.0f;
				}
				else { // Positional
					
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

					// Blinn - Phong
					//vec3 halfVector = normalize(lightVector + viewVector);
					//totalColor += attenuation * fallOffFactor * pow(max(dot(fragWorldNormal, halfVector),0.0f),object.specularExp) * specularColor * lights[i].specularColor;
				}
			}
		}
	}
	else {

		totalColor = diffuseColor;
	}
	fragmentColor = totalColor;

	// Gamma Correction 
	//fragmentColor.rgb = pow(fragmentColor.rgb, vec3(1.0f/ gamma));

}

---

## Explanation

Break down the code into key components and explain each part in detail.

### Step 1: Vertex Attributes
- `aPosition`: Input vertex position in object space.
- `vec4(aPosition, 1.0)`: Converts a `vec3` to a `vec4` for matrix multiplication.

### Step 2: Transformations
- `modelMatrix`: Converts object space to world space.
- `viewMatrix`: Converts world space to view space.
- `projectionMatrix`: Converts view space to clip space.
- Final multiplication order ensures proper transformation.

---

## Output and Visualization

If applicable, describe the expected output. Include screenshots, diagrams, or links to a live demo.

Example:
- The transformed vertex positions define a 3D object in screen space.
- When rendered, the object appears properly positioned and oriented.

---

## Common Errors and Debugging

List common issues that might arise and how to fix them.

Example:
- **Problem:** Objects appear distorted → **Solution:** Check the projection matrix.
- **Problem:** Model not visible → **Solution:** Ensure proper camera setup.

---

## Conclusion

Summarize the key takeaways. Optionally, suggest further readings or improvements.

> In this presentation, we covered GLSL transformations and how to convert vertex positions from object space to clip space. Understanding this pipeline is essential for rendering 3D graphics.

---

## References

Include any links or resources you used for the presentation.

- [[OpenGL Transformation Pipeline](https://www.khronos.org/opengl/wiki/Vertex_Post-Processing)](https://www.khronos.org/opengl/wiki/Vertex_Post-Processing)
- [[GLSL Official Documentation](https://www.khronos.org/opengl/wiki/Core_Language_(GLSL))](https://www.khronos.org/opengl/wiki/Core_Language_(GLSL))

---

## Q&A

Encourage the audience to ask questions or provide feedback!

---