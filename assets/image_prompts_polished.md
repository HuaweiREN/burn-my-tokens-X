# burn-my-tokens Family — Unified Image Generation Prompts

> Polished for consistency across DALL-E 3, Midjourney v6, and Stable Diffusion XL.  
> All prompts follow the same structure: **Subject → Details → Style → Color → Lighting → Composition → No-Text Rule**.

---

## Style Guidelines (Shared Across All Prompts)

Every prompt should end with this exact declaration:

```
No text, no labels, no typography, no watermarks.
```

**Recommended modifiers to append based on target platform:**

| Platform | Suffix |
|----------|--------|
| DALL-E 3 | `Digital illustration, clean vector art style, crisp edges.` |
| Midjourney v6 | `--ar 16:9 --style raw --v 6` (or `--ar 1:1` for icons) |
| Stable Diffusion XL | `masterpiece, best quality, highly detailed, flat design, vector art style.` |

---

## README Assets | 仓库插图

### 1. Hero / Cover Image | 封面主视觉

**Placement:** Top of README, below the title.  
**Aspect Ratio:** 16:9

```
A majestic phoenix constructed from luminous binary code and interconnected neural network nodes, rising powerfully from decaying clock icons and sand timers. Its wings spread wide, shedding sparks that transform into seven distinct floating symbols: a lightbulb for ideation, a chart document for research, a rocket for MVP, a database cylinder for data, a shield with checkmarks for tests, a magnifying glass for review, and a wrench for refactoring. Deep space-black background with electric blue, vibrant orange, and molten gold light trails. Clean vector-style digital art with high contrast, subtle glow effects, and crisp geometric edges. No text, no labels, no typography, no watermarks.
```

---

### 2. Pipeline Diagram | 流水线流程图

**Placement:** "The Burn Pipeline" section.  
**Aspect Ratio:** 16:9

```
A clean, modern flowchart infographic depicting a 7-stage horizontal pipeline on a pure white background. Seven rounded rectangular nodes are connected by thick, flowing arrows that gradually shift from cool blue on the left to warm orange on the right. Each node contains a single minimalist icon: lightbulb, magnifying glass, rocket, database cylinder, shield, checklist, wrench. Below the pipeline, a circular arrow loops from the final node back to the first, symbolizing iterative refinement. Flat design, minimal soft shadows, enterprise software documentation aesthetic, generous whitespace. No text, no labels, no typography, no watermarks.
```

---

### 3. Skill Family Overview | 家族成员概览

**Placement:** "Family Members" section.  
**Aspect Ratio:** 1:1

```
An isometric honeycomb diagram on a dark charcoal background, featuring seven smaller hexagonal cards surrounding one larger central hexagon. The peripheral hexagons display distinct icons and accent colors: lightbulb in amber yellow, magnifying glass in cobalt blue, rocket in coral red, database cylinder in emerald green, shield in violet purple, checklist in teal, wrench in tangerine orange. The central hexagon is dark graphite with a subtle luminous "X" mark, connected to all surrounding cards via thin glowing lines with soft pulse effects. Clean vector art style, isometric perspective, tech documentation aesthetic, crisp edges, subtle ambient glow. No text, no labels, no typography, no watermarks.
```

---

### 4. Burn Loop Mechanism | 燃烧循环机制

**Placement:** Near the "Burn Contract" or "Core Execution Loop" descriptions.  
**Aspect Ratio:** 16:9

```
A circular loop diagram on a clean white background, illustrating an autonomous burn mechanism. At the top, a glowing token counter feeds into a central gear mechanism. Three compact worker robots branch from the gear in parallel, each carrying documents toward a stacked output tray. A diamond-shaped decision node splits the flow: one path loops back to the gear with a circular arrow, the other path leads to a golden trophy and finish flag. A gentle hand-shaped stop button hovers beside the gear, and a clock icon with a circular arrow indicates auto-resume. Flat illustration style, navy blue and warm orange color scheme, clean vector lines, minimal shadows, generous whitespace. No text, no labels, no typography, no watermarks.
```

---

### 5. Before / After Comparison | 前后对比图

**Placement:** Near the value proposition or quick start.  
**Aspect Ratio:** 16:9

```
A split-screen comparison illustration on a clean white background. Left side: a slumped robot sits beside an emptying hourglass, translucent token particles drifting away like ghosts into a gray void. Right side: the same robot stands upright and energized, surrounded by solid, colorful deliverables — folders, documents, code blocks, bar charts, and green checkmarks. A vibrant flame icon marks the dividing line between the two halves, symbolizing transformation. Clean, friendly, modern flat illustration style with rounded shapes, soft gradients, and cheerful colors. No text, no labels, no typography, no watermarks.
```

---

## burn-my-tokens-idea Skill Prompt Template | 创意技能图片规范

### Prompt Template Structure

When generating prompts for ideas in `TOP_PICKS.md`, follow this exact structure:

