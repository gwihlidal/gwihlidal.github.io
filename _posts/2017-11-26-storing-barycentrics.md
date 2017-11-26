---
layout: post
title: Storing Barycentric Coordinates
description: "Just about everything you'll need to style in the theme: headings, paragraphs, blockquotes, tables, code blocks, and more."
modified: 2017-11-26
tags: [barycentric, amd, nvidia, fastgs, vbuffer, visibility buffer, gpu, hlsl, spirv, post]
image:
  feature: barycentrics.png
  credit:
  creditlink:
---

Using a visibility buffer (vbuffer) to accelerate sample shading has been a very interesting topic since the original technique was presented.[^1]

The general idea behind a vbuffer is..X



Lorem ipsum dolor sit amet, test link adipiscing elit. **This is strong**. Nullam dignissim convallis est. Quisque aliquam.

![Smithsonian Image]({{ site.url }}/images/3953273590_704e3899d5_m.jpg)
{: .image-right}

*This is emphasized*. Donec faucibus. Nunc iaculis suscipit dui. 53 = 125. Water is H<sub>2</sub>O. Nam sit amet sem. Aliquam libero nisi, imperdiet at, tincidunt nec, gravida vehicula, nisl. The New York Times <cite>(That’s a citation)</cite>. <u>Underline</u>. Maecenas ornare tortor. Donec sed tellus eget sapien fringilla nonummy. Mauris a ante. Suspendisse quam sem, consequat at, commodo vitae, feugiat in, nunc. Morbi imperdiet augue quis tellus.

HTML and <abbr title="cascading stylesheets">CSS<abbr> are our tools. Mauris a ante. Suspendisse quam sem, consequat at, commodo vitae, feugiat in, nunc. Morbi imperdiet augue quis tellus. Praesent mattis, massa quis luctus fermentum, turpis mi volutpat justo, eu volutpat enim diam eget metus.

### Blockquotes

> Lorem ipsum dolor sit amet, test link adipiscing elit. Nullam dignissim convallis est. Quisque aliquam.

## List Types

### Ordered Lists

1. Item one
   1. sub item one
   2. sub item two
   3. sub item three
2. Item two

### Unordered Lists

* Item one
* Item two
* Item three

## Tables

| Header1 | Header2 | Header3 |
|:--------|:-------:|--------:|
| cell1   | cell2   | cell3   |
| cell4   | cell5   | cell6   |
|----
| cell1   | cell2   | cell3   |
| cell4   | cell5   | cell6   |
|=====
| Foot1   | Foot2   | Foot3
{: rules="groups"}

## Code Snippets

Generic Geometry Shader:
```glsl
#version 450

#extension GL_ARB_separate_shader_objects : enable
#extension GL_ARB_shading_language_420pack : enable
#extension GL_ARB_shader_image_load_store : enable

layout(triangles) in;
layout(triangle_strip, max_vertices=3) out;

in layout (location = 1) vec3 g_inNormal[];
in layout (location = 2) vec2 g_inTexcoord[];

out layout (location = 0) vec2 g_outTexcoord;
out smooth layout (location = 1) vec3 g_outBarycentrics;
out layout (location = 2) flat uint g_primitiveId;
out layout (location = 3) flat uint g_drawId;

void main()
{
	gl_Position = gl_in[0].gl_Position;
	g_outTexcoord = g_inTexcoord[0];
	g_outBarycentrics = vec3(1, 0, 0);
	g_primitiveId = gl_PrimitiveID;
	g_drawId = 0;
	gl_Layer = 0;
	EmitVertex();

	gl_Position = gl_in[1].gl_Position;
	g_outTexcoord = g_inTexcoord[1];
	g_outBarycentrics = vec3(0, 1, 0);
	g_primitiveId = gl_PrimitiveID;
	g_drawId = 0;
	gl_Layer = 0;
	EmitVertex();

	gl_Position = gl_in[2].gl_Position;
	g_outTexcoord = g_inTexcoord[2];
	g_outBarycentrics = vec3(0, 0, 1);
	g_primitiveId = gl_PrimitiveID;
	g_drawId = 0;
	gl_Layer = 0;
	EmitVertex();
}
```

Generic Pixel Shader:
```glsl
#version 450

#extension GL_ARB_separate_shader_objects : enable
#extension GL_ARB_shading_language_420pack : enable
#extension GL_ARB_shader_image_load_store : enable
#extension GL_GOOGLE_include_directive : enable

#include "VisibilityCommon.inc"

layout(early_fragment_tests) in;

layout (location = 0) in vec2 uv_in;
layout (location = 1) in smooth vec3 barycentrics_in;
layout (location = 2) in flat uint primitiveId_in;
layout (location = 3) in flat uint drawId_in;

// Y = I,J bary coords, K can be calculated
layout (location = 0) out uvec2 id_and_barys;
// X,Y = dFdX, dFdY
layout (location = 1) out uvec2 derivs;
layout (location = 2) out vec4 debug_out;

layout(origin_upper_left) in vec4 gl_FragCoord;

void main()
{
	debug_out = vec4(barycentrics_in, 0);

	id_and_barys.x = calculateVisibilityId(true, drawId_in, primitiveId_in);
	id_and_barys.y = packUnorm2x16(debug_out.xy);

	derivs.x = packSnorm2x16(dFdx(uv_in));
	derivs.y = packSnorm2x16(dFdy(uv_in));
}
```

