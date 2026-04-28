# Unity Audio-Reactive Compute Shader Grid

[![Unity](https://img.shields.io/badge/Unity-2022.3+-000000?style=flat-square&logo=unity&logoColor=white)](https://unity.com/)
[![C#](https://img.shields.io/badge/C%23-Audio_Pipeline-512BD4?style=flat-square&logo=dotnet&logoColor=white)](https://dotnet.microsoft.com/)
[![HLSL](https://img.shields.io/badge/HLSL-Compute_Shader-5586A4?style=flat-square)](https://learn.microsoft.com/en-us/windows/win32/direct3dhlsl/dx-graphics-hlsl)
[![FFT](https://img.shields.io/badge/FFT-Spectrum_Analysis-FF6F61?style=flat-square)](https://en.wikipedia.org/wiki/Fast_Fourier_transform)
[![GPU](https://img.shields.io/badge/GPU-Procedural_Geometry-77B829?style=flat-square)](https://docs.unity3d.com/Manual/class-ComputeShader.html)

A GPU-driven audio visualiser that generates a dynamic grid of cubes whose height responds to real-time audio spectrum data. The geometry is entirely computed on the GPU using a compute shader with Perlin noise modulation.

> **Tutorial:** [Watch on YouTube](https://www.youtube.com/watch?v=_bxBC56Zx2o) (German)

## Objective

Build a fully GPU-driven audio visualisation pipeline where no geometry exists on the CPU. A compute shader generates an entire cube grid per frame, with cube heights driven by FFT frequency band data and Perlin noise. The project demonstrates the complete path from audio input through spectral analysis to procedural mesh output — all without `GameObject` transforms or CPU-side vertex buffers.

## Methodology

**1. Audio Spectrum Analysis.** Extract frequency data from the audio source via FFT, group it into perceptually meaningful bands, and normalise for visual mapping.

**2. Band Buffering.** Smooth raw frequency values through a decay buffer to prevent harsh visual flickering between frames.

**3. GPU Cube Generation.** Dispatch a compute shader that generates the full grid geometry (vertices, normals, indices) directly into GPU buffers, modulated by the buffered audio amplitude and animated Perlin noise.

## Audio Analysis Pipeline

### Spectrum Extraction

Unity's `AudioSource.GetSpectrumData` provides 512 raw FFT samples using a Blackman-Harris window. These samples are grouped into 8 frequency bands with exponentially increasing bin sizes:

$$n_i = 2^{i+1}, \quad i \in [0, 7]$$

For each band $i$, the weighted average is computed as:

$$B_i = \frac{10}{n_i} \sum_{j=s_i}^{s_i + n_i - 1} 2 \cdot S_j \cdot (j + 1)$$

where $S_j$ is the sample value at index $j$ and $s_i$ is the starting index of band $i$. The factor $(j+1)$ applies frequency-weighted scaling, emphasising higher frequencies within each band.

### Decay Buffer & Normalisation

Raw frequency values are smoothed through a decay buffer. For each band $i$ at frame $t$:

$$\text{buf}_i^{(t)} = \begin{cases} B_i & \text{if } B_i > \text{buf}_i^{(t-1)} \\ \text{buf}_i^{(t-1)} - \dfrac{\text{buf}_i^{(t-1)} - B_i}{8} & \text{otherwise} \end{cases}$$

Bands are then normalised against their observed peak to produce values in $[0, 1]$:

$$\hat{B}_i = \text{clamp}\!\left(\frac{B_i}{\max_{t}(B_i)},\ 0,\ 1\right)$$

The global amplitude buffer aggregates all bands:

$$A = \frac{\sum_{i=0}^{7} \hat{B}_i}{\max_{t}\!\left(\sum_{i=0}^{7} \hat{B}_i\right)}$$

## GPU Cube Generation

A compute shader dispatched with `[numthreads(8, 8, 1)]` generates a $W \times L$ grid of cubes entirely on the GPU. Each thread computes one cube.

### Perlin Noise Modulation

Perlin noise is evaluated per cube using a time-evolving seed:

$$n(\mathbf{p}) = \text{noise}(x_i,\ y_i,\ t_{\text{seed}})$$

The cube height $h$ combines the noise value with the audio amplitude buffer $A$ through a smoothstep transition:

$$h = \text{lerp}\!\left(10,\ |n|^{-5} \cdot A,\ \text{smoothstep}(0, 1, A)\right)$$

### Vertex Layout

Each cube is defined by 8 vertices and 36 indices (12 triangles), written directly into `RWStructuredBuffer` objects.

| Buffer | Content | Stride |
|:---|:---|:---|
| Vertex Buffer | `float3` position + `float3` normal | 24 bytes per vertex |
| Index Buffer | Triangle indices | 4 bytes per index |

## Parameters

| Parameter | Location | Description | Default |
|:---|:---|:---|:---:|
| `width` / `length` | CubeGenerator | Grid dimensions | `10 × 10` |
| `gap` | CubeGenerator | Spacing between cubes | `0.5` |
| `seedChangeSpeed` | CubeGenerator | Noise animation speed | `1.0` |
| `amplitude` | CubeGenerator | Base amplitude scalar | — |
| `bandCount` | AudioData | Number of frequency bands | `8` |

## Getting Started

```bash
git clone https://github.com/maybebool/Unity-AudioReactiveGrid.git
```

1. Open the project in Unity 2022.3+.
2. Attach an `AudioSource` with a music clip to the scene.
3. Press Play — the grid generates and responds to the audio in real time.

**Prerequisites:** Unity 2022.3+, GPU with Compute Shader support.

## Tech Stack

[![Unity](https://img.shields.io/badge/Unity-000000?style=flat-square&logo=unity&logoColor=white)](https://unity.com/)
[![.NET](https://img.shields.io/badge/.NET-512BD4?style=flat-square&logo=dotnet&logoColor=white)](https://dotnet.microsoft.com/)
[![HLSL](https://img.shields.io/badge/HLSL-5586A4?style=flat-square)](https://learn.microsoft.com/en-us/windows/win32/direct3dhlsl/dx-graphics-hlsl)

| Category | Technology |
|:---|:---|
| Engine | Unity 2022.3+ |
| Language | C# — FFT analysis, buffer smoothing, CPU-GPU bridge |
| GPU Compute | HLSL Compute Shader — procedural cube generation with Perlin noise |
| Rendering | `Graphics.DrawProceduralIndirect`, `RWStructuredBuffer` |
| Audio | Unity `AudioSource` FFT with Blackman-Harris windowing |

## Limitations & Future Work

The current system generates a uniform cube grid modulated by a single amplitude value. Possible extensions include:

- Per-band grid regions where different frequency bands drive different sections of the grid
- Alternative geometry (spheres, hexagonal prisms) selectable at runtime
- Colour mapping driven by frequency content (low = warm, high = cool)
- Mesh deformation instead of height scaling for smoother organic surfaces
- VR placement for immersive audio-visual environments
- Multiple audio source support with blended influence zones
