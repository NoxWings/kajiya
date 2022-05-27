# Global illumination overview

`kajiya` uses a range of techniques to render an approximation of global illumination in real time. It strikes a compromise between performance and correctness. Being a toy renderer free from the constraints necessary to ship games, this compromise is a bit different from those made by the large engines out there. The renderer is a vehicle for learning rather than something strictly pragmatic, and some well-known algorithms are intentionally avoided in order to avoid retracing the same steps.

## Test case

Here's a 1920x1080 image rendered by `kajiya` in 8.3 milliseconds on a Radeon RX 6800 XT.

![image][final kajiya frame]

_[The scene][home interior scene] by Olga Shachneva was exported from Unreal Engine 4 via [Epic's GLTF Exporter][gltf exporter]._

For reference, `kajiya`'s built-in path tracer produces the following image in 30 seconds, tracing around 1000 paths per pixel _(with caustic suppression which increases specular roughness for further bounces; [see here](https://user-images.githubusercontent.com/16522064/170572535-c3784ebd-5211-405d-a038-f8285d83db7e.png) for an image without caustic suppression)_:

![image][reference frame]

This serves to illustrate both the renderer's strengths, as well as weaknesses.

For example, the renderer tends to preserve roughness map detail, as can be seen here:

![image](https://user-images.githubusercontent.com/16522064/170583510-29bddfd9-3e6d-4373-a691-64be99a3a9a7.png) ![image](https://user-images.githubusercontent.com/16522064/170583523-0498182c-ebd3-4984-8de4-2a0ac1708cbf.png)

The overall brightness of the scene is similar, with many features preserved, including complex shadowing on rough specular reflections:

![image](https://user-images.githubusercontent.com/16522064/170583583-acd3b2dd-fbea-49fe-ac34-0c806da49904.png) ![image](https://user-images.githubusercontent.com/16522064/170583592-a553f9cb-1fb9-4945-8f76-0d36333dbc80.png)

Rougher surfaces are more difficult to denoise though, and some explicit bias is used, which can distort rough reflections (hard to see unless flipping between the images):

![image](https://user-images.githubusercontent.com/16522064/170573439-38cd5a26-c0ca-4620-9aac-adb22a4eee61.png) ![image](https://user-images.githubusercontent.com/16522064/170573456-6ba85635-6952-4cf1-93ac-0063663c67b3.png)

Sometimes this can manifest as feature loss:

![image](https://user-images.githubusercontent.com/16522064/170573574-710ce400-f41f-4ee3-a2bd-b71eca1463f6.png) ![image](https://user-images.githubusercontent.com/16522064/170573584-5e855781-9e2f-4665-bc84-c050858453a8.png)

Indirect shadows tend to be rather blurry:

![image](https://user-images.githubusercontent.com/16522064/170573694-8ceb481a-6401-484e-89fc-f12a146bdc6c.png) ![image](https://user-images.githubusercontent.com/16522064/170573710-ad4b06c1-aa84-4e6c-b72c-0749328a164c.png)

Reflections are not traced recursively, resulting in a less punchy look:

![image](https://user-images.githubusercontent.com/16522064/170573846-1b559e3f-ca17-4333-8e6b-f68e40bb7208.png) ![image](https://user-images.githubusercontent.com/16522064/170573852-7e92fd87-2363-4108-a142-e1fe6569e3e1.png)

Complex geometry below a certain scale can result in light leaking and temporal instability:

![image](https://user-images.githubusercontent.com/16522064/170573953-8e113303-5bd4-4770-8053-524e9413dbee.png) ![image](https://user-images.githubusercontent.com/16522064/170573963-67868ebf-f9b1-4230-a7fe-84112f849f95.png)

And finally, comparing against the reference image _without caustic suppression_, multi-bounce specular light transport turns diffuse, reducing contrast, and clamping potentially important features:

![image](https://user-images.githubusercontent.com/16522064/170574376-7068383e-a520-42d8-ac1a-12f0960bb42c.png) ![image](https://user-images.githubusercontent.com/16522064/170574387-9e35f8bf-6a62-4666-a414-d86af46e45e6.png)

Some of those will be possible to improve, but ultimately sacrifices will be necessary to have the global illumination update in real time:

![realtime-gi-update](https://user-images.githubusercontent.com/16522064/170589682-477679d1-dd44-4942-9e5e-670f17a8a182.gif)

## Lighting components

_Note that due to how the images are captured here, there's frame-to-frame variability, e.g. different rays being shot, TAA shimmering, GI fluctuations._

Lighting only

![image](https://user-images.githubusercontent.com/16522064/170574589-6a32b117-0e2d-483b-928e-08b932538f3a.png)

Indirect diffuse

![image][indirect diffuse only]

Reflections

![image][reflections only]

Direct lighting

![image](https://user-images.githubusercontent.com/16522064/169663854-293990e9-11ba-4dd1-927b-857437eb3354.png)

# G-buffer pass: ~1.1ms

The geometry is rasterized into a G-buffer packed in a single `RGBA32` image. The four dwords store:
* Albedo (8:8:8, with one byte to spare)
* Normal (11:10:11)
* Roughness & metalness (2xf16; could be packed more)
* Emissive (shared-exponent rgb9e5)

All dielectrics are forced to 4% F0 reflectance.

G-buffer albedo

![image](https://user-images.githubusercontent.com/16522064/169663999-218fc9c8-5221-4562-b28b-c44e0f3293d9.png)

G-buffer roughness

![image](https://user-images.githubusercontent.com/16522064/169663983-6fb21ced-102b-46ad-bfea-5edf1cfc7609.png)

G-buffer metalness

![image](https://user-images.githubusercontent.com/16522064/169664027-7c80ee34-95f9-4ac5-8a91-915c3c4a4f6a.png)

G-buffer normals

![image](https://user-images.githubusercontent.com/16522064/169663966-7f78616b-b18d-4000-b7d0-da8743f8c610.png)

# Indirect diffuse: ~2.3ms

Indirect diffuse starts with a half-resolution trace. Rays are launched from the world-space positions corresponding to g-buffer pixels. Since the trace happens at half-resolution, only one in four pixels traces a ray. The pixel in a 2x2 tile chosen for this changes every frame.

If the hit point of the ray happens to be visible from the primary camera's point of view, the irradiance from the previous frame is reprojected. Otherwise the geometric attributes returned in the ray payload are used by the ray generation shader to perform lighting. An additional ray is potentially used for sun shadows.

![image](https://user-images.githubusercontent.com/16522064/170575279-63a06fd6-5265-4003-9361-66430994e8af.png)

The output of this pass is not merely radiance but also:
* Normal of the hit point;
* Ray offset from the trace origin to the hit point.

The results are not used directly for lighting calculations, but fed into [ReSTIR][ReSTIR paper] reservoirs.

_ReSTIR ELI5: Each reservoir remembers its favorite sample. Every frame (ish) you feed new candidates into reservoirs, and they maybe change their minds. They can also gossip between each other (spatial resampling). `W` makes the math happy. `M` controls the length of the reservoirs' memory. With just the temporal part, you get slowdown of noise, but lower variance; that means slower temporal convergence though! Spatial resampling speeds it up again because neighbors likely contain "just as good" samples, and favorites flip often again. Spatial reduces quality unless you're VERY careful and also use ray tracing to check visibility. Clamp `M` to reduce the reservoirs' memory, and don't feed spatial back into temporal unless starved for samples._

One-sample reservoirs are stored at half resolution, and along with them, additional information needed for ReSTIR:
* Origin of the ray currently selected by the reservoir;
* Incident radiance seen through the selected ray;
* Normal of the hit point of the selected ray;
* Offset of the hit point from the trace origin for the selected ray.

Through temporal reservoir exchange and _an interpretation_ of [permutation sampling][permutation sampling], ReSTIR selects promising samples. Their incident radiance looks much brighter on average, meaning that it's improving sample quality.

With just temporal reservoir exchange (`M` clamped to 10):

![image](https://user-images.githubusercontent.com/16522064/170576375-75989d99-b350-400a-9ea5-34c803e1953e.png)

And when we add permutation sampling (a form of spatial resampling which gets fed back into temporal resampling in subsequent frames):

![image](https://user-images.githubusercontent.com/16522064/170576464-a608b189-6db0-42f2-a388-4e039e880a26.png)

Note that we have lost some micro-detail due to naively running the spatial part without any occlusion checks, but our subsequent spatial reuse passes will recover that by being a bit more careful.

After one spatial reuse pass using 7 neighbors:

![image](https://user-images.githubusercontent.com/16522064/170576587-5c4b97dc-e5c6-4efe-a501-40c681ce026d.png)

After the second spatial reuse pass using 4 neighbors:

![image](https://user-images.githubusercontent.com/16522064/170576624-63b0a58b-0a8c-40ba-a048-ad24957c2125.png)

The micro-shadowing is regained because spatial reuse performs a minimal screen-space ray march between the center pixel and the hit point of the neighbor (max 4 taps into a half-res depth buffer). Such shadowing is hugely approximate and lossy, but considerably cheaper than additional ray tracing would be.

To get rid of the 2x2 pixel artifacts, the final ReSTIR resolve uses 4 neighbors (reservoirs) to reconstruct a full-resolution image:

![image](https://user-images.githubusercontent.com/16522064/170576733-9c80e460-159b-4303-a234-6e45919f208d.png)

This is then thrown at a fairly basic temporal denoiser which uses color bounding box clamping (and is informed by ReSTIR):

![image](https://user-images.githubusercontent.com/16522064/170576962-c1296ead-8e52-49c5-8c0d-3d46b26cdfe4.png)

Additional noise reduction is performed by TAA at the end of the frame:

![image][indirect diffuse only]

## Sample validation

The above is a formula for fairly stable, but very laggy diffuse bounce. If the lighting in the scene changes, the stored reservoir, ray, and radiance information will not be updated, and thus stale radiance values will be reused through the temporal reservoir exchange. To fix this, we must introduce sample validation from [ReSTIR GI][ReSTIR GI].

The basic premise is simple: we must re-trace the samples kept in reservoirs, and check if the radiance they were tracking is still the same.

Ideally we should do that without a 2x cost on ray tracing.

Due to the spatiotemporal reuse of reservoirs, especially the permutation sampling, we can't do this for a fraction of pixels at a time -- if we update some, they might be replaced by the stale ones in the next frame.

We must instead update all reservoirs at the same time. In order to hide the cost, this happens every third frame, and on that frame, no new candidates are generated for ReSTIR. That is, each frame is either a candidate generation frame, or a validation frame.

As for the actual validation process: when the old and new radiance differ significantly, the `M` of the corresponding reservoir is reduced. Additionally, whenever the ray hits approximately the same point as before, its tracked radiance is also updated. The `M` clamping ensures that next time new candidates are generated, they will take precedence. The radiance update makes reaction even faster. Its position check is necessary due to the validation rays being shot from old positions, which can cause self-intersection problems on moving geometry.

## Micro-detail

For the sake of performance, the ReSTIR implementation in `kajiya` is the biased flavor (see [the paper][ReSTIR paper]). Preserving micro-scale light bounce has proven to be difficult. Every spatial resampling pass erodes detail a bit; after the spatiotemporal permutation sampling and two spatial passes, the image is visibly affected.

First the path-traced reference:

![image](https://user-images.githubusercontent.com/16522064/169713676-4e20796f-033c-47d0-9be0-a6cf6dddbf9b.png)

And a naive real-time version. Notice how the corner on the left is darkened, and that the door frame looks rather artificial:

![image](https://user-images.githubusercontent.com/16522064/169713691-37afd37f-9e1f-4294-8593-d8d720204049.png)

An observation can be made that the corners are not a major source of variance, and don't require all of the ReSTIR machinery:

![image](https://user-images.githubusercontent.com/16522064/169713973-3ebeea21-705e-40b8-9667-bd508ce4414d.png)

Following this observation, the diffuse resolve pass performs a near field - far field split, and constructs the image from two different sources of information:

* For far hits: ReSTIR reservoirs and their associated ray and radiance data;
* For near hits: the raw ray data which is traced every frame to provide candidates for ReSTIR.

A smooth blending factor is used to combine the two. "Nearness" is determined based on screen-space metrics: for points near the camera, the near threshold is low; for points far from the camera, the near threshold is high.

With this tweak applied, we are able to recover much of the micro-detail:

![image](https://user-images.githubusercontent.com/16522064/169714197-f845bb73-0c08-499e-9135-a89ec5663a14.png)

A final complication here comes in the form of the aforementioned ReSTIR sample validation. Since one in three frames does not produce candidates for ReSTIR, it wouldn't have data for the near-field either. While not having new ReSTIR candidates is fine, excluding the near-field from the diffuse resolve pass would bring back some of the darkening and introduce temporal instability. To overcome this, the ray tracing pass is brought back for the validation frame, but it only traces very short rays for the near field. Even with this, the cost of validation frames tends to be lower than that of candidate generation frames.

# Irradiance cache: ~0.55ms

The diffuse ray tracing described above is not recursive, therefore it only provides a single bounce of light. If that was the entire story, the image would be too dark:

![image](https://user-images.githubusercontent.com/16522064/169664388-5e2971bc-a276-4bd4-a4b3-ad230abad801.png)

Compared to the reference:

![reference][reference frame]

One could use path tracing instead of the single-bounce trace, and that's pretty much what [ReSTIR GI][ReSTIR GI] does, however that's a rather expensive proposition. The additional bounces of light are often very blurry, and sometimes (mostly in outdoor scenes) don't significantly contribute to the image.

Instead, `kajiya` uses a low-resolution irradiance cache. It's stored as a set of camera-aligned sparsely-allocated 32x32x32 clip maps.

![image](https://user-images.githubusercontent.com/16522064/169664275-9f66ef15-6405-4a4a-b896-91eb6753b608.png)

Entries (voxels) are allocated on-demand, and deallocated a few frames after they're last used. Note that no requests to the irradiance cache are made if the irradiance can be reprojected from the last frame's _indirect diffuse_ output, therefore voxels in the debug visualization will often flicker to black as they're deallocated:

![gi-overview-irradiance-cache](https://user-images.githubusercontent.com/16522064/170589894-ad010f6c-2bc3-41f3-bbd1-7d6bbec53e17.gif)

The cache is not temporally stable, and does not provide a spatially-smooth sampling method.

On the other hand, it is very quick to react to lighting changes, provides a reasonable approximation to multi-bounce diffuse light transport, and, for its relative simplicity, is quite resistant to light leaks.

Unlike other volumetric GI techniques (such as [DDGI](https://morgan3d.github.io/articles/2019-04-01-ddgi/)), this one does not have a canonical point within each voxel from which the rays would be traced. In fact, that point changes every frame for every voxel. The animation below shows cubes at ray trace origins:

![ircache-trace-origins-debug](https://user-images.githubusercontent.com/16522064/170589967-201649b3-db31-4a5b-9c36-b787cbaf1e20.gif)

The role of the irradiance cache is to answer queries coming from other ray-traced effects. It's never queried directly from any screen pixel; instead, when a diffuse or reflection ray wants to know what the "tail" of light bounces is at its hit point, it asks the cache.

Each query location becomes a candidate for the cache to trace rays from. Among the candidates in a given voxel, one is chosen with uniform probability every frame.

![ircache-query-voting](https://user-images.githubusercontent.com/16522064/170590000-96d7abe6-c4d6-40f7-a2d0-3bc21fbe7f4f.gif)

_Notice how the candidate positions are offset slightly away from hit points; this is because the cache uses spherical traces in order to calculate directional irradiance._

This voting system makes the cache adapt to how it's used. It tackles the otherwise nightmarish case of thin walls in buildings, where the outside is exposed to intense sunlight, while the inside, not seeing the light source, must be pitch black. If an irradiance cache is not output-sensitive, it will eventually run out of resolution, and produce leaks in this case. Here, when the camera is on the inside, the candidates will also be inside, therefore leaks should not happen. Once the camera moves out, the candidates also appear on the outside.

If every voxel is only ever queried at one point, the irradiance cache can even be exact (although many factors make this impossible in practice). Averaged over time, voxels yield mean irradiance of their query points. This is somewhat inspired by the [multiresolution hash encoding by Müller et al.](https://nvlabs.github.io/instant-ngp/): their hash maps allow collisions, and then neural nets learn how to resolve them. The cache in `kajiya` doesn't have any neural nets or multiple overlapping hash maps, but (partially) resolves collisions via a ranking system and normal biasing.

In the animation below, the resolution of the irradiance cache has been reduced, and sky lighting disabled. The interior starts lit, then the sun angle changes, leaving the interior pitch black. Despite the sun still striking one side of the structure, the light does not leak inside as long as the camera is also inside.

![ircache-leak-resistance](https://user-images.githubusercontent.com/16522064/170590019-49021a65-8d7a-48b2-b1f2-84c8c463b8f2.gif)

## Ranking system

For multi-bounce lighting to work, irradiance cache entries should be instantiated not just from the indirect diffuse and reflection rays that originate from the g-buffer, but from the rays that the irradiance cache itself traces to calculate lighting.

This can create a situation where irradiance cache entries on the outside of a structure (such as a building) vote for positions visible from their point of view. If the camera is on the inside, the outside votes can cause leaks:

![ircache-voting-conflict-animation](https://user-images.githubusercontent.com/16522064/170590101-d615c1d2-6f58-4dee-bf71-03c471cb6cf3.gif)

To demonstrate this in practice, we need a more complex scene. Let's consider [Epic's Sun Temple](https://www.unrealengine.com/marketplace/en-US/product/sun-temple), but instantiated a few times:

![image](https://user-images.githubusercontent.com/16522064/170513611-c62008f4-2b56-45cf-b63d-57c390bf6e5e.png)

On the inside, there is a secluded area lit by emissive torches:

![image](https://user-images.githubusercontent.com/16522064/170514578-5dfc2e5a-9275-4188-a3db-33b18d91732c.png)

The sun takes many bounces to get there, losing most of its energy. If we disable the torches, then at this exposure level, the image should be black. And yet, the outside votes cause the inside to light up:

![ircache-voting-conflict-leaks](https://user-images.githubusercontent.com/16522064/170590140-0c6effda-da90-42fd-8ab5-e1b1aa958276.gif)

_Note that for illustration purposes this is still using a reduced irradiance cache resolution._

Intuitively, we don't want a candidate from a further light bounce (counting from the camera) to replace a candidate from an earlier light bounce. To achieve this, each irradiance cache entry stores the lowest light bounce index which accessed it. Anything visible from rays traced from the screen gets _rank 1_. Any irradiance cache entry spawned from a _rank 1_ entry gets _rank 2_, and so on. When a new trace origin vote comes in, it will only be considered if the previous one if the new rank is less or equal that of the previous one.

![ircache-voting-ranking](https://user-images.githubusercontent.com/16522064/170590213-c4d7028f-1152-4528-8e37-38763b7cbe7c.gif)

With ranking in place, the leaks disappear:

![ircache-voting-result](https://user-images.githubusercontent.com/16522064/170590230-e5fb00ad-289b-4956-ac34-841d0e91c2c9.gif)

## Normal biasing

Even with the irradiance cache at normal resolution, there can still be cases where thin surfaces can be seen by indirect rays from both directions. A common occurrence of that is... tables. A table lit from the top should not be causing light leaks at the bottom -- yet that's a difficult case for a meshless irradiance cache.

![ircache-normal-conflict](https://user-images.githubusercontent.com/16522064/170590256-b0232518-4bc9-486c-81ae-354c58f4440e.gif)

In order to reduce those leaks, the look-up position into the irradiance cache is offset by the surface normal:

![ircache-normal-offset](https://user-images.githubusercontent.com/16522064/170590271-fa881632-554e-4965-b4d7-8c7b207b7345.gif)

Please note that this is a tradeoff, and sometimes can result in other kinds of collisions, but it tends to work a bit better on average.

## Irradiance calculation

Each cache entry uses temporal reservoir resampling to calculate irradiance. The reservoirs are stratified via a tiny 4x4 octahedral map, and each frame four of the octahedral map pixels generate new candidates. At hit positions of candidate rays, direct lighting from the sun is calculated, and indirect lighting from the irradiance cache is fed back into itself (no double-buffering; race conditions are fine here).

[ReSTIR GI][ReSTIR GI]-style sample validation is done with another four rays per entry per frame.

After the raygen shader has generated new reservoir candidates, a compute pass convolves the incident spherical radiance from reservoirs into directional irradiance, and stores as L1 spherical harmonics for sampling by other shaders.

# Reflections: ~2.2ms

Much like indirect diffuse, reflections are traced at half resolution. Screen-space irradiance is used whenever the ray's hit point is visible from the primary camera. Reflections are calculated after diffuse, therefore the current frame's data can be used instead of reprojecting the previous frame.

The quality of samples (ray directions) matters a lot here, with blue noise and [VNDF sampling][VNDF sampling] being essential.

Note that even with VNDF, some of the generated rays can end up being "invalid" because they point towards the surface rather than away from it. This is where multiple scattering happens -- the ray bounces off a microfacet, and heads inwards towards another one. Following potentially more bounces, the light either gets absorbed, or emerges out. As suggested by the [simulations done by Eric Heitz et al.](https://eheitzresearch.wordpress.com/240-2/), the multiply-scattered ray distribution still resembles the original BRDF shape. For this reason, when VNDF "fails" to generate an outgoing ray direction, it's simply attempted again (up to a few times), until a valid outgoing direction is found. Conservation of energy is assured by using a preintegrated term at the end of the reflection process instead -- along with accounting for the increase in saturation that multiple scattering causes in metals.

By following this procedure, we make every ray matter. Even then, the image is not very useful at this stage:

![image](https://user-images.githubusercontent.com/16522064/170577440-72a7badf-eef5-478f-967b-fbed05aee335.png)

Back in the days of screen space reflections, we could rely on [filtered importance sampling](https://developer.nvidia.com/gpugems/gpugems3/part-iii-rendering/chapter-20-gpu-based-importance-sampling) to get significant variance reduction. No such luck here -- with ray tracing we don't get prefiltering. Instead, we need to be much better at using those samples.

BRDF importance sampling is great when the scene has fairly uniform radiance. That generally isn't the case in practice. What we need is product sampling: generation of samples proportional to the product of BRDF and incident radiance terms. This is once again accomplished by using ReSTIR.

Similarly to how the indirect diffuse works, we throw the generated samples at temporal reservoir resampling (`M` clamped to 8). The reservoirs will track the more promising samples.

At present, `kajiya` doesn't have spatial reservoir exchange for reflections, but it will certainly come in handy for rough surfaces. Even then, the temporal part alone helps tremendously with smooth and mid-rough materials.

Now that we have the reservoir data, we can proceed to resolve a full-resolution image:

![image](https://user-images.githubusercontent.com/16522064/170577837-1b94b220-6ebd-43a2-aa09-6002bf01ba08.png)

Once again, using half-resolution input results in a pixelated look; the noise level is also way too high. To address both issues, eight half-resolution samples are used in the resolve pass. The spatial sample pattern is based on a projection of BRDF lobe footprint.

![image](https://user-images.githubusercontent.com/16522064/170577988-af972676-41a4-4663-9d7a-c30fd6838199.png)

Combining reservoir resampling with a neighbor-reusing reconstruction filter provides great sample efficiency, although at the expense of implementation complexity. ReSTIR is not directly compatible with the simple _ratio estimation_ techniques used in [some][stochastic all the things] [previous](http://h3.gd/stochastic-ssr/) [work](https://eheitzresearch.wordpress.com/705-2/), but they can be meshed together with enough voodoo magic. Great care is needed to avoid fireflies and black pixels, especially with very smooth materials; more on that in another write up.

This is too noisy, but it's stable enough to feed into a temporal filter. The one here uses [dual-source reprojection][stochastic all the things] and color bounding box clamping (informed by ReSTIR sample validation). Despite its simplicity, it provides decent noise reduction:

![image](https://user-images.githubusercontent.com/16522064/170579762-45b130bc-36dc-4e15-bfbb-8d9554c83629.png)

TAA handles the final denoising:

![reflections only][reflections only]

To illustrate the win from temporal reservoir resampling, here's how the image looks without it:

![image](https://user-images.githubusercontent.com/16522064/170579109-387aba8f-626f-4a33-a480-53b149332207.png)

## Sample validation

Since reflections only use spatial reservoir resampling, they are less sensitive to reusing invalidated reservoirs; we don't need to check them all in the same frame. As such, a simpler scheme is applied here. Instead of temporally staggering the validation traces, they are simply done every frame, but at quarter-resolution (half of the trace resolution).

When a previous ReSTIR sample is detected to have changed sufficiently, its 2x2 quad neighbors are inspected. If a neighbor tracks a point of similar radiance, it is invalidated as well. This get part way there to running the validation at the full trace resolution, at a tiny fraction of the cost.

# Direct sun shadows: ~0.52ms

Shadows are traced at full resolution towards points chosen randomly (with blue noise) on the sun's disk:

![image](https://user-images.githubusercontent.com/16522064/170593831-6d66d128-6ffd-41b0-908b-667af73886c4.png)

They're denoised using a slightly modified version of AMD's [FidelityFX Shadow Denoiser](https://gpuopen.com/fidelityfx-denoiser/). The changes are primarily about integrating it with `kajiya`s temporal reprojection pipeline -- using a shared reprojection map instead of recalculating it from scratch.

![image](https://user-images.githubusercontent.com/16522064/170594102-e229d111-1eee-4e7e-9a8c-949701a709c9.png)

The denoised shadow mask is used in a deferred pass, and attenuates both the diffuse and specular contribution from the sun (sorry, @self_shadow...)

# Miscellaneous

## Screen-space ambient occlusion: ~0.17ms

`kajiya` uses screen-space ambient occlusion, but not for directly modulating any lighting. Instead, the AO informs certain passes, e.g. as a cross-bilateral guide in indirect diffuse denoising, and for determining the kernel radius in spatial reservoir resampling.

It is based on [GTAO](https://iryoku.com/downloads/Practical-Realtime-Strategies-for-Accurate-Indirect-Occlusion.pdf), but keeps the radius fixed in screen-space. Due to how it's used, we can get away with low sample counts and sloppy denoising:

![image](https://user-images.githubusercontent.com/16522064/170668256-bf380f87-0663-49d2-87ab-00d970080580.png)

Without using a feature guide like this, it's easy to over-filter detail:

![image](https://user-images.githubusercontent.com/16522064/170670833-d15f8389-a808-4019-986f-7613aedbdb0e.png)

With the cheap and simple SSAO-based guiding, we get better feature definition:

![image](https://user-images.githubusercontent.com/16522064/170670955-d6703c87-85a2-4b3b-8882-140b17f54228.png)

Note that normally, `kajiya` uses very little in terms of spatial filtering, but it's forced to do it when ReSTIR reservoirs are starved for samples (e.g. upon camera jumps). If we force the spatial filters to actually run, the difference is a lot more pronounced.

Without the SSAO guide:

![image](https://user-images.githubusercontent.com/16522064/170674450-f55baa2b-afc7-482e-9699-fabcb08faf41.png)

And with:

![image](https://user-images.githubusercontent.com/16522064/170674525-4d480d28-ff54-4d9e-8eee-ae575eb163c6.png)

## Sky & atmosphere: ~0.1ms

Atmospheric scattering directly uses [Felix Westin's MinimalAtmosphere](https://github.com/Fewes/MinimalAtmosphere). It drives both the sky and sun color.

A tiny 64x64x6 cube map is generated every frame for the sky. It is used for reflection rays and for sky pixels directly visible to the camera. An even smaller, 16x16x6 cube map is also convolved from this one, and used for diffuse rays.

# Known issues

As alluded to earlier, the global illumination described here is far from perfect. It is a spare-time research project of one person. Getting it to a shippable state would be a journey of its own.

## Reflections reveal the irradiance cache

Reflections are not currently traced recursively. At their hit points, direct lighting is calculated as normal, but indirect lighting is directly sampled from the irradiance cache. This is at odds with the design goals of the irradiance cache -- it is merely a Monte Carlo integration shortcut, and not something to be displayed on the screen. As such, whenever irradiance can't be reprojected from the screen, the blocky nature of the cache is revealed:

![image](https://user-images.githubusercontent.com/16522064/170549116-21a4a8ae-d06c-41af-aa8f-b10095d4b262.png)

The irradiance cache is also not temporally stable, which once again becomes clear in reflections (as large-scale fluctuations):

![reflections-reveal-ircache-flicker](https://user-images.githubusercontent.com/16522064/170590304-c427f499-46ab-407a-91e2-1fb58de7e998.gif)

It it will be possible to improve the stability of the irradiance cache, and hopefully recursive tracing and filtering of reflections will make those issues less severe.

## Noise with small and bright emissive surfaces

If the scene contains sources of very high variance, ReSTIR will fail to sufficiently reduce it. For example, in [this scene by burunduk](https://sketchfab.com/3d-models/flying-world-battle-of-the-trash-god-350a9b2fac4c4430b883898e7d3c431f) lit by emissive torches and candles:

![extreme-variance-flicker](https://user-images.githubusercontent.com/16522064/170590328-5e4993cf-7efd-482a-b279-070f38df3846.gif)

The artifacts become even more pronounced in motion, as newly revealed pixels will not have good samples in reservoirs yet (frame rate reduced to 10Hz for illustration purposes):

![extreme-variance-motion](https://user-images.githubusercontent.com/16522064/170590357-02b64276-23a3-4952-96bf-2df5a4434535.gif)

While it might be possible to improve on this with better spatiotemporal reservoir exchange, this is starting to reach limit of what ReSTIR can do. A path traced version of this scene at _one path per pixel_ looks like this:

![image](https://user-images.githubusercontent.com/16522064/170554333-f3b39bd7-aadf-4374-88f6-6eaccdf4dd45.png)

Those emissive surfaces should be handled as explicit light sources in the future.

## Noise in newly disoccluded areas

The denoising presented here needs additional work. Especially newly revealed areas can appear very noisy.

Stable-state frame:

![image](https://user-images.githubusercontent.com/16522064/170558001-537c0ca1-0b0d-4f12-9a13-15812def532a.png)

After moving a large distance to the left within one frame:

![image](https://user-images.githubusercontent.com/16522064/170558422-c5d25ffe-dbfe-4ea9-a50e-037016993c80.png)

In such circumstances, aggressive spatial filtering could help. Conditionally feeding back the output of spatial reservoir resampling into the temporal reservoirs might also speed up convergence.

# GPU profiler overview

"Events" view in _Radeon GPU Profiler_; please observe the additional annotations under the top chart:

![image](https://user-images.githubusercontent.com/16522064/170564660-48460e41-ca11-4233-91b3-83593b168e7a.png)

`kajiya`'s own performance counters averaged over 30 frames; note that there is some overlap between passes, making this not entirely accurate:

![image](https://user-images.githubusercontent.com/16522064/170565478-32a8618b-64b7-417f-824b-6eb6be402ed4.png)

# Ray count breakdown

There are two types of rays being traced: shadow and "gbuffer". The latter return gbuffer-style information from hit points, and don't recursively launch more rays. Lighting is done in a deferred way. There is just one light: the sun.

* Irradiance cache: usually fewer than 16k cache entries:
  * Main trace: 4/entry * (1 gbuffer ray + 1 shadow ray for the sun)
  * ReSTIR validation trace: 4/entry * (1 gbuffer ray + 1 shadow ray for the sun)
  * Accessibility check: 16/entry _short_ shadow rays

* Sun shadow pass: 1/pixel shadow ray

* Indirect diffuse trace (final gather) done at half-res; every third frame is a ReSTIR validation frame, and instead of tracing new candidates, it checks the old ones, and updates their radiance. In addition to that, the validation frame also traces very short contact rays; on paper it seems like it would be doing more work, but it's actually slightly cheaper, so counting conservatively here:
  * 2/3 frames: regular trace: 0.25/pixel * (1 gbuffer ray + 1 shadow ray)
  * 1/3 frames:
    * validation trace: 0.25/pixel * (1 gbuffer ray + 1 shadow ray)
    * contact trace: 0.25/pixel * (1 gbuffer ray + 1 shadow ray)

* Reflections done at half-res, validation every frame at quarter-res
  * Main trace: 0.25/pixel * (1 gbuffer ray + 1 shadow ray)
  * Validation trace: 0.0625/pixel * (1 gbuffer ray + 1 shadow ray)

Summing it up, we have:

* Irradiance cache: 128k gbuffer rays and 384k shadow rays
* Sun shadows: 1 shadow ray per pixel
* Final gather: 0.25..0.5 gbuffer rays and 0.25..0.5 shadow rays per pixel
* Reflections gather: 0.3125 gbuffer rays 0.3125 shadow rays per pixel

Therefore, averaging:

(0.65/pixel + 128k) gbuffer rays and (1.65/pixel + 384k) shadow rays per frame.

[final kajiya frame]: https://user-images.githubusercontent.com/16522064/170568385-fe4eb7af-3da3-4a25-a55e-3d18ea05f884.png
[reference frame]: https://user-images.githubusercontent.com/16522064/170572191-a867ec5a-426b-4092-8e4d-bde45436b801.png
[indirect diffuse only]: https://user-images.githubusercontent.com/16522064/170574686-07a62b6f-4ad9-4bab-a39c-0535494cb55e.png
[reflections only]: https://user-images.githubusercontent.com/16522064/170574776-c26528e9-87fd-4871-96d5-c76a00c0e12d.png

[home interior scene]: https://www.unrealengine.com/marketplace/en-US/product/home-interior-01
[gltf exporter]: https://www.unrealengine.com/marketplace/en-US/product/gltf-exporter
[ReSTIR GI]: https://research.nvidia.com/publication/2021-06_restir-gi-path-resampling-real-time-path-tracing
[permutation sampling]: https://twitter.com/more_fps/status/1457749362025459715
[ReSTIR paper]: https://cs.dartmouth.edu/wjarosz/publications/bitterli20spatiotemporal.html
[VNDF sampling]: https://jcgt.org/published/0007/04/01/
[stochastic all the things]: http://h3.gd/raytracing-in-hybrid-real-time-rendering/