AMD Vertex Shader:
```glsl
#version 450

#extension GL_ARB_separate_shader_objects : enable
#extension GL_ARB_shading_language_420pack : enable
#extension GL_ARB_shader_draw_parameters : enable

#define kProjViewMatricesBindingPos 0
#define kModelMatricesBindingPos 0

layout (location = 0) in vec3 pos;
layout (location = 2) in vec2 uv_in;

layout (location = 0) flat out uint draw_id;
layout (location = 1) out vec2 uv_out;
layout (location = 2) flat out vec4 pos0;
layout (location = 3)      out vec4 pos1;

layout (constant_id = 0) const uint num_materials = 1U;
layout (constant_id = 1) const uint num_lights = 1U;

struct Light
{
	vec4 pos_radius;
	vec4 diff_colour;
	vec4 spec_colour;
};

// Could be packed better but it's kept like this until optimisation stage
struct MatConsts
{
	vec4 diffuse_dissolve;
	vec4 specular_shininess;
	vec3 ambient;
	/* 32-bit padding goes here on host side, but GLSL will transform
		the ambient vec3 into a vec4 */
	vec3 emission;
};

layout (std430, set = 0, binding = kProjViewMatricesBindingPos) buffer MainStaticBuffer
{
	mat4 proj;
	mat4 view;
	mat4 inv_proj;
	mat4 inv_view;
	Light lights[num_lights];
	MatConsts mat_consts[num_materials];
};

layout (std430, set = 1, binding = kModelMatricesBindingPos) buffer ModelMats
{
	mat4 model_mats[];
};

layout(push_constant) uniform PushConsts
{
	uint val;
}
mesh_id;

void main()
{
	draw_id = mesh_id.val;
	uv_out = uv_in;
	vec4 temp = proj * view * model_mats[draw_id] * vec4(pos, 1.f);
	pos0 = temp;
	pos1 = temp;
	gl_Position = temp;
}
```

AMD Intrinsics Pixel Shader:[^2]
```glsl
#version 450

#extension GL_ARB_separate_shader_objects : enable
#extension GL_ARB_shading_language_420pack : enable
#extension GL_AMD_shader_explicit_vertex_parameter : enable
#extension GL_ARB_shader_image_load_store : enable
#extension GL_GOOGLE_include_directive : enable

#include "VisibilityCommon.inc"

layout(early_fragment_tests) in;

layout (location = 0) flat in uint draw_id;
layout (location = 1) in vec2 uv_in;
layout (location = 2) flat in vec4 pos0;
layout (location = 3) __explicitInterpAMD in vec4 pos1;

// Y = I,J bary coords, K can be calculated
layout (location = 0) out uvec2 id_and_barys;
// X,Y = dFdX, dFdY
layout (location = 1) out uvec2 derivs;
layout (location = 2) out vec4 debug_out;

void main()
{
	id_and_barys.x = calculateVisibilityId(true, draw_id, gl_PrimitiveID);

	vec4 v0 = interpolateAtVertexAMD(pos1, 0);
	vec4 v1 = interpolateAtVertexAMD(pos1, 1);
	vec4 v2 = interpolateAtVertexAMD(pos1, 2);

	if (v0 == pos0)
	{
		debug_out.y = gl_BaryCoordSmoothAMD.x;
		debug_out.z = gl_BaryCoordSmoothAMD.y;
		debug_out.x = 1 - debug_out.z - debug_out.y;
	}
	else if (v1 == pos0)
	{
		debug_out.x = gl_BaryCoordSmoothAMD.x;
		debug_out.y = gl_BaryCoordSmoothAMD.y;
		debug_out.z = 1 - debug_out.x - debug_out.y;
	}
	else if (v2 == pos0)
	{
		debug_out.z = gl_BaryCoordSmoothAMD.x;
		debug_out.x = gl_BaryCoordSmoothAMD.y;
		debug_out.y = 1 - debug_out.x - debug_out.z;
	}

	derivs.x = packSnorm2x16(dFdx(uv_in));
	derivs.y = packSnorm2x16(dFdy(uv_in));
	id_and_barys.y = packUnorm2x16(debug_out.xy);

	debug_out.z = 0;
}
```

