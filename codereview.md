ChatGPT MarkDown Template

# Code Presentation Template

## Title: IQGeo Code Review Presentation 

### Author: Bradley Nafziger
### Date: 3/3/25

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
Here the glBindBuffer function is the workhorse of the code. For the view matrix you need to consider the world and projection blocks, but for modeling and projection matrices there the same. 
```cpp
void SharedTransformations::setViewMatrix( glm::mat4 viewMatrix)
{
	if (projViewBlock.getSize() > 0  ) {
		// Bind the buffer. 
		glBindBuffer(GL_UNIFORM_BUFFER, projViewBlock.getBuffer());
		SharedTransformations::viewMatrix = viewMatrix;
		glBufferSubData(GL_UNIFORM_BUFFER, viewLocation, sizeof(glm::mat4), glm::value_ptr(viewMatrix));
	}

	if (worldEyeBlock.getSize() > 0 ) {
		// Bind the buffer.
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