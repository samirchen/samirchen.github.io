---
layout: post
title: AI 魔法提示词集锦第 2 期：城市的超现实全景和食物的分层动效
description: 这期我们来展示使用提示词让 AI 创作给定城市的超现实的球形全景和食物的分层动效。
category: blog
tag: AI
---

## 1、创作给定城市的超现实的球形全景

Google Gemini 示例出图：

![巴黎的超现实的球形全景](https://gjzkeyframe.github.io/assets/resource/aigc-prompt/aigc-prompt-2-1.jpeg)

![罗马的超现实的球形全景](https://gjzkeyframe.github.io/assets/resource/aigc-prompt/aigc-prompt-2-2.jpeg)

![纽约的超现实的球形全景](https://gjzkeyframe.github.io/assets/resource/aigc-prompt/aigc-prompt-2-3.jpeg)

![迪拜的超现实的球形全景](https://gjzkeyframe.github.io/assets/resource/aigc-prompt/aigc-prompt-2-4.jpeg)

![伦敦的超现实的球形全景](https://gjzkeyframe.github.io/assets/resource/aigc-prompt/aigc-prompt-2-5.jpeg)

![马德里的超现实的球形全景](https://gjzkeyframe.github.io/assets/resource/aigc-prompt/aigc-prompt-2-6.jpeg)


![东京的超现实的球形全景](https://gjzkeyframe.github.io/assets/resource/aigc-prompt/aigc-prompt-2-7.jpeg)



Prompt:

```
Create a hyperrealistic, surreal spherical panorama of [CITY NAME], with its most iconic landmarks and architecture seamlessly curving around the top of a planet-like surface, forming a cohesive miniature world. Integrate sleek, modern 3D typography spelling “[CITY NAME]” naturally into the urban landscape, harmonized with the environment rather than floating unnaturally.

The scene is viewed from a top-down, orbiting perspective, emphasizing the tiny-planet effect and spherical distortion while maintaining architectural clarity.

Soft, natural daylight filters through a partly cloudy sky, casting gentle, realistic shadows across lush greenery, streets, trees, and buildings.

The background transitions smoothly into a dramatic, swirling sky, enhancing the surreal atmosphere without overpowering the city.

Use a natural yet vivid color palette — crisp greens, soft blues, and muted earth tones appropriate to [CITY NAME].

Render in a polished, photorealistic style with fine architectural detail, realistic textures, atmospheric depth, and high visual fidelity.
```


## 2、食物的分层动效



高质量的食物展示视频：

![高质量的食物展示视频](https://gjzkeyframe.github.io/assets/resource/aigc-prompt/aigc-prompt-2-8.gif)




这个作品的创作分为 3 步：

- 创作首帧图
- 创作尾帧图
- 使用 Kling 首尾帧功能生成视频


第一步：生成首帧图

Image Prompt 1: Kling 2.5 Turbo + Nano Banana Pro

```
Overhead, top-down food photography of a vibrant, healthy steak salad bowl on a light beige stone surface. A shallow ceramic bowl filled with finely chopped curly kale as the base. Medium-rare sliced steak arranged neatly across the center, pink interior with seared edges and visible seasoning. On one side, fanned avocado slices with cracked black pepper. Bright red diced bell peppers, crumbled white cheese (feta-style), and finely chopped greens distributed evenly around the bowl. A lemon wedge tucked against the edge of the bowl. A blue-handled fork resting inside the bowl, angled slightly toward the center. Natural soft daylight from above, minimal shadows, clean editorial food styling. Sharp focus, high detail, realistic textures, fresh and appetizing. Modern cookbook / Instagram food photography aesthetic. No hands, no text, no branding, no clutter.
```

第二步：生成尾帧图

Image Prompt 2:

```
Using the original steak salad image as the sole ingredient reference, create a clean, vertically stacked exploded-view visualization of the same salad.

The ingredients are arranged in a strict bottom-to-top order, evenly spaced along a single centered vertical axis, with symmetry, alignment, and visual balance.

Bottom layer (base of the salad):

Finely chopped curly kale and mixed leafy greens, forming a soft, natural pile that anchors the composition.

Middle layers (core ingredients), stacked upward in this order:

– Diced red bell peppers and chopped greens, evenly distributed
– Crumbled white cheese (feta-style), centered and proportionally scaled
– Medium-rare sliced steak, laid flat and neatly fanned, pink interior visible
– Avocado slices, evenly cut and symmetrically fanned

Top layer (finishing elements):

Lemon wedge and light vinaigrette droplets, placed delicately at the top of the image to signal freshness and completion.

All ingredient layers are parallel, evenly spaced, and centered, with no rotation, tilt, or perspective distortion.

Ingredients appear to float gently while maintaining realistic proportions, textures, and color accuracy.

Add short, minimal annotations with thin leader lines.

Alternate caption placement from left to right as you move up the stack to create visual balance and avoid crowding.

Example annotations:

– Leafy greens: “Fresh base, rich in nutrients”
– Steak: “High-quality protein”
– Avocado: “Healthy fats, creamy balance”
– Red pepper: “Natural sweetness & antioxidants”
– Lemon & vinaigrette: “Light finish enhancing natural flavors”

Background is pure white or very light neutral, matte and distraction-free.

Lighting is soft, even, and shadow-minimized with a clean editorial food-photography feel.

Style is premium food photography combined with a technical exploded diagram, suitable for marketing, nutrition education, or app UI.

No hands, no bowl, no clutter, no branding, no dramatic shadows.
```


第三步：使用 Kling 首尾帧功能生成视频

Kling 2.5 Turbo Prompt:

```
A high-angle, cinematic studio shot of a fresh steak salad in a white ceramic bowl, centered on a clean white background. The salad is fully assembled: finely chopped curly kale and leafy greens, medium-rare sliced steak, avocado slices, diced red bell peppers, crumbled white cheese, and a lemon wedge.

After a brief moment of stillness, the salad bursts upward in a controlled, elegant explosion, with each ingredient separating cleanly and moving upward along a vertical axis. The motion is smooth, slow, and weightless—no chaos, no spinning—creating a precise, visually satisfying deconstruction.

The ingredients settle into a perfectly aligned exploded-view composition, hovering in mid-air in distinct horizontal layers, evenly spaced and centered:

– Leafy greens at the bottom
– Red bell peppers and crumbled cheese above
– Medium-rare sliced steak laid flat and neatly fanned
– Avocado slices symmetrically arranged
– Lemon wedge and light vinaigrette droplets at the top

Once the ingredients are fully separated and stable, minimal technical annotation lines and labels fade in, alternating left and right for balance. Text is clean, modern, and readable, connected with thin leader lines, never overlapping the ingredients.

Lighting is soft and diffuse, studio-style, with minimal shadows.

Motion is slow-motion and cinematic, emphasizing clarity and elegance.

Ultra-realistic food textures, high detail, premium editorial aesthetic.

Camera remains locked and steady throughout.

No hands, no people, no clutter, no branding, no background movement.

End on the fully exploded, clearly labeled ingredient view.
```






---

推荐一款端侧离线换脸工具，如果你想快速二创人物作品，可以使用 **FaceXSwap** 在手机端换脸实现即可：

![FaceXSwap 换脸生成作品](https://gjzkeyframe.github.io/assets/resource/aigc-prompt/aigc-prompt-1-10.gif)

![在 AppStore 搜索 'facexswap'](https://gjzkeyframe.github.io/assets/resource/aigc-product/facexswap-2.png)

![FaceXSwap](https://gjzkeyframe.github.io/assets/resource/aigc-product/facexswap.png)

- FaceXSwap 官网：<a href="https://www.facexswap.com" target="_blank">FaceXSwap: On-Device Offline AI Face Swap for Free</a>
- FaceXSwap iOS App 下载：<a href="https://apps.apple.com/app/id6752116909  " target="_blank">FaceXSwap iOS App Download</a>



[SamirChen]: http://www.samirchen.com "SamirChen"
[1]: {{ page.url }} ({{ page.title }})
[2]: http://samirchen.com/aigc-prompts-collection-2/