NV FastGS Geometry Shader:
```glsl
// https://www.khronos.org/registry/OpenGL/extensions/NV/NV_geometry_shader_passthrough.txt

#version 450

#extension GL_ARB_separate_shader_objects : enable
#extension GL_ARB_shading_language_420pack : enable
#extension GL_ARB_shader_image_load_store : enable
#extension GL_NV_geometry_shader_passthrough : require

layout (push_constant) uniform PushConstants
{
	float halfViewWidth;
	float halfViewHeight;
}
g_pcs;

layout(triangles) in;

layout(passthrough) in gl_PerVertex
{
	vec4 gl_Position;
}
gl_in[];

in layout (location = 0, passthrough) vec3 g_inPosition[];
in layout (location = 1, passthrough) vec3 g_inNormal[];
in layout (location = 2, passthrough) vec2 g_inTexcoord[];

out layout (location = 3) vec4 g_planeEquationBx;
out layout (location = 4) vec4 g_planeEquationBy;
out layout (location = 5) flat uint g_primitiveId;
out layout (location = 6) flat uint g_drawId;

// Standard plane equation from three points.
vec4 planeEquation(vec3 p0, vec3 p1, vec3 p2)
{
	// Using a standard plane eqn derivation for the barycentrics is potentially inefficient because the z coords are
	// all 0 or 1.  But, IMO this is more comprehensible than expanding out the terms.
	const vec3 v01 = p1 - p0;
	const vec3 v12 = p2 - p1;
	const vec3 n   = cross(v01, v12);
	const float d  = -dot(n, p0);
	return vec4(n, d);
}

vec2 viewportTransform(vec4 p)
{
	float x = (p.x / p.w) * g_pcs.halfViewWidth  + g_pcs.halfViewWidth;
	float y = (p.y / p.w) * g_pcs.halfViewHeight + g_pcs.halfViewHeight;
	return vec2(x,y);
}

void computeBarycentricPlaneEquations(
	in vec4 proj0, in vec4 proj1, in vec4 proj2,
	out vec4 planeEquationBx, out vec4 planeEquationBy)
{
	// Construct planes in (x,y,b) space where x and y are screen-space position and b is one of the
	// barycentric coords.  We need one per barycentric coord.
	vec2 screen0 = viewportTransform(proj0);
	vec2 screen1 = viewportTransform(proj1);
	vec2 screen2 = viewportTransform(proj2);
	planeEquationBx = planeEquation(vec3(screen0, 1), vec3(screen1, 0), vec3(screen2, 0));
	planeEquationBy = planeEquation(vec3(screen0, 0), vec3(screen1, 1), vec3(screen2, 0));
}

void main()
{
	gl_Layer = 0;

	g_primitiveId = gl_PrimitiveID;
	g_drawId = 0;

	computeBarycentricPlaneEquations(
		gl_in[0].gl_Position, gl_in[1].gl_Position, gl_in[2].gl_Position,
		g_planeEquationBx, g_planeEquationBy);
}
```

Pass-through Pixel Shader:

```glsl
#version 450

#extension GL_ARB_separate_shader_objects : enable
#extension GL_ARB_shading_language_420pack : enable
#extension GL_ARB_shader_image_load_store : enable
#extension GL_GOOGLE_include_directive : enable

#include "VisibilityCommon.inc"

layout(early_fragment_tests) in;

layout (location = 0) in vec4 pos0;
layout (location = 1) in vec2 uv_in;
layout (location = 3) in vec4 planeEquationBx_in;
layout (location = 4) in vec4 planeEquationBy_in;
layout (location = 5) in flat uint primitiveId_in;
layout (location = 6) in flat uint drawId_in;

// Y = I,J bary coords, K can be calculated
layout (location = 0) out uvec2 id_and_barys;
// X,Y = dFdX, dFdY
layout (location = 1) out uvec2 derivs;
layout (location = 2) out vec4 debug_out;

layout(origin_upper_left) in vec4 gl_FragCoord;

void main()
{
	// Reconstruct barycentric coords from screen pos and the plane eqns set up by the GS.
	const float bx = -(planeEquationBx_in.w + planeEquationBx_in.x * gl_FragCoord.x + planeEquationBx_in.y * gl_FragCoord.y) / planeEquationBx_in.z;
	const float by = -(planeEquationBy_in.w + planeEquationBy_in.x * gl_FragCoord.x + planeEquationBy_in.y * gl_FragCoord.y) / planeEquationBy_in.z;
	const vec3 barycentrics = vec3(bx, by, 1 - (bx + by)); // Reconstruct 3rd barycentric from the other 2.
	debug_out = vec4(barycentrics, 0);

	id_and_barys.x = calculateVisibilityId(true, drawId_in, primitiveId_in);
	id_and_barys.y = packUnorm2x16(debug_out.xy);

	derivs.x = packSnorm2x16(dFdx(uv_in));
	derivs.y = packSnorm2x16(dFdy(uv_in));
}
```

Non Pygments code example

    <div id="awesome">
        <p>This is great isn't it?</p>
    </div>

## Thanks!

Alex Dunn
Tomasz Stachowiak

## References

[^1]: <http://jcgt.org/published/0002/02/04/>Blah
[^2]: <https://h3r2tic.github.io/>
[^3]: <https://gpuopen.com/stable-barycentric-coordinates/>
[^4]: <https://developer.nvidia.com/sites/default/files/akamai/opengl/specs/GL_NV_geometry_shader_passthrough.txt>
