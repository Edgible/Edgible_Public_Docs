# GPU Optimization Guide for M4 Chip

## Current GPU Usage

### ✅ Components Using GPU

1. **Ollama (LLM)** - Native macOS installation
   - Automatically uses Metal Performance Shaders (MPS)
   - Leverages M4 GPU for LLM inference
   - Status: ✅ Optimized

2. **Embedding Service** - Native macOS installation
   - Uses PyTorch with MPS backend
   - Model: `all-MiniLM-L6-v2` (small, fast)
   - Status: ✅ Using GPU, but can be optimized

### ❌ Components NOT Using GPU

1. **Weaviate (Vector DB)** - Docker container
   - No GPU access from Docker on macOS
   - Vector search happens on CPU
   - Status: Limited by Docker limitations

2. **Progento API** - Docker container
   - Only makes HTTP calls, doesn't process data
   - Status: N/A (doesn't need GPU)

## Optimization Opportunities

### 1. Batch Processing for Embeddings ⚡ (HIGH IMPACT)

**Current State:**
- Processing **one chunk at a time**
- Very inefficient for GPU usage
- GPUs excel at parallel processing

**Recommendation:**
- Batch **8-16 chunks** together before generating embeddings
- Better GPU utilization (parallel processing)
- Faster overall processing
- Still memory-efficient (small batches)

**Impact:** 5-10x faster embedding generation

### 2. Larger Embedding Model 🎯 (MEDIUM IMPACT)

**Current State:**
- Using `all-MiniLM-L6-v2` (22MB, 384 dimensions)
- Small model, fast but less accurate

**Options:**
- `BAAI/bge-small-en-v1.5` (33MB, 384 dimensions) - Better accuracy, similar speed
- `BAAI/bge-base-en-v1.5` (128MB, 768 dimensions) - Much better accuracy, still fast with GPU
- `BAAI/bge-large-en-v1.5` (1.3GB, 1024 dimensions) - Best accuracy, requires more memory

**Recommendation:**
- Try `BAAI/bge-base-en-v1.5` for better accuracy with M4 GPU
- M4 GPU can handle larger models efficiently
- Unified memory architecture helps with memory management

**Impact:** Better embedding quality, minimal speed impact with GPU

### 3. Optimize Batch Size in Embedding Service 📊

**Current State:**
- Embedding service uses `batch_size=32` internally
- But we only send 1 text at a time from repo_service

**Recommendation:**
- Send 8-16 chunks per request to embedding service
- Let the service batch internally (already optimized)
- Reduces HTTP overhead

**Impact:** 3-5x faster, better GPU utilization

### 4. Memory Management 💾

**Current State:**
- Aggressive GC after each chunk
- Processing one chunk at a time

**With Batching:**
- Process batches of chunks
- GC after each batch (not each chunk)
- M4 unified memory handles large batches well

**Impact:** Better performance, still memory-safe

### 5. Weaviate GPU (Future) 🔮

**Limitation:**
- Weaviate runs in Docker on macOS
- No GPU passthrough from Docker on macOS
- Vector search happens on CPU

**Future Options:**
- Run Weaviate natively on macOS (if available)
- Or use a different vector DB with native macOS GPU support
- Current: Acceptable performance with CPU

## Recommended Optimizations (Priority Order)

### Priority 1: Enable Batch Processing ⚡

**Implementation:**
- Collect 8-16 chunks before generating embeddings
- Send batches to embedding service
- Better GPU utilization

**Expected Impact:**
- 5-10x faster embedding generation
- Better GPU utilization (70-90% vs 10-20%)
- Still memory-efficient

### Priority 2: Optimize Embedding Model 🎯

**Implementation:**
- Switch to `BAAI/bge-base-en-v1.5`
- Larger model but better accuracy
- M4 GPU can handle it efficiently

**Expected Impact:**
- Better embedding quality (better search results)
- Minimal speed impact with GPU batching
- Still fast with M4 GPU

### Priority 3: Fine-tune Batch Sizes 📊

**Implementation:**
- Tune batch size based on chunk size
- Smaller chunks = larger batches (16-32)
- Larger chunks = smaller batches (8-16)

**Expected Impact:**
- Optimal GPU utilization
- Memory-safe
- Best performance

## M4 Chip Advantages

✅ **Unified Memory Architecture**
- GPU and CPU share memory pool
- Can handle larger batches without issues
- No memory transfer overhead

✅ **Metal Performance Shaders (MPS)**
- Optimized for Apple Silicon
- Better than CUDA on macOS
- Excellent PyTorch support

✅ **Neural Engine**
- Additional ML acceleration
- Used automatically by some frameworks
- Can accelerate certain operations

## Testing GPU Usage

Use the component test to verify GPU usage:

```bash
./scripts/test_components.sh
```

Look for section 6 (GPU Status and Usage):
- Embedding Service should show `device: mps` or `mps:0`
- PyTorch MPS should be available
- Ollama should use Metal/MPS automatically

## Monitoring GPU Usage

**Real-time Monitoring:**
```bash
# Activity Monitor (GUI)
open -a "Activity Monitor"
# Then: Window > Energy tab > Look at GPU column

# Command line (if powermetrics installed)
sudo powermetrics --samplers gpu_power -n 1 -i 1000
```

**During Inference:**
- Look for `ollama` process (LLM inference)
- Look for `Python` process (embedding generation)
- GPU usage should spike during these operations

## Conclusion

Current setup is good but can be optimized:

1. ✅ **Ollama** - Already using GPU optimally
2. ⚡ **Embedding Service** - Using GPU but can batch better
3. ❌ **Weaviate** - Limited by Docker (no GPU access)

**Biggest Win:** Enable batch processing for embeddings (5-10x improvement)

