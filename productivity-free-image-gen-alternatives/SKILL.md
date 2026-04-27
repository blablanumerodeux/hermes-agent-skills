---
name: free-image-gen-alternatives
description: Free image generation APIs and services that don't require a credit card. Alternatives when MiniMax image-01 is unavailable.
triggers:
  - "free image generation API"
  - "image gen no credit card"
  - "alternatives to paid image AI"
  - "budget image generation"
  - "riverflow openrouter image model"
  - "free image model openrouter"
---

# Free Image Generation Alternatives

## ⚠️ Critical Finding: OpenRouter Has NO Free Image Gen Models

Searched all models on OpenRouter — confirmed: **zero image generation models with `price: "0"`**. All image models are paid:

| Model | Output | Price/token |
|-------|--------|-------------|
| `google/gemini-2.5-flash-image` | image+text | $0.0000003 (≈ free) |
| `google/gemini-3.1-flash-image-preview` | image+text | $0.0000005 |
| `google/gemini-3-pro-image-preview` | image+text | $0.000002 |
| `openai/gpt-5-image-mini` | image+text | $0.0000025 |
| `openai/gpt-5-image` | image+text | $0.00001 |

Cheapest: **Gemini Flash Image** at $0.0000003/token — fractions of a cent per image.

---

## Official Hermes Image Gen: MiniMax image-01 (PAID — Requires Plus $20/mo)

The official way to generate images with Hermes is `mmx image generate --model MiniMax-Image-01` via the mmx-cli. However:

| Plan | Price | Daily limit |
|------|-------|-------------|
| Starter | $10/mo | ❌ Not included |
| **Plus** | **$20/mo** | **50 images/day** |
| Max | $50/mo | 120 images/day |

Image generation is NOT available on free or Starter plans. This is a hard limitation.

---

## Free Alternatives (No Credit Card Required)

### Budgetpixel
- **URL**: https://budgetpixel.com
- **Free tier**: 300 credits to start, no credit card required
- **Models**: Flux, Seedream, GPT Image, 20+ models
- **Best for**: Quick one-off use, no signup
- **Browser**: Navigate to URL, generate directly
- **⚠️ Note**: Credits-based, not truly unlimited. Free tier sufficient for occasional use.

### Pixazo
- **URL**: https://www.pixazo.ai
- **Free tier**: 100 images/day, no credit card required
- **Models**: Flux Schnell (free tier), Stable Diffusion, SDXL
- **Signup**: https://accounts.pixazo.ai/register
- **Endpoint**: `https://gateway.pixazo.ai/flux-1-schnell-text-to-image-xxx/v1`
- **Auth**: `Ocp-Apim-Subscription-Key: YOUR_KEY` header
- **Best for**: CLI/programmatic use, daily image generation

### Fal.ai
- **URL**: https://fal.ai/models/fal-ai/flux-1/schnell/api
- **Free tier**: Generous quota with signup
- **Client**: `npm install @fal-ai/client`
- **Auth**: `FAL_KEY` environment variable
- **Best for**: Developers wanting npm client

### PicLumen
- **URL**: https://piclumen.com
- **Free tier**: Unlimited, no subscription, no credit system
- **Best for**: Unlimited free use without account

---

## Quick Test Commands

```bash
# Budgetpixel — open in browser (simplest, no CLI)
open https://budgetpixel.com

# Pixazo API test (needs API key)
curl -X POST "https://gateway.pixazo.ai/flux-1-schnell-text-to-image-xxx/v1/..." \
  -H "Content-Type: application/json" \
  -H "Ocp-Apim-Subscription-Key: YOUR_KEY" \
  -d '{"prompt": "your prompt", "image_size": "auto"}'
```

---

## Priority Order

1. **Official Hermes (paid)** → `mmx image generate --model MiniMax-Image-01` if on Plus plan
2. **Need something now, zero setup** → Budgetpixel.com (300 free credits)
3. **Need CLI/API access** → Pixazo (100/day, no card)
4. **Want npm client** → Fal.ai
5. **Need truly unlimited free** → PicLumen

---

## Notes

- **OpenRouter image models**: All paid. Cheapest is Gemini Flash Image at ~$0.0000003/token — effectively free but not `price: "0"`
- **MiniMax image-01**: Requires Plus ($20/mo) plan. Free/Starter tiers cannot generate images.
- **Budgetpixel**: Credit-based (300 free), not truly unlimited
- **Pixazo/Fal.ai**: Require API key signup but no credit card
- For production/high-volume: paid MiniMax Plus+ or Pixazo paid plans
