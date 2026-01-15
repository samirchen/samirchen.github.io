---
layout: post
title: AI 魔法提示词集锦第 4 期：氛围感
description: 这期我们来展示使用提示词让 AI 创作迷人的氛围感人物。
category: blog
tag: AI
---





## 1、躺着自拍的氛围感


Gemini Nano Banana Pro 出图示例：

![氛围感美女](https://gjzkeyframe.github.io/assets/resource/aigc-prompt/aigc-prompt-4-1.jpeg)


![氛围感美女](https://gjzkeyframe.github.io/assets/resource/aigc-prompt/aigc-prompt-4-2.jpeg)


![氛围感美女](https://gjzkeyframe.github.io/assets/resource/aigc-prompt/aigc-prompt-4-3.jpeg)




Prompt：

```
Vertical, realistic low-light phone selfie in a dim bedroom at night. Same face as reference (no changes). Cool bluish-purple screen glow lighting on a pale face; background mostly dark with beige blackout curtains on the left. Sleepy, cozy, late-night lo-fi vibe.
Close-up, slightly low angle. Subject lying on stomach, propped on a grey pillow with subtle pattern. Messy black bedhead hair with wispy bangs. Soft, dreamy/blank stare, mouth slightly open. Hand near face, index finger touching lowerlip.
Wearing a black sleeveless camisole; thin lace strap visible, left shoulder bare. Soft focus, visible grain/noise, high contrast between lit face and dark background.
Negative Prompt:
bright daylight, sunshine, studio/flash lighting, warm/orange tones, outdoor, sharp focus, professional camera, 4k clean, smooth skin, heavy makeup, cartoon, anime, 3D render, deformed hands, extra/missing fingers, out-of-focus face.
```

出处：[https://x.com/SimplyAnnisa/status/2007876757119287381](https://x.com/SimplyAnnisa/status/2007876757119287381)


---

本文所介绍的 AI 人物生成，也可以使用 FaceXSwap 端侧离线换脸生成。

![FaceXSwap 端侧离线换脸生成新内容](https://gjzkeyframe.github.io/assets/resource/aigc-product/facexswap_preview_short.gif)

---


## 2、从下往上仰拍镜头的氛围感

Gemini Nano Banana Pro 出图示例：

![仰拍镜头](https://gjzkeyframe.github.io/assets/resource/aigc-prompt/aigc-prompt-4-4.jpeg)
仰拍镜头_

![仰拍镜头](https://gjzkeyframe.github.io/assets/resource/aigc-prompt/aigc-prompt-4-5.jpeg)


Prompt:

```
{
  "visual_narrative": {
    "subject": "一个充满活力的年轻女性[用户肖像]，以动态的背面姿势被捕捉。她背向相机，但扭转上半身回头向下看镜头，表情迷人、富有吸引力。一只手插入头发 向后捋着头发，右腿夸张地高高勾起在身后，动作可爱而充满活力。她穿着[服装/装束]，在天空背景下被强烈照亮。",
    "environment": "一个风格化的城市路口。视觉主导是天空：极其深邃、饱和且透明的天蓝（偏光效果），配以清晰闪亮的白云。街灯杆和现代摩天楼的锐利黑色剪影构成边缘框架，与强烈的蓝色形成图形化的对比。",
    "composition": "极端低角度（蚯蚓视角），几乎垂直向上。使用16mm广角镜头扭曲透视，使被摄者抬起的脚和腿显得更大，被摄者在观者面前显得高大，从而加强姿势的俏皮气场与权威感。"
  },
  "technical_parameters": {
    "camera_angle": "垂直向上倾斜 / 地面视角。",
    "lens_choice": "16mm 超广定焦。",
    "lighting": "硬质、高强度直闪（带色片或裸灯泡）。前景主体被冷色、清脆的闪光灯照射以冻结动作并形成清晰轮廓，使她与背景明显分离。背景天空曝光降低1–2档，以实现浓郁、深邃、鲜艳的电光蓝饱和度，模仿“昼夜替换”或高端时尚闪光摄影的美学。",
    "color_palette": "富士经典负片（NC）改版。强调高对比与“偏光”蓝色。天空应为充满活力、透明的钴蓝色，而在闪光灯下肤色保持明亮如珠光。冷色阴影，锐利高光。",
    "aspect_ratio": "3:4"
  },
  "mood_atmosphere": "高电压能量感、Y2K 美学、清晰如晶、鲜活、俏皮、自信、超现实。",
  "negative_prompt": "薄雾, 灰色天空, 褪色的蓝色, 阴天光线, 柔和光线, 僵硬姿势, 解剖畸形, 扭曲的脖子, 多余的四肢, 模糊的面部, 低对比度, 杂乱的背景元素, 暗淡的颜色, 噪点, 颗粒, 平面图像"
}
```


## 3、冬日雪地的氛围感

Gemini Nano Banana Pro 出图示例：

![冬日雪地](https://gjzkeyframe.github.io/assets/resource/aigc-prompt/aigc-prompt-4-6.jpeg)



Prompt:

```
{
  "reference_analysis": {
    "style": "Social media aesthetic photography, soft focus, high-key exposure,  stylish portrait",
    "lighting": "Bright natural daylight, strong snow reflection acting as fill light, crisp shadows on snow surface",
    "colors": "Dominant icy blues and bright whites, pastel accents, creamy fur tones",
    "mood": "Playful, cute, romantic, wintery, innocent",
    "composition": "High angle shot looking down, subject sitting in snow"
  },
  "subject": {
    "main": "Close-up portrait of a young woman in winter attire making a heart shape with her gloved hands around her eye",
    "details": [
      "wearing a fluffy white fur hat",
      "light blue coat with large fur cuffs",
      "cute patterned gloves",
      "winking playfully",
      "snowy background texture"
    ]
  },
  "style": {
    "type": "High-quality lifestyle photography",
    "reference": "Soft, airy, dreamy filter, optimized color grading for fresh skin tones and vibrant snow"
  },
  "lighting": {
    "type": "Bright natural sunlight with snow reflection",
    "mood": "Radiant, airy, high-key, sparkling"
  },
  "color": {
    "palette": [
      "bright white",
      "icy blue",
      "soft cream",
      "pale pink"
    ],
    "mood": "Cool toned, fresh, clean, pastel aesthetic"
  },
  "mood": "Cheerful, romantic, cute, lively winter atmosphere",
  "technical_tags": [
    "close-up shot",
    "zoomed in perspective",
    "high angle",
    "depth of field",
    "soft focus",
    "color graded",
    "best quality",
    "photorealistic",
    "8k resolution",
    "sparkling snow texture"
  ]
}
```

出处：[https://x.com/sidona/status/2011264608556954052](https://x.com/sidona/status/2011264608556954052)


## 4、厨房和面的生活氛围感

Gemini Nano Banana Pro 出图示例：

![厨房和面的生活氛围感](https://gjzkeyframe.github.io/assets/resource/aigc-prompt/aigc-prompt-4-7.jpeg)


![厨房和面的生活氛围感](https://gjzkeyframe.github.io/assets/resource/aigc-prompt/aigc-prompt-4-8.jpeg)


![厨房和面的生活氛围感](https://gjzkeyframe.github.io/assets/resource/aigc-prompt/aigc-prompt-4-9.jpeg)


Prompt:

```
{
  "prompt_type": "descriptive_portrait",
  "subject_details": {
    "demographics": "Young female, sun-kissed skin, fit and athletic build, strictly matching the reference photo.",
    "facial_features": {
      "expression": "Laughing with mouth open, a smudge of white flour on her nose and cheek, playful and messy yet attractive, facial expression matching the reference photo.",
      "eyes": "Bright green eyes, aligned with the reference photo angle.",
      "hair": "Slightly messy short-to-medium hair, casually styled, exactly following the reference photo hairstyle."
    },
    "apparel": {
      "dress": "Plain black t-shirt fitting exactly like the reference photo pose, paired with tiny pajama shorts.",
      "accessories": "None.",
      "footwear": "Barefoot."
    }
  },
  "pose_and_action": {
    "body_position": "Leaning over the kitchen island counter, elbows resting on the surface, body position strictly following the reference photo.",
    "hands": "Hands lightly dusted with flour. One hand holds the phone, the other is near her face, matching the reference photo hand positions exactly.",
    "camera_angle": "Eye-level mirror selfie captured in a reflective kitchen cabinet or window, camera angle matching the reference photo precisely."
  },
  "background_environment": {
    "location": "Messy kitchen.",
    "lighting_source": "Bright overhead kitchen lights.",
    "objects": {
      "details": "Bowl of dough, scattered flour on the counter, cracked eggshells."
    }
  },
  "technical_specs": {
    "style": "Playful, domestic, ultra-realistic, high detail on natural skin texture and flour dust, strict adherence to the reference photo for realism.",
    "aspect_ratio": "4:5"
  },
  "constraints": [
    "Preserve facial identity exactly as in the reference image",
    "No facial reshaping or beautification",
    "No artificial filters or stylization",
    "Maintain natural proportions and lighting"
  ],
  "output_goal": "Create a playful, ultra-realistic mirror selfie portrait of a young woman in a messy kitchen, laughing naturally with flour on her face, perfectly matching the reference photo in pose, expression, and identity."
}
```



出处：[https://x.com/saniaspeaks_/status/2011257477275468178](https://x.com/saniaspeaks_/status/2011257477275468178)


---

推荐一款端侧离线换脸工具，如果你想快速二创人物作品，可以使用 [FaceXSwap](https://www.facexswap.com) 在手机端换脸实现即可：

![FaceXSwap 换脸生成作品](https://gjzkeyframe.github.io/assets/resource/aigc-product/facexswap_preview_short.gif)


![在 AppStore 搜索 'facexswap'](https://gjzkeyframe.github.io/assets/resource/aigc-product/facexswap-2.png)


![FaceXSwap](https://gjzkeyframe.github.io/assets/resource/aigc-product/facexswap.png)


- FaceXSwap 官网：<a href="https://www.facexswap.com" target="_blank">FaceXSwap: On-Device Offline AI Face Swap for Free</a>
- FaceXSwap iOS App 下载：<a href="https://apps.apple.com/app/id6752116909  " target="_blank">FaceXSwap iOS App Download</a>



[SamirChen]: http://www.samirchen.com "SamirChen"
[1]: {{ page.url }} ({{ page.title }})
[2]: http://samirchen.com/aigc-prompts-collection-4/