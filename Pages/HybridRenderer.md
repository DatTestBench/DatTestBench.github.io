![Hybrid Renderer](../Images/HybridRenderer.gif)

## Hybrid Renderer

A 3D rasterized renderer that allows you to switch between DirectX 11, and software rendering.
- Software and DirectX rendering modes.
- ImGui debug UI and logger



## Implementation

2 Seperate Render Pipelines

- **Software**
```cpp
const auto clearColor = RGBColor(128.f, 128.f, 128.f);

// Clear OpenGL and software buffers
glClearColor(clearColor.r, clearColor.g, clearColor.b, 1.0f);
glClear(GL_COLOR_BUFFER_BIT | GL_DEPTH_BUFFER_BIT);
SDL_LockSurface(m_pSoftwareBuffer);

std::fill_n(m_pDepthBuffer, static_cast<uint64_t>(m_Width * m_Height), FLT_MAX);
std::fill_n(static_cast<uint32_t*>(m_pSoftwareBufferPixels), static_cast<uint64_t>(m_Width * m_Height), SDL_MapRGB(m_pSoftwareBuffer->format,
    static_cast<uint8_t>(clearColor.r),
    static_cast<uint8_t>(clearColor.g),
    static_cast<uint8_t>(clearColor.b)));

// Prepare new ImGui frame
ImGui_ImplOpenGL2_NewFrame();
ImGui_ImplSDL2_NewFrame(m_pWindow);
ImGui::NewFrame();

// Render
for (auto& o : m_pSceneGraph->GetCurrentSceneObjects())
{
	m_pSceneGraph->GetCamera()->MakeScreenSpace(o);
	o->Rasterize(m_pSoftwareBuffer, static_cast<uint32_t*>(m_pSoftwareBuffer->pixels), m_pDepthBuffer, m_Width, m_Height);
}
SDL_LockSurface(m_pSoftwareBuffer);

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

//Present
m_pSwapChain->Present(0, 0);
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
glGenTextures(1, &texture );
glBindTexture(GL_TEXTURE_2D, texture );

// Make a Texture2D from the software buffer we rendered
glTexImage2D(GL_TEXTURE_2D, 0, GL_RGBA, m_pSoftwareBuffer->w, m_pSoftwareBuffer->h, 0,
	GL_BGRA,GL_UNSIGNED_BYTE, m_pSoftwareBuffer->pixels );

// Mipmap filtering. Has to be set for texture to render
glTexParameteri(GL_TEXTURE_2D, GL_TEXTURE_MIN_FILTER, GL_NEAREST);
glTexParameteri(GL_TEXTURE_2D, GL_TEXTURE_MAG_FILTER, GL_NEAREST);

// Draw fullscreen quad with software renderer output as texture
glEnable(GL_TEXTURE_2D);
{
	glBegin(GL_QUADS);
	{

		const auto uvLeft = 0.0f;
		const auto uvRight = 1.0f;
		const auto uvTop = 0.0f;
		const auto uvBottom = 1.0f;
				
		glTexCoord2f(uvLeft, uvBottom); glVertex3f(-1.f, -1.f, -1.f);
		glTexCoord2f(uvLeft, uvTop); glVertex3f(-1.f, 1.f, -1.f);
		glTexCoord2f(uvRight, uvTop); glVertex3f(1.f, 1.f, -1.f);
		glTexCoord2f(uvRight, uvBottom); glVertex3f(1.f, -1.f, -1.f);
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


[Github Link](https://github.com/DatTestBench/HybridRenderer)

<br>

[<- Back](../index.md)