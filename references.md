
```cpp
std::vector<GLint> SharedUniformBlock::setUniformBlockForShader(GLuint shaderProgram, std::string blockName, std::vector<std::string> blockMembers)
{
	// Determine the size of the block and set the binding point for the block(s)
	determineBlockSizeSetBindingPoint(shaderProgram, blockName);

	// Has the buffer been created and have the byte offset been found?
	if (blockSizeAndOffetsSet == false ) {

		// Set up the buffers and bind to binding points
		allocateBuffer(shaderProgram);

		// Find the byte offsets of the uniform block variables
		findOffsets(shaderProgram, blockMembers);

		// Indicate the buffer has already been allocated and the offsets have been determined
		blockSizeAndOffetsSet = true;
	}

	return offsets;

} // end setUniformBlockForShader
```