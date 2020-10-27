![Hybrid Renderer](../Images/HybridRenderer.gif)

## Hybrid Renderer

A 3D rasterized renderer that allows you to switch between DirectX 11, and software rendering.
- Software and DirectX rendering modes.
- ImGui debug UI and logger.
- Color and depth view in software mode.
- Transparency in hardware mode

## Implementation

2 Seperate Render Pipelines

- **Software**

```cpp
// This is the surface we'll be rendering to, we need to lock it to write to it
SDL_LockSurface(m_pSoftwareBuffer);

// Clear OpenGL and software buffers
const auto clearColor = RGBColor(128.f, 128.f, 128.f);
glClearColor(clearColor.r / 255.f, clearColor.g / 255.f, clearColor.b / 255.f, 1.0f);
glClear(GL_COLOR_BUFFER_BIT);

std::fill_n(m_pDepthBuffer, m_Width * m_Height, std::numeric_limits<float>::infinity());
std::fill_n(static_cast<uint32_t*>(m_pSoftwareBufferPixels), m_Width * m_Height,
	SDL_MapRGB(m_pSoftwareBuffer->format,
    static_cast<uint8_t>(clearColor.r),
    static_cast<uint8_t>(clearColor.g),
    static_cast<uint8_t>(clearColor.b)));

// Prepare new ImGui frame
ImGui_ImplOpenGL2_NewFrame();
ImGui_ImplSDL2_NewFrame(m_pWindow);
ImGui::NewFrame();

// Render
for (auto pObject : m_pSceneGraph->GetCurrentSceneObjects())
{
	m_pSceneGraph->GetCamera()->MakeScreenSpace(pObject);
	pObject->Rasterize(m_pSoftwareBuffer, m_pSoftwareBufferPixels, m_pDepthBuffer, m_Width, m_Height);
}

// We're done writing to the surface, so we can unlock it
SDL_UnlockSurface(m_pSoftwareBuffer);

// Render software render as background image to allow ImGui overlay
ImplementSoftwareWithOpenGL();

Logger::GetInstance()->OutputLog();
SceneGraph::GetInstance()->RenderDebugUI();

// Present ImGui data before final OpenGL render
ImGui::Render();
ImGui_ImplOpenGL2_RenderDrawData(ImGui::GetDrawData());

// Swap the OpenGL buffers for final output
SDL_GL_SwapWindow(m_pWindow);
```

- **DirectX**
<br>

```cpp
// Confirm DirectX pipeline is initialized
if (!m_IsInitialized)
	return;

// Clear Buffers
const auto clearColor = RGBColor(0.f, 0.f, 0.3f);
m_pDeviceContext->ClearRenderTargetView(m_pRenderTargetView, &clearColor.r);
m_pDeviceContext->ClearDepthStencilView(m_pDepthStencilView, D3D11_CLEAR_DEPTH | D3D11_CLEAR_STENCIL, 1.0f, 0);

// Prepare new ImGui frame
ImGui_ImplDX11_NewFrame();
ImGui_ImplSDL2_NewFrame(m_pWindow);
ImGui::NewFrame();

//Render
for (auto& mesh : m_pSceneGraph->GetCurrentSceneObjects())
{
	mesh->Render(m_pDeviceContext, m_pSceneGraph->GetCamera());
}

Logger::GetInstance()->OutputLog();
SceneGraph::GetInstance()->RenderDebugUI();

// Present ImGui data before final DX render
ImGui::Render();	
ImGui_ImplDX11_RenderDrawData(ImGui::GetDrawData());

//Present DirectX render
m_pSwapChain->Present(0, 0);
break;
```
<br>

To allow Dear ImGui to work in software mode, you may notice that OpenGL is used to present the final render to the user.
The software image is rendered to an `SDL_Surface`, which can be used to create an OpenGL texture as follows
<br>
```cpp
void Renderer::ImplementSoftwareWithOpenGL() const noexcept
{
	// Generate and bind a texture resource from OpenGL
	GLuint texture;
	glGenTextures(1, &texture);
	glBindTexture(GL_TEXTURE_2D, texture);

	// Make a Texture2D from the software buffer we rendered
	glTexImage2D(GL_TEXTURE_2D, 0, GL_RGBA, m_pSoftwareBuffer->w, m_pSoftwareBuffer->h, 0,
		GL_BGRA,GL_UNSIGNED_BYTE, m_pSoftwareBufferPixels );

	// Mipmap filtering. Has to be set for texture to render
	glTexParameteri(GL_TEXTURE_2D, GL_TEXTURE_MIN_FILTER, GL_NEAREST);
	glTexParameteri(GL_TEXTURE_2D, GL_TEXTURE_MAG_FILTER, GL_NEAREST);

	// Draw fullscreen quad with software renderer output as texture
	glEnable(GL_TEXTURE_2D);
	{
		glBegin(GL_QUADS);
		{
			const auto uvLeft = 0.0f;	const auto vLeft = -1.f;
			const auto uvRight = 1.0f;	const auto vRight = 1.f;
			const auto uvTop = 0.0f;	const auto vTop = 1.f;
			const auto uvBottom = 1.0f;	const auto vBottom = -1.f;

			glTexCoord2f(uvLeft, uvBottom);		glVertex2f(vLeft, vBottom);
			glTexCoord2f(uvLeft, uvTop);	 	glVertex2f(vLeft, vTop);
			glTexCoord2f(uvRight,uvTop);	 	glVertex2f(vRight,vTop);
			glTexCoord2f(uvRight,uvBottom);		glVertex2f(vRight,vBottom);
		}
		glEnd();
	}
	glDisable(GL_TEXTURE_2D);
	
	// Don't forget to delete the texture you've just created, since this is happening every frame! (unless you want to make a ticking memory leak time-bomb, I guess)
	glDeleteTextures(1, &texture);

	// Unbinding the texture for good measure
	glBindTexture(GL_TEXTURE_2D, 0);
}
```

Some of the libraries used for this project:
- [glm](https://glm.g-truc.net/0.9.9/index.html) (math library)
- [Dear ImGui](https://github.com/ocornut/imgui) (UI)
- [SDL2](https://www.libsdl.org/) (program window)
- [SDL_image](https://www.libsdl.org/projects/SDL_image/) (loading textures)
- [magic_enum](https://github.com/Neargye/magic_enum) (pretty neat enum reflection)
- [vld](https://kinddragon.github.io/vld/) (memory leak detection)

[Project Repository](https://github.com/DatTestBench/HybridRenderer)

<br>

[<- Back](../index.md)