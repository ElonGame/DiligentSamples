# Tutorial09 - Quads

This tutorial shows how to render multiple 2D quads, frequently swithcing textures and blend modes.

![](Screenshot.png)

The tutorial is based on [Tutorial06 - Multithreading](../Tutorial06_Multithreading), but renders
2D quads instead of 3D cubes and changes the states frequently. It also demonstrates *Pipeline State
compatibility feature* - if two PSOs share the same shader resource layout, they can use shader resource
binding objects interchangeably, i.e. SRBs created by one PSO can be committed when another PSO is bound.

## Shaders

The tutorial has two rendering modes, non-batched (Batch Size parameter equals 1) and batched 
(Batch Size parameter greater than 1). In the first mode, the quads are rendered one-by-one.
In the second mode, the quads are rendered in small batches, which is more efficient.

Non-batched vertex shader generates two triangles and applies rotation, scale and offset transformation.
It does not use any input other than the vertex id. Quad transformation 2x2 matrix is packed into one `float4` vector.

```hlsl
cbuffer QuadAttribs
{
    float4 g_QuadRotationAndScale; // float2x2 doesn't work
    float4 g_QuadCenter;
};

struct PSInput 
{ 
    float4 Pos : SV_POSITION; 
    float2 uv : TEX_COORD;
};

PSInput main(in uint VertID : SV_VertexID) 
{
    float4 pos_uv[4];
    pos_uv[0] = float4(-1.0,+1.0, 0.0,0.0);
    pos_uv[1] = float4(-1.0,-1.0, 0.0,1.0);
    pos_uv[2] = float4(+1.0,+1.0, 1.0,0.0);
    pos_uv[3] = float4(+1.0,-1.0, 1.0,1.0);

    float2 pos = pos_uv[VertID].xy;
    float2x2 mat = MatrixFromRows(g_QuadRotationAndScale.xy, g_QuadRotationAndScale.zw);
    pos = mul(pos, mat);
    pos += g_QuadCenter.xy;
    PSInput ps;
    ps.Pos = float4(pos, 0.0, 1.0);
    ps.uv = pos_uv[VertID].zw;
    return ps;
}
```

`MatrixFromRows()` is a function defined by the engine that creates a matrix from rows, in an API-spesific fashion.

Non-batched pixel shader is straightforward and simply samples the texture:

```hlsl
Texture2D g_Texture;
SamplerState g_Texture_sampler; // By convention, texture samplers must use _sampler suffix

struct PSInput 
{ 
    float4 Pos : SV_POSITION; 
    float2 uv : TEX_COORD;
};

float4 main(PSInput ps_in) : SV_TARGET
{
    return g_Texture.Sample(g_Texture_sampler, ps_in.uv).rgbg;
}
```

In batched mode, instancing is used to rendered a number of quads in the same draw call. The quad
attributes (rotation, scale, translation) are fetched by the vertex shader from the vertex buffer:

```hlsl
struct PSInput 
{ 
    float4 Pos : SV_POSITION; 
    float2 uv : TEX_COORD;
    float TexIndex : TEX_ARRAY_INDEX;
};

PSInput main(in uint VertID : SV_VertexID,
             in float4 QuadRotationAndScale : ATTRIB0,
             in float2 QuadCenter : ATTRIB1,
             in float TexArrInd : ATTRIB2)
{
    float4 pos_uv[4];
    pos_uv[0] = float4(-1.0,+1.0, 0.0,0.0);
    pos_uv[1] = float4(-1.0,-1.0, 0.0,1.0);
    pos_uv[2] = float4(+1.0,+1.0, 1.0,0.0);
    pos_uv[3] = float4(+1.0,-1.0, 1.0,1.0);

    float2 pos = pos_uv[VertID].xy;
    float2x2 mat = MatrixFromRows(QuadRotationAndScale.xy, QuadRotationAndScale.zw);
    pos = mul(pos, mat);
    pos += QuadCenter.xy;
    PSInput ps;
    ps.Pos = float4(pos, 0.0, 1.0);
    ps.uv = pos_uv[VertID].zw;
    ps.TexIndex = TexArrInd;
    return ps;
}
```

Quad textures are packed into the texture array. The vertex shader passes texture array index over to 
the pixel shader that uses the index to select texture array slice:

```hlsl
Texture2DArray g_Texture;
SamplerState g_Texture_sampler; // By convention, texture samplers must use _sampler suffix

struct PSInput
{
    float4 Pos : SV_POSITION;
    float2 uv : TEX_COORD;
    float TexIndex : TEX_ARRAY_INDEX;
};

float4 main(PSInput ps_in) : SV_TARGET
{
    return g_Texture.Sample(g_Texture_sampler, float3(ps_in.uv, ps_in.TexIndex)).rgbg;
}
```

## Initializing Pipeline States and Shader Resource Binding Objects

The tutorial creates multiple PSO objects that share the same shaders, but define different blend modes.
Two pipeline states that share the same shader resource layout are called *compatible*. The enigne exposes
`IPipelineState::IsCompatibleWith()` method that checks if two pipeline states are compatible.
Compatible pipeline states can use shader resource binding objects interchangeably. We create shader
resource binding objects using the first pipeline state version, but use them with other PSOs:

```cpp
for (int tex = 0; tex < NumTextures; ++tex)
{
    // Create one Shader Resource Binding for every texture
    // http://diligentgraphics.com/2016/03/23/resource-binding-model-in-diligent-engine-2-0/
    m_pPSO[0][0]->CreateShaderResourceBinding(&m_SRB[tex]);
    m_SRB[tex]->GetVariable(SHADER_TYPE_PIXEL, "g_Texture")->Set(m_TextureSRV[tex]);
}

m_pPSO[1][0]->CreateShaderResourceBinding(&m_BatchSRB);
m_BatchSRB->GetVariable(SHADER_TYPE_PIXEL, "g_Texture")->Set(m_TexArraySRV);
```

Note that we create one SRB per texture for non-batched mode and just one SRB for batched mode.

## Rendering

The tutorial largely uses the same rendering scheme as [Tutorial06 - Multithreading](../Tutorial06_Multithreading). 
If multithreading is enabled, command lists are recorded in parallel by multiple threads and are then executed by the
immediate context. Few important things to note:

Resources are transitioned to correct states before render loop starts:

```cpp
for (size_t i = 0; i < _countof(m_SRB); ++i)
    m_pImmediateContext->TransitionShaderResources(m_pPSO[0][0], m_SRB[i]);
m_pImmediateContext->TransitionShaderResources(m_pPSO[1][0], m_BatchSRB);
```

This avoids checking the states inside every draw command:

```cpp
// Shader resources have been explicitly transitioned to correct states, so
// no COMMIT_SHADER_RESOURCES_FLAG_TRANSITION_RESOURCES flag needed
pCtx->CommitShaderResources(m_SRB[CurrInstData.TextureInd], 0);
```

Every thread uses its own rendering context to avoid contention.
