# Unity Audio-Reactive Compute Shader Grid

A GPU-driven audio visualizer built in Unity that generates a dynamic grid of cubes whose height responds to real-time audio spectrum data. The geometry is entirely computed on the GPU using a compute shader with Perlin noise modulation.

> 🎥 **Tutorial**: [Watch on YouTube](https://www.youtube.com/watch?v=_bxBC56Zx2o) (german)

## How It Works

The system consists of three core components that form a pipeline from audio input to GPU-driven mesh output:

### 1. Audio Spectrum Analysis

Unity's `AudioSource.GetSpectrumData` provides 512 raw FFT samples using a **Blackman-Harris window**. These samples are grouped into 8 frequency bands with exponentially increasing bin sizes:

$$n_i = 2^{i+1}, \quad i \in [0, 7]$$

For each band $i$, the weighted average is computed as:

$$B_i = \frac{10}{n_i} \sum_{j=s_i}^{s_i + n_i - 1} 2 \cdot S_j \cdot (j + 1)$$

where $S_j$ is the sample value at index $j$ and $s_i$ is the starting index of band $i$. The factor $(j+1)$ applies frequency-weighted scaling, emphasizing higher frequencies within each band.

### 2. Band Buffering & Normalization

Raw frequency values are smoothed through a **decay buffer** to prevent harsh visual flickering. For each band $i$ at frame $t$:

$$\text{buf}_i^{(t)} = \begin{cases} B_i & \text{if } B_i > \text{buf}_i^{(t-1)} \\[6pt] \text{buf}_i^{(t-1)} - \dfrac{\text{buf}_i^{(t-1)} - B_i}{8} & \text{otherwise} \end{cases}$$

Bands are then normalized against their observed peak to produce values in $[0, 1]$:

$$\hat{B}_i = \text{clamp}\!\left(\frac{B_i}{\max_{t}(B_i)},\ 0,\ 1\right)$$

The global **amplitude buffer** aggregates all bands:

$$A = \frac{\sum_{i=0}^{7} \hat{B}_i}{\max_{t}\!\left(\sum_{i=0}^{7} \hat{B}_i\right)}$$

### 3. Compute Shader — GPU Cube Generation

A compute shader dispatched with `[numthreads(8, 8, 1)]` generates an $W \times L$ grid of cubes entirely on the GPU. Each thread computes one cube.

**Perlin Noise** is evaluated per cube using a time-evolving seed:

$$n(\mathbf{p}) = \text{noise}(x_i,\ y_i,\ t_{\text{seed}})$$

The cube height $h$ is derived by combining the noise value with the audio amplitude buffer $A$ through a smoothstep transition:

$$h = \text{lerp}\!\left(10,\ |n|^{-5} \cdot A,\ \text{smoothstep}(0, 1, A)\right)$$

Each cube is defined by **8 vertices** and **36 indices** (12 triangles), written directly into `RWStructuredBuffer` objects. The vertex stride is:

$$\text{stride} = \text{sizeof}(\texttt{float3}) \times 2 = 24 \ \text{bytes}$$

storing position and normal per vertex.

### Pipeline Overview

```
AudioSource ──► FFT (512 samples) ──► 8 Frequency Bands ──► Decay Buffer ──► Normalized Amplitude
                                                                                      │
                                                                                      ▼
                                                              Compute Shader (GPU) ◄───┘
                                                                      │
                                                                      ▼
                                                              W×L Cube Grid Mesh
```

## Parameters

| Parameter | Location | Description | Default |
|---|---|---|---|
| `width` / `length` | CubeGenerator | Grid dimensions | `10 × 10` |
| `gap` | CubeGenerator | Spacing between cubes | `0.5` |
| `seedChangeSpeed` | CubeGenerator | Noise animation speed | `1.0` |
| `amplitude` | CubeGenerator | Base amplitude scalar | — |
| `bandCount` | AudioData | Number of frequency bands | `8` |

## Tech Stack

- **Unity** — Engine and audio pipeline
- **HLSL Compute Shader** — GPU-side cube generation with Perlin noise
- **C#** — Audio FFT analysis, buffer smoothing, and CPU-GPU bridge