```markdown
# Image Prompt: {Idea Name}

{Subject sentence: what the image shows, in one vivid sentence.}
{Detail sentence: specific visual elements, materials, textures, or interactions.}
{Atmosphere sentence: mood, lighting, and environmental qualities.}
{Style sentence: art direction, rendering technique, and composition notes.}
No text, no labels, no typography, no watermarks.
```

**Rules:**
1. Always write in English (best compatibility with all image models).
2. Exactly 4 sentences before the no-text declaration.
3. Sentence 1 = Subject (what). Sentence 2 = Details (how). Sentence 3 = Atmosphere (mood/light). Sentence 4 = Style (art direction).
4. Include at least one concrete style hint: `3D render`, `minimalist illustration`, `tech infographic`, `flat vector art`, `photorealistic`, etc.
5. Include a color palette hint when relevant.
6. Always end with the no-text declaration.

---

### Polished Example

```markdown
# Image Prompt: AI Soil Microbiome Analyzer

A futuristic vertical farm tower with bioluminescent roots and holographic bacteria visualization. Microscopic organisms drift through beams of cyan light that display floating data readouts and nutrient metrics. Clean, optimistic sci-fi atmosphere with soft volumetric lighting and a sense of precision agriculture. Matte white surfaces, soft blue and green palette, 3D render, isometric view, crisp details, subtle glow effects. No text, no labels, no typography, no watermarks.
```

---

## Quick Copy — All Prompts (Platform-Ready)

### DALL-E 3 Ready

```
Hero: A majestic phoenix constructed from luminous binary code and interconnected neural network nodes, rising powerfully from decaying clock icons and sand timers. Its wings spread wide, shedding sparks that transform into seven distinct floating symbols: a lightbulb for ideation, a chart document for research, a rocket for MVP, a database cylinder for data, a shield with checkmarks for tests, a magnifying glass for review, and a wrench for refactoring. Deep space-black background with electric blue, vibrant orange, and molten gold light trails. Clean vector-style digital art with high contrast, subtle glow effects, and crisp geometric edges. Digital illustration, clean vector art style, crisp edges. No text, no labels, no typography, no watermarks.

Pipeline: A clean, modern flowchart infographic depicting a 7-stage horizontal pipeline on a pure white background. Seven rounded rectangular nodes are connected by thick, flowing arrows that gradually shift from cool blue on the left to warm orange on the right. Each node contains a single minimalist icon: lightbulb, magnifying glass, rocket, database cylinder, shield, checklist, wrench. Below the pipeline, a circular arrow loops from the final node back to the first, symbolizing iterative refinement. Flat design, minimal soft shadows, enterprise software documentation aesthetic, generous whitespace. Digital illustration, clean vector art style, crisp edges. No text, no labels, no typography, no watermarks.

Family: An isometric honeycomb diagram on a dark charcoal background, featuring seven smaller hexagonal cards surrounding one larger central hexagon. The peripheral hexagons display distinct icons and accent colors: lightbulb in amber yellow, magnifying glass in cobalt blue, rocket in coral red, database cylinder in emerald green, shield in violet purple, checklist in teal, wrench in tangerine orange. The central hexagon is dark graphite with a subtle luminous "X" mark, connected to all surrounding cards via thin glowing lines with soft pulse effects. Clean vector art style, isometric perspective, tech documentation aesthetic, crisp edges, subtle ambient glow. Digital illustration, clean vector art style, crisp edges. No text, no labels, no typography, no watermarks.

Loop: A circular loop diagram on a clean white background, illustrating an autonomous burn mechanism. At the top, a glowing token counter feeds into a central gear mechanism. Three compact worker robots branch from the gear in parallel, each carrying documents toward a stacked output tray. A diamond-shaped decision node splits the flow: one path loops back to the gear with a circular arrow, the other path leads to a golden trophy and finish flag. A gentle hand-shaped stop button hovers beside the gear, and a clock icon with a circular arrow indicates auto-resume. Flat illustration style, navy blue and warm orange color scheme, clean vector lines, minimal shadows, generous whitespace. Digital illustration, clean vector art style, crisp edges. No text, no labels, no typography, no watermarks.

Before/After: A split-screen comparison illustration on a clean white background. Left side: a slumped robot sits beside an emptying hourglass, translucent token particles drifting away like ghosts into a gray void. Right side: the same robot stands upright and energized, surrounded by solid, colorful deliverables — folders, documents, code blocks, bar charts, and green checkmarks. A vibrant flame icon marks the dividing line between the two halves, symbolizing transformation. Clean, friendly, modern flat illustration style with rounded shapes, soft gradients, and cheerful colors. Digital illustration, clean vector art style, crisp edges. No text, no labels, no typography, no watermarks.
```

### Midjourney v6 Ready

Append `--ar 16:9 --style raw --v 6` to all prompts above. For the Family hexagon, use `--ar 1:1`.

---

*Generated for burn-my-tokens skill family v1.1.0. Place images in `assets/` and update README references.*
