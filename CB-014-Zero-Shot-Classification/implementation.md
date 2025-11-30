# CB-014: Zero-Shot Classification API

## Overview

Zero-shot text classification API supporting multiple models via Hugging Face Transformers.js. This implementation runs locally and supports **CPU, GPU, and Apple Silicon** devices.

**Default Model:** `MoritzLaurer/deberta-v3-large-zeroshot-v2.0` (recommended, best accuracy)

## Technical Stack

| Component | Technology |
|-----------|------------|
| Default Model | `MoritzLaurer/deberta-v3-large-zeroshot-v2.0` |
| Alternative | `Xenova/bart-large-mnli`, `Xenova/nli-deberta-v3-large` |
| Runtime | [Transformers.js](https://huggingface.co/docs/transformers.js) |
| Backend | ONNX Runtime (auto-detects CPU/GPU/MPS) |

## Features

- ✅ **Zero-shot classification** - Classify text into any labels without training
- ✅ **Multi-label mode** - Allow multiple true labels
- ✅ **Batch processing** - Classify multiple texts in one request
- ✅ **Multiple models** - Switch between models at runtime
- ✅ **Auto hardware detection** - Automatically uses GPU/MPS if available
- ✅ **Model caching** - Downloads once, cached locally
- ✅ **Singleton pattern** - Model loaded once, reused across requests

## Available Models

| Alias | Full Path | Description | Accuracy |
|-------|-----------|-------------|----------|
| `deberta-v3-large-zeroshot` | `MoritzLaurer/deberta-v3-large-zeroshot-v2.0` | **Recommended** - Best accuracy | ⭐⭐⭐⭐⭐ |
| `deberta-v3-large` | `Xenova/nli-deberta-v3-large` | DeBERTa NLI model | ⭐⭐⭐⭐ |
| `deberta-v3-base` | `Xenova/nli-deberta-v3-base` | Smaller DeBERTa | ⭐⭐⭐ |
| `bart-large-mnli` | `Xenova/bart-large-mnli` | BART MNLI, widely used | ⭐⭐⭐⭐ |
| `distilbart-mnli` | `Xenova/distilbart-mnli-12-3` | Faster, smaller | ⭐⭐⭐ |
| `mobilebert-mnli` | `Xenova/mobilebert-uncased-mnli` | Fastest, smallest | ⭐⭐ |

## Model Comparison

| Task | DeBERTa-v3-zeroshot | BART-MNLI |
|------|---------------------|-----------|
| "travel" classification | **0.9956** | 0.9938 |
| "urgent" classification | **0.8152** | 0.5227 |
| Sentiment (positive) | **0.9995** | - |
| Topic (finance) | **0.9947** | - |

**DeBERTa-v3-large-zeroshot-v2.0** provides significantly better accuracy, especially for nuanced classification tasks.

## Files Created

| File | Description |
|------|-------------|
| `src/Services/ZeroShotClassificationService.js` | Core classification service with singleton pipeline |
| `src/controllers/classification.controller.js` | Express controller for HTTP endpoints |
| `src/routes/classification.routes.js` | Route definitions with Swagger documentation |

## API Endpoints

### POST `/api/classification/classify`

Classify text into candidate labels.

**Request:**
```json
{
  "text": "I have a problem with my iphone that needs to be resolved asap!",
  "candidateLabels": ["urgent", "not urgent", "phone", "tablet", "computer"],
  "multiLabel": false
}
```

**Response:**
```json
{
  "success": true,
  "sequence": "I have a problem with my iphone that needs to be resolved asap!",
  "labels": ["urgent", "phone", "computer", "not urgent", "tablet"],
  "scores": [0.8152, 0.1779, 0.0052, 0.0007, 0.0006],
  "processingTimeMs": 223,
  "model": "MoritzLaurer/deberta-v3-large-zeroshot-v2.0"
}
```

### POST `/api/classification/batch`

Batch classify multiple texts.

**Request:**
```json
{
  "items": [
    {"text": "I need help with my billing", "candidateLabels": ["billing", "technical", "sales"]},
    {"text": "How do I reset my password?", "candidateLabels": ["billing", "technical", "sales"]}
  ]
}
```

### GET `/api/classification/models`

List available models.

**Response:**
```json
{
  "success": true,
  "models": {
    "deberta-v3-large-zeroshot": "MoritzLaurer/deberta-v3-large-zeroshot-v2.0",
    "bart-large-mnli": "Xenova/bart-large-mnli"
  },
  "currentModel": "MoritzLaurer/deberta-v3-large-zeroshot-v2.0",
  "recommended": "deberta-v3-large-zeroshot"
}
```

### POST `/api/classification/models/switch`

Switch to a different model.

**Request:**
```json
{
  "model": "bart-large-mnli"
}
```

### GET `/api/classification/status`

Get model loading status.

### GET `/api/classification/health`

Health check with test classification.

### POST `/api/classification/preload`

Trigger model preload without classification.

## Configuration

### Environment Variables

| Variable | Default | Description |
|----------|---------|-------------|
| `ENABLE_CLASSIFICATION_API` | `false` | **Enable/disable API and service** |
| `ZERO_SHOT_MODEL` | `deberta-v3-large-zeroshot` | Model alias or full path |
| `ZERO_SHOT_DEVICE` | `null` (auto) | Force device: `cpu`, `gpu` |
| `TRANSFORMERS_CACHE` | `./.cache/transformers` | Model cache directory |
| `CLASSIFICATION_PRELOAD_ON_STARTUP` | `true` | Preload model when server starts |
| `CLASSIFICATION_MAX_BATCH_SIZE` | `50` | Max items per batch |

### Enable/Disable API

```bash
# Enable Classification API (in .env)
ENABLE_CLASSIFICATION_API=true

# Disable Classification API (default)
ENABLE_CLASSIFICATION_API=false
```

When **enabled** (`ENABLE_CLASSIFICATION_API=true`):
- ✅ API routes are registered
- ✅ Model is preloaded on server startup (if `CLASSIFICATION_PRELOAD_ON_STARTUP=true`)
- ✅ Service is available for classification

When **disabled** (default):
- ❌ API routes are NOT registered (returns 404)
- ❌ Model is NOT loaded
- ❌ No memory/resources used

## Performance

| Model | Size | First Load | Inference |
|-------|------|------------|-----------|
| DeBERTa-v3-large-zeroshot | ~800MB | 3-5 min | 70-230ms |
| BART-large-MNLI | ~400MB | 2-4 min | 50-200ms |
| DistilBART-MNLI | ~200MB | 1-2 min | 30-100ms |
| MobileBERT-MNLI | ~100MB | 30s-1 min | 20-50ms |

## Usage Examples

### cURL

```bash
# Simple classification
curl -X POST http://localhost:3001/api/classification/classify \
  -H "Content-Type: application/json" \
  -d '{"text": "one day I will see the world", "candidateLabels": ["travel", "cooking", "dancing"]}'

# Multi-label classification
curl -X POST http://localhost:3001/api/classification/classify \
  -H "Content-Type: application/json" \
  -d '{"text": "one day I will see the world", "candidateLabels": ["travel", "exploration"], "multiLabel": true}'

# List available models
curl -X GET http://localhost:3001/api/classification/models

# Switch model
curl -X POST http://localhost:3001/api/classification/models/switch \
  -H "Content-Type: application/json" \
  -d '{"model": "bart-large-mnli"}'
```

### JavaScript

```javascript
const response = await fetch('http://localhost:3001/api/classification/classify', {
  method: 'POST',
  headers: { 'Content-Type': 'application/json' },
  body: JSON.stringify({
    text: 'This product exceeded my expectations!',
    candidateLabels: ['positive', 'negative', 'neutral']
  })
});
const result = await response.json();
console.log(result.labels[0]); // 'positive' (score: 0.9995)
```

## Hardware Support

| Device | Status | Notes |
|--------|--------|-------|
| **CPU** | ✅ Works | Default fallback |
| **GPU (CUDA)** | ✅ Auto-detected | NVIDIA GPUs |
| **Apple Silicon (MPS)** | ✅ Auto-detected | M1/M2/M3 via Metal |

## Model Information

### MoritzLaurer/deberta-v3-large-zeroshot-v2.0 (Recommended)

Specifically trained for zero-shot classification with superior accuracy. Uses the DeBERTa-v3 architecture with enhanced classification capabilities.

**Reference:** [MoritzLaurer/deberta-v3-large-zeroshot-v2.0](https://huggingface.co/MoritzLaurer/deberta-v3-large-zeroshot-v2.0)

### facebook/bart-large-mnli (Alternative)

NLI-based zero-shot classification by treating text as premise and labels as hypotheses.

**Reference:** [facebook/bart-large-mnli](https://huggingface.co/facebook/bart-large-mnli)

## Dependencies Added

```json
{
  "@huggingface/transformers": "^3.8.0"
}
```

## Swagger Documentation

Access full API documentation at: `http://localhost:3001/api-docs`
