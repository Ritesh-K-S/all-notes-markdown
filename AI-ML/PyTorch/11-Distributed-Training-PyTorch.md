# Chapter 11: Distributed Training in PyTorch

## Table of Contents
- [11.1 Why Distributed Training?](#111-why-distributed-training)
- [11.2 DataParallel (DP)](#112-dataparallel-dp)
- [11.3 DistributedDataParallel (DDP)](#113-distributeddataparallel-ddp)
- [11.4 Fully Sharded Data Parallel (FSDP)](#114-fully-sharded-data-parallel-fsdp)
- [11.5 Model Parallelism](#115-model-parallelism)
- [11.6 Pipeline Parallelism](#116-pipeline-parallelism)
- [11.7 DeepSpeed Integration](#117-deepspeed-integration)
- [11.8 Multi-Node Training](#118-multi-node-training)
- [11.9 Debugging and Profiling Distributed Training](#119-debugging-and-profiling-distributed-training)
- [11.10 Common Mistakes](#1110-common-mistakes)
- [11.11 Interview Questions](#1111-interview-questions)
- [11.12 Quick Reference](#1112-quick-reference)

---

## 11.1 Why Distributed Training?

### What It Is
Distributed training means splitting your training workload across **multiple GPUs or machines** to train faster. Imagine you have 1 million images to classify — instead of one person looking at all of them, you hire 8 people, each looks at 125,000 images, and they share what they learned. That's distributed training in a nutshell.

### Why It Matters
- **Models are getting HUGE** — GPT-3 has 175B parameters (700 GB in FP32). No single GPU can hold it
- **Datasets are massive** — ImageNet (14M images), Common Crawl (petabytes of text)
- **Time is money** — Training BERT on 1 GPU takes ~11 days; on 64 GPUs it takes ~4 hours
- **GPU memory is limited** — Even an A100 has only 80 GB; a large model + optimizer states can exceed this

### The Two Fundamental Strategies

```
┌───────────────────────────────────────────────────────────────────┐
│                    DISTRIBUTED TRAINING                           │
│                                                                   │
│  ┌─────────────────────┐         ┌─────────────────────────────┐ │
│  │   DATA PARALLELISM  │         │     MODEL PARALLELISM       │ │
│  │                     │         │                             │ │
│  │  Same model on      │         │  Model split across         │ │
│  │  each GPU, different│         │  GPUs, same data flows      │ │
│  │  data subsets        │         │  through all                │ │
│  │                     │         │                             │ │
│  │  GPU 0: Model + D₁  │         │  GPU 0: Layers 1-4          │ │
│  │  GPU 1: Model + D₂  │         │  GPU 1: Layers 5-8          │ │
│  │  GPU 2: Model + D₃  │         │  GPU 2: Layers 9-12         │ │
│  │  GPU 3: Model + D₄  │         │  GPU 3: Layers 13-16        │ │
│  │                     │         │                             │ │
│  │  Use when: Model    │         │  Use when: Model doesn't   │ │
│  │  fits on 1 GPU      │         │  fit on 1 GPU              │ │
│  └─────────────────────┘         └─────────────────────────────┘ │
│                                                                   │
│  ┌──────────────────────────────────────────────────────────────┐ │
│  │  HYBRID: FSDP / DeepSpeed ZeRO                              │ │
│  │  Shard model + data + optimizer across all GPUs              │ │
│  │  Best of both worlds — handles any model size                │ │
│  └──────────────────────────────────────────────────────────────┘ │
└───────────────────────────────────────────────────────────────────┘
```

### Key Concepts

| Term | Meaning | Analogy |
|------|---------|---------|
| **World size** | Total number of processes/GPUs | Number of workers in the team |
| **Rank** | Unique ID of each process (0 to world_size-1) | Worker badge number |
| **Local rank** | GPU index on the current machine | Desk number in an office |
| **Process group** | Collection of processes that communicate | A team chat channel |
| **AllReduce** | Sum/average gradients across all GPUs | Everyone sharing their notes |
| **Scatter** | Split data and send pieces to each GPU | Dealing cards to players |
| **Gather** | Collect data from all GPUs to one | Collecting answer sheets |
| **Broadcast** | Send data from one GPU to all | PA announcement |

### Communication Backends

```
┌──────────────┬────────────┬──────────────┬──────────────────────────┐
│ Backend      │ GPU↔GPU    │ CPU↔CPU      │ Best for                 │
├──────────────┼────────────┼──────────────┼──────────────────────────┤
│ NCCL         │ ✅ Fast    │ ❌           │ Multi-GPU (NVIDIA)       │
│ Gloo         │ ⚠️ Slow    │ ✅           │ CPU training, fallback   │
│ MPI          │ ✅         │ ✅           │ HPC clusters             │
└──────────────┴────────────┴──────────────┴──────────────────────────┘
```

---

## 11.2 DataParallel (DP)

### What It Is
`DataParallel` is PyTorch's **simplest** multi-GPU solution. It's a single-line wrapper: take your model, wrap it, done. Think of it as photocopying your model onto each GPU and having a manager (GPU 0) coordinate everything.

### Why It Matters
- **Easiest to use** — literally one line of code
- **Good for prototyping** — quick way to use multiple GPUs
- **But slow in production** — GPU 0 bottleneck, GIL limitations

### How It Works

```
Step 1: Scatter input        Step 2: Replicate model      Step 3: Forward pass
┌─────────┐                  ┌─────────┐                  ┌─────────┐
│ Batch   │──split──▶        │ Model   │──copy──▶         │ GPU 0   │──▶ loss₀
│ [32]    │                  │ (GPU 0) │                  │ GPU 1   │──▶ loss₁
│         │  GPU 0:[16]      │         │  GPU 0: Model    │ GPU 2   │──▶ loss₂
│         │  GPU 1:[16]      │         │  GPU 1: Model    │ GPU 3   │──▶ loss₃
└─────────┘                  └─────────┘                  └─────────┘

Step 4: Gather losses         Step 5: Backward (parallel)  Step 6: Update (GPU 0)
┌─────────┐                  ┌─────────┐                  ┌─────────┐
│ GPU 0   │◀── gather ──     │ grad₀   │                  │ GPU 0   │
│ total   │                  │ grad₁   │──reduce──▶       │ params  │
│ loss    │                  │ grad₂   │   GPU 0          │ updated │
│         │                  │ grad₃   │                  │         │
└─────────┘                  └─────────┘                  └─────────┘
```

> **Problem:** GPU 0 does extra work (scatter, gather, parameter updates). This creates a bottleneck.

### Code Example

```python
import torch
import torch.nn as nn
from torch.utils.data import DataLoader, TensorDataset

# Simple model
class SimpleModel(nn.Module):
    def __init__(self):
        super().__init__()
        self.net = nn.Sequential(
            nn.Linear(784, 512),
            nn.ReLU(),
            nn.Linear(512, 256),
            nn.ReLU(),
            nn.Linear(256, 10)
        )
    
    def forward(self, x):
        return self.net(x)

# Create model
model = SimpleModel()

# ═══════════════════════════════════════════
# ONE LINE to use multiple GPUs
# ═══════════════════════════════════════════
if torch.cuda.device_count() > 1:
    print(f"Using {torch.cuda.device_count()} GPUs!")
    model = nn.DataParallel(model)  # That's it!

model = model.cuda()

# Training loop — UNCHANGED from single-GPU code
optimizer = torch.optim.Adam(model.parameters(), lr=1e-3)
criterion = nn.CrossEntropyLoss()

# Create dummy data
dataset = TensorDataset(torch.randn(1000, 784), torch.randint(0, 10, (1000,)))
dataloader = DataLoader(dataset, batch_size=64, shuffle=True)

for epoch in range(5):
    for batch_x, batch_y in dataloader:
        batch_x, batch_y = batch_x.cuda(), batch_y.cuda()
        
        output = model(batch_x)
        loss = criterion(output, batch_y)
        
        optimizer.zero_grad()
        loss.backward()
        optimizer.step()
    
    print(f"Epoch {epoch+1}, Loss: {loss.item():.4f}")

# ═══════════════════════════════════════════
# Accessing the original model (common gotcha)
# ═══════════════════════════════════════════
# model.module gives the unwrapped model
original_model = model.module if isinstance(model, nn.DataParallel) else model
torch.save(original_model.state_dict(), "model.pth")
```

### Why DataParallel Is NOT Recommended for Production

| Problem | Explanation |
|---------|-------------|
| GPU 0 bottleneck | All gradients gathered to GPU 0; it does 2x work |
| GIL limitation | Single Python process = Python thread contention |
| Memory imbalance | GPU 0 uses more memory (holds all gathered gradients) |
| No multi-node | Only works within a single machine |
| Slow communication | Uses Python threads, not NCCL-optimized |

> **Rule of Thumb:** Use `DataParallel` for quick experiments only. For anything serious, use `DistributedDataParallel`.

---

## 11.3 DistributedDataParallel (DDP)

### What It Is
`DistributedDataParallel` is PyTorch's **production-grade** data parallel solution. Unlike `DataParallel`, it launches **one Python process per GPU**, eliminates the GIL bottleneck, and uses NCCL for highly efficient GPU-to-GPU communication. Think of it as upgrading from one manager with many assistants to a team of equals who coordinate via walkie-talkie.

### Why It Matters
- **2-3x faster** than `DataParallel` in practice
- **Scales to multiple machines** (multi-node training)
- **No GPU 0 bottleneck** — every GPU does equal work
- **The standard** — used by virtually all large-scale training pipelines

### How DDP Works

```
┌─────────────────────────────────────────────────────────────────┐
│                 DDP: Each GPU = Separate Process                 │
│                                                                  │
│  Process 0 (GPU 0)         Process 1 (GPU 1)                    │
│  ┌──────────────────┐     ┌──────────────────┐                  │
│  │ Model (copy)     │     │ Model (copy)     │                  │
│  │ Data subset 0    │     │ Data subset 1    │                  │
│  │ Forward → Loss₀  │     │ Forward → Loss₁  │                  │
│  │ Backward → Grad₀ │     │ Backward → Grad₁ │                  │
│  └────────┬─────────┘     └────────┬─────────┘                  │
│           │                        │                             │
│           └────── AllReduce ───────┘                             │
│                  (NCCL Ring)                                     │
│              avg_grad = (Grad₀ + Grad₁) / 2                    │
│                                                                  │
│  Both GPUs now have identical averaged gradients                │
│  → Both apply same update → Models stay in sync                 │
└─────────────────────────────────────────────────────────────────┘
```

### AllReduce Ring Algorithm

```
Ring AllReduce — Each GPU sends to next, receives from previous
Efficient: Each GPU sends/receives N/P data (P = num GPUs)

Step 1: Scatter-Reduce          Step 2: AllGather
GPU 0 ──▶ GPU 1                 GPU 0 ──▶ GPU 1
  ▲          │                    ▲          │
  │          ▼                    │          ▼
GPU 3 ◀── GPU 2                 GPU 3 ◀── GPU 2

After: Every GPU has the sum of all gradients
Communication cost: 2(P-1)/P × N — nearly optimal!
```

### Complete DDP Training Script

```python
"""
ddp_train.py — Complete DDP training script

Launch with:
  torchrun --nproc_per_node=4 ddp_train.py          # Single machine, 4 GPUs
  torchrun --nproc_per_node=4 --nnodes=2 \           # 2 machines, 4 GPUs each
           --node_rank=0 --master_addr=10.0.0.1 \
           --master_port=29500 ddp_train.py
"""

import os
import torch
import torch.nn as nn
import torch.distributed as dist
from torch.nn.parallel import DistributedDataParallel as DDP
from torch.utils.data import DataLoader, TensorDataset
from torch.utils.data.distributed import DistributedSampler

def setup():
    """Initialize the distributed process group"""
    # torchrun sets these environment variables automatically
    dist.init_process_group(backend="nccl")  # NCCL for GPU communication

def cleanup():
    """Clean up the process group"""
    dist.destroy_process_group()

def get_rank():
    return dist.get_rank()

def get_world_size():
    return dist.get_world_size()

def get_local_rank():
    return int(os.environ["LOCAL_RANK"])


class ResidualBlock(nn.Module):
    def __init__(self, dim):
        super().__init__()
        self.block = nn.Sequential(
            nn.Linear(dim, dim),
            nn.LayerNorm(dim),
            nn.GELU(),
            nn.Linear(dim, dim),
            nn.LayerNorm(dim)
        )
    
    def forward(self, x):
        return x + self.block(x)


class LargeModel(nn.Module):
    def __init__(self, input_dim=784, hidden_dim=1024, num_classes=10, num_blocks=6):
        super().__init__()
        self.embed = nn.Linear(input_dim, hidden_dim)
        self.blocks = nn.Sequential(*[ResidualBlock(hidden_dim) for _ in range(num_blocks)])
        self.head = nn.Linear(hidden_dim, num_classes)
    
    def forward(self, x):
        x = self.embed(x)
        x = self.blocks(x)
        return self.head(x)


def train(epochs=10):
    setup()
    
    local_rank = get_local_rank()
    rank = get_rank()
    world_size = get_world_size()
    
    # ═══════════════════════════════════════════
    # KEY: Set the GPU for this process
    # ═══════════════════════════════════════════
    torch.cuda.set_device(local_rank)
    device = torch.device(f"cuda:{local_rank}")
    
    # ═══════════════════════════════════════════
    # Create model on the correct GPU
    # ═══════════════════════════════════════════
    model = LargeModel().to(device)
    
    # Wrap with DDP — this handles gradient synchronization
    model = DDP(model, device_ids=[local_rank])
    
    # ═══════════════════════════════════════════
    # Create dataset and DISTRIBUTED sampler
    # ═══════════════════════════════════════════
    dataset = TensorDataset(
        torch.randn(10000, 784),
        torch.randint(0, 10, (10000,))
    )
    
    # DistributedSampler ensures each GPU gets DIFFERENT data
    sampler = DistributedSampler(
        dataset,
        num_replicas=world_size,  # Total number of GPUs
        rank=rank,                # This GPU's ID
        shuffle=True
    )
    
    dataloader = DataLoader(
        dataset,
        batch_size=64,            # Per-GPU batch size
        sampler=sampler,          # Use distributed sampler (NOT shuffle=True)
        num_workers=4,
        pin_memory=True           # Faster CPU→GPU transfer
    )
    
    # ═══════════════════════════════════════════
    # Optimizer and loss — same as single GPU
    # ═══════════════════════════════════════════
    optimizer = torch.optim.AdamW(model.parameters(), lr=1e-3, weight_decay=0.01)
    criterion = nn.CrossEntropyLoss()
    
    # Learning rate scaling: linear scaling rule
    # Effective batch = per_gpu_batch × world_size
    # Scale LR proportionally (or use warmup)
    effective_batch = 64 * world_size
    if rank == 0:
        print(f"World size: {world_size}")
        print(f"Effective batch size: {effective_batch}")
    
    # ═══════════════════════════════════════════
    # Training loop
    # ═══════════════════════════════════════════
    for epoch in range(epochs):
        # CRITICAL: Set epoch for shuffling to work correctly
        sampler.set_epoch(epoch)
        
        model.train()
        total_loss = 0.0
        num_batches = 0
        
        for batch_x, batch_y in dataloader:
            batch_x = batch_x.to(device, non_blocking=True)
            batch_y = batch_y.to(device, non_blocking=True)
            
            output = model(batch_x)
            loss = criterion(output, batch_y)
            
            optimizer.zero_grad()
            loss.backward()         # DDP automatically syncs gradients here
            optimizer.step()
            
            total_loss += loss.item()
            num_batches += 1
        
        avg_loss = total_loss / num_batches
        
        # Only print from rank 0 to avoid duplicate output
        if rank == 0:
            print(f"Epoch {epoch+1}/{epochs}, Loss: {avg_loss:.4f}")
    
    # ═══════════════════════════════════════════
    # Save model — ONLY from rank 0
    # ═══════════════════════════════════════════
    if rank == 0:
        # Save the unwrapped model (model.module)
        torch.save(model.module.state_dict(), "model_ddp.pth")
        print("Model saved!")
    
    # Wait for rank 0 to finish saving before cleanup
    dist.barrier()
    cleanup()


if __name__ == "__main__":
    train()
```

### DDP with Gradient Accumulation

When your effective batch size needs to be even larger, or GPU memory is tight:

```python
"""
Gradient accumulation with DDP.
Accumulate gradients over N mini-batches before syncing.
"""

def train_with_accumulation(
    model, dataloader, optimizer, criterion,
    accumulation_steps=4, device='cuda'
):
    model.train()
    optimizer.zero_grad()
    
    for step, (batch_x, batch_y) in enumerate(dataloader):
        batch_x = batch_x.to(device)
        batch_y = batch_y.to(device)
        
        # ═══════════════════════════════════════════
        # KEY: Skip gradient sync for accumulation steps
        # Only sync on the last accumulation step
        # ═══════════════════════════════════════════
        is_accumulating = (step + 1) % accumulation_steps != 0
        
        # no_sync() skips the AllReduce — saves communication cost
        with model.no_sync() if is_accumulating else nullcontext():
            output = model(batch_x)
            loss = criterion(output, batch_y)
            # Scale loss by accumulation steps to get correct average
            loss = loss / accumulation_steps
            loss.backward()
        
        if not is_accumulating:
            # Gradients are now synced across GPUs
            torch.nn.utils.clip_grad_norm_(model.parameters(), max_norm=1.0)
            optimizer.step()
            optimizer.zero_grad()

# Helper for context manager
from contextlib import nullcontext
```

### DDP with Mixed Precision

```python
"""
DDP + AMP (Automatic Mixed Precision) for maximum throughput.
FP16 forward/backward + FP32 optimizer = 2x memory savings + faster.
"""

import torch
from torch.cuda.amp import GradScaler, autocast

def train_ddp_amp(model, dataloader, optimizer, criterion, device, epochs=10):
    scaler = GradScaler()  # Handles FP16 gradient scaling
    
    for epoch in range(epochs):
        model.train()
        
        for batch_x, batch_y in dataloader:
            batch_x = batch_x.to(device)
            batch_y = batch_y.to(device)
            
            optimizer.zero_grad()
            
            # Mixed precision forward pass
            with autocast():
                output = model(batch_x)
                loss = criterion(output, batch_y)
            
            # Scaled backward pass (prevents FP16 underflow)
            scaler.scale(loss).backward()
            
            # Unscale before clipping
            scaler.unscale_(optimizer)
            torch.nn.utils.clip_grad_norm_(model.parameters(), 1.0)
            
            # Optimizer step with scaler
            scaler.step(optimizer)
            scaler.update()
```

### DDP Checkpoint Save/Load

```python
import torch
import torch.distributed as dist

def save_checkpoint(model, optimizer, epoch, loss, filepath):
    """Save checkpoint — call ONLY from rank 0"""
    if dist.get_rank() == 0:
        checkpoint = {
            'epoch': epoch,
            'model_state_dict': model.module.state_dict(),  # Unwrap DDP
            'optimizer_state_dict': optimizer.state_dict(),
            'loss': loss,
        }
        torch.save(checkpoint, filepath)
        print(f"Checkpoint saved: {filepath}")
    
    # All ranks wait for rank 0 to finish saving
    dist.barrier()

def load_checkpoint(model, optimizer, filepath, device):
    """Load checkpoint — all ranks must call this"""
    # map_location ensures tensors go to the right GPU
    map_location = {'cuda:0': f'cuda:{dist.get_rank()}'}
    checkpoint = torch.load(filepath, map_location=map_location)
    
    model.module.load_state_dict(checkpoint['model_state_dict'])
    optimizer.load_state_dict(checkpoint['optimizer_state_dict'])
    
    return checkpoint['epoch'], checkpoint['loss']
```

### Launch Methods Comparison

```bash
# Method 1: torchrun (RECOMMENDED — PyTorch 1.10+)
# Handles process spawning, env vars, fault tolerance
torchrun --nproc_per_node=4 train.py --lr 0.001

# Method 2: torch.distributed.launch (DEPRECATED)
python -m torch.distributed.launch --nproc_per_node=4 train.py

# Method 3: Manual spawn (for custom control)
# (See mp.spawn example below)

# Method 4: SLURM (HPC clusters)
srun --ntasks=8 --gpus-per-task=1 python train.py
```

```python
# Manual spawn with mp.spawn
import torch.multiprocessing as mp

def train_worker(rank, world_size, args):
    """Each GPU runs this function"""
    os.environ['MASTER_ADDR'] = 'localhost'
    os.environ['MASTER_PORT'] = '29500'
    dist.init_process_group('nccl', rank=rank, world_size=world_size)
    
    # ... training code here ...
    
    dist.destroy_process_group()

if __name__ == "__main__":
    world_size = torch.cuda.device_count()
    mp.spawn(train_worker, args=(world_size, args), nprocs=world_size, join=True)
```

---

## 11.4 Fully Sharded Data Parallel (FSDP)

### What It Is
FSDP takes data parallelism to the extreme by **sharding (splitting) the model parameters, gradients, and optimizer states** across all GPUs. Each GPU only holds a fraction of the model at any time. When a layer needs to compute, FSDP gathers its parameters from all GPUs, computes, then discards them. It's like a relay race where each runner only carries part of the baton.

Think of DDP as every student having the complete textbook, while FSDP is each student having only one chapter — when they need a different chapter, they borrow it from a classmate.

### Why It Matters
- **Train models that don't fit on a single GPU** — even with data parallelism
- **Reduces memory by up to 8x** compared to DDP
- **Native PyTorch** — no external libraries needed (unlike DeepSpeed)
- **Used to train LLaMA, OPT, and other large models**

### Memory Comparison

For a model with $P$ parameters, $N$ GPUs:

| Strategy | Parameters | Gradients | Optimizer (Adam) | Total per GPU |
|----------|-----------|-----------|------------------|---------------|
| No parallelism | $P$ | $P$ | $2P$ | $4P$ |
| DDP | $P$ | $P$ | $2P$ | $4P$ |
| FSDP (ZeRO-3) | $P/N$ | $P/N$ | $2P/N$ | $4P/N$ |

> With 8 GPUs and FSDP, each GPU uses **8x less memory** than DDP!

### How FSDP Works

```
FSDP Lifecycle for one forward pass through Layer L:

1. Before forward:     2. AllGather:        3. Compute:         4. After forward:
   ┌─────────┐          ┌─────────┐         ┌─────────┐        ┌─────────┐
   │ GPU 0   │          │ GPU 0   │         │ GPU 0   │        │ GPU 0   │
   │ shard₀  │  ──▶     │ FULL    │  ──▶    │ output₀ │  ──▶   │ shard₀  │
   │ (1/4)   │          │ params  │         │         │        │ (1/4)   │
   ├─────────┤          ├─────────┤         ├─────────┤        ├─────────┤
   │ GPU 1   │          │ GPU 1   │         │ GPU 1   │        │ GPU 1   │
   │ shard₁  │  ──▶     │ FULL    │  ──▶    │ output₁ │  ──▶   │ shard₁  │
   │ (1/4)   │          │ params  │         │         │        │ (1/4)   │
   └─────────┘          └─────────┘         └─────────┘        └─────────┘
                    AllGather params      Forward pass       Free full params
                    from all GPUs                            (keep only shard)
```

### Complete FSDP Training Script

```python
"""
fsdp_train.py — Training with Fully Sharded Data Parallel

Launch: torchrun --nproc_per_node=4 fsdp_train.py
"""

import os
import functools
import torch
import torch.nn as nn
import torch.distributed as dist
from torch.distributed.fsdp import (
    FullyShardedDataParallel as FSDP,
    MixedPrecision,
    ShardingStrategy,
    CPUOffload,
)
from torch.distributed.fsdp.wrap import (
    size_based_auto_wrap_policy,
    transformer_auto_wrap_policy,
)
from torch.utils.data import DataLoader, TensorDataset
from torch.utils.data.distributed import DistributedSampler


class TransformerBlock(nn.Module):
    """A transformer-like block — FSDP shards at this granularity"""
    def __init__(self, dim=512, heads=8):
        super().__init__()
        self.attn = nn.MultiheadAttention(dim, heads, batch_first=True)
        self.norm1 = nn.LayerNorm(dim)
        self.ffn = nn.Sequential(
            nn.Linear(dim, dim * 4),
            nn.GELU(),
            nn.Linear(dim * 4, dim)
        )
        self.norm2 = nn.LayerNorm(dim)
    
    def forward(self, x):
        # Self-attention with residual
        attn_out, _ = self.attn(x, x, x)
        x = self.norm1(x + attn_out)
        # Feedforward with residual
        x = self.norm2(x + self.ffn(x))
        return x


class LargeTransformer(nn.Module):
    def __init__(self, vocab_size=50000, dim=512, num_layers=12, heads=8, num_classes=10):
        super().__init__()
        self.embedding = nn.Embedding(vocab_size, dim)
        self.layers = nn.ModuleList([
            TransformerBlock(dim, heads) for _ in range(num_layers)
        ])
        self.head = nn.Linear(dim, num_classes)
    
    def forward(self, x):
        x = self.embedding(x)  # [B, S] → [B, S, D]
        for layer in self.layers:
            x = layer(x)
        x = x.mean(dim=1)      # Global average pooling
        return self.head(x)


def train():
    dist.init_process_group("nccl")
    local_rank = int(os.environ["LOCAL_RANK"])
    rank = dist.get_rank()
    world_size = dist.get_world_size()
    torch.cuda.set_device(local_rank)
    
    # ═══════════════════════════════════════════
    # FSDP Configuration
    # ═══════════════════════════════════════════
    
    # Mixed precision policy — FP16 compute, FP32 parameters
    mixed_precision_policy = MixedPrecision(
        param_dtype=torch.float16,      # Parameters in FP16 during compute
        reduce_dtype=torch.float16,     # Gradient reduction in FP16
        buffer_dtype=torch.float16,     # Buffers in FP16
    )
    
    # Auto-wrap policy — which modules to shard separately
    # Option A: Size-based (shard modules with >100M parameters)
    size_policy = functools.partial(
        size_based_auto_wrap_policy,
        min_num_params=100_000_000
    )
    
    # Option B: Transformer-based (shard each TransformerBlock)
    # This is MUCH better for transformer models
    transformer_policy = functools.partial(
        transformer_auto_wrap_policy,
        transformer_layer_cls={TransformerBlock}  # Shard at this granularity
    )
    
    # ═══════════════════════════════════════════
    # Create model and wrap with FSDP
    # ═══════════════════════════════════════════
    model = LargeTransformer(num_layers=12)
    
    model = FSDP(
        model,
        auto_wrap_policy=transformer_policy,
        mixed_precision=mixed_precision_policy,
        sharding_strategy=ShardingStrategy.FULL_SHARD,  # ZeRO-3 equivalent
        device_id=local_rank,
        # cpu_offload=CPUOffload(offload_params=True),  # Offload to CPU if needed
    )
    
    if rank == 0:
        # Print FSDP-wrapped model structure
        print(model)
    
    # ═══════════════════════════════════════════
    # Dataset and DataLoader
    # ═══════════════════════════════════════════
    dataset = TensorDataset(
        torch.randint(0, 50000, (5000, 128)),  # [samples, seq_len]
        torch.randint(0, 10, (5000,))           # Labels
    )
    sampler = DistributedSampler(dataset, num_replicas=world_size, rank=rank)
    dataloader = DataLoader(dataset, batch_size=16, sampler=sampler, num_workers=2)
    
    # ═══════════════════════════════════════════
    # Optimizer — create AFTER FSDP wrapping
    # ═══════════════════════════════════════════
    optimizer = torch.optim.AdamW(model.parameters(), lr=1e-4, weight_decay=0.01)
    criterion = nn.CrossEntropyLoss()
    
    # ═══════════════════════════════════════════
    # Training loop
    # ═══════════════════════════════════════════
    for epoch in range(5):
        sampler.set_epoch(epoch)
        model.train()
        total_loss = 0
        
        for batch_x, batch_y in dataloader:
            batch_x = batch_x.to(local_rank)
            batch_y = batch_y.to(local_rank)
            
            output = model(batch_x)
            loss = criterion(output, batch_y)
            
            optimizer.zero_grad()
            loss.backward()
            
            # Gradient clipping with FSDP
            model.clip_grad_norm_(1.0)  # FSDP method, not torch.nn.utils
            
            optimizer.step()
            total_loss += loss.item()
        
        if rank == 0:
            print(f"Epoch {epoch+1}, Loss: {total_loss/len(dataloader):.4f}")
    
    # ═══════════════════════════════════════════
    # Save FSDP model checkpoint
    # ═══════════════════════════════════════════
    from torch.distributed.fsdp import FullStateDictConfig, StateDictType
    
    # Get full state dict on rank 0
    save_policy = FullStateDictConfig(offload_to_cpu=True, rank0_only=True)
    with FSDP.state_dict_type(model, StateDictType.FULL_STATE_DICT, save_policy):
        state_dict = model.state_dict()
        if rank == 0:
            torch.save(state_dict, "model_fsdp.pth")
            print("Model saved!")
    
    dist.barrier()
    dist.destroy_process_group()


if __name__ == "__main__":
    train()
```

### FSDP Sharding Strategies

```python
from torch.distributed.fsdp import ShardingStrategy

# Strategy 1: FULL_SHARD (= ZeRO-3)
# Shards parameters + gradients + optimizer states
# Maximum memory savings, highest communication
ShardingStrategy.FULL_SHARD

# Strategy 2: SHARD_GRAD_OP (= ZeRO-2)
# Shards gradients + optimizer states, keeps full parameters
# Less communication, more memory
ShardingStrategy.SHARD_GRAD_OP

# Strategy 3: NO_SHARD (= DDP)
# No sharding — equivalent to DDP
# Least communication, most memory
ShardingStrategy.NO_SHARD

# Strategy 4: HYBRID_SHARD
# Full shard within a node, replicate across nodes
# Best for multi-node with fast intra-node interconnect
ShardingStrategy.HYBRID_SHARD
```

| Strategy | Memory Savings | Communication | Best For |
|----------|---------------|---------------|----------|
| `FULL_SHARD` | Maximum (~8x) | Highest | Single-node, large models |
| `SHARD_GRAD_OP` | Medium (~4x) | Medium | Balance speed & memory |
| `HYBRID_SHARD` | High (within node) | Low (across nodes) | Multi-node clusters |
| `NO_SHARD` | None (= DDP) | Lowest | Model fits in GPU memory |

---

## 11.5 Model Parallelism

### What It Is
Model parallelism splits the **model itself** across GPUs — different layers live on different GPUs. It's like building a car on an assembly line: station 1 installs the engine, station 2 adds the body, station 3 paints it. Data flows through GPUs sequentially.

### Why It Matters
- Only option when a **single layer** is too large for one GPU (rare but happens with huge embedding tables)
- Foundation for pipeline parallelism (next section)
- Combined with data parallelism for maximum scale

### How It Works

```
Model Parallelism (Naive):

Data ──▶ [GPU 0: Layers 1-6] ──▶ [GPU 1: Layers 7-12] ──▶ Output

Problem: Only ONE GPU is active at a time!

Timeline:
GPU 0: [Forward█████]_________________[Backward█████]_________
GPU 1: _______________[Forward█████]_________________[Backward]

                    ⬆ GPU 1 idle        ⬆ GPU 0 idle
                    50% GPU utilization!
```

### Code Example

```python
import torch
import torch.nn as nn

class ModelParallelNet(nn.Module):
    """
    Split model across 2 GPUs manually.
    Layers 1-3 on GPU 0, Layers 4-6 on GPU 1.
    """
    def __init__(self):
        super().__init__()
        
        # First half on GPU 0
        self.block1 = nn.Sequential(
            nn.Linear(1024, 2048),
            nn.ReLU(),
            nn.Linear(2048, 2048),
            nn.ReLU(),
            nn.Linear(2048, 2048),
            nn.ReLU(),
        ).to('cuda:0')
        
        # Second half on GPU 1
        self.block2 = nn.Sequential(
            nn.Linear(2048, 2048),
            nn.ReLU(),
            nn.Linear(2048, 1024),
            nn.ReLU(),
            nn.Linear(1024, 10),
        ).to('cuda:1')
    
    def forward(self, x):
        # Input starts on GPU 0
        x = x.to('cuda:0')
        x = self.block1(x)
        
        # Transfer intermediate output to GPU 1
        x = x.to('cuda:1')  # Cross-GPU transfer (slow!)
        x = self.block2(x)
        return x

# Usage
model = ModelParallelNet()
input_data = torch.randn(32, 1024)

output = model(input_data)
# Loss must be computed on the same device as output
loss = nn.CrossEntropyLoss()(output, torch.randint(0, 10, (32,)).to('cuda:1'))
loss.backward()
```

### Improving Utilization with Micro-batching

```python
class PipelinedModelParallel(nn.Module):
    """
    Split batch into micro-batches to overlap computation.
    While GPU 1 processes micro-batch 1, GPU 0 processes micro-batch 2.
    """
    def __init__(self, split_size=8):
        super().__init__()
        self.split_size = split_size
        
        self.block1 = nn.Sequential(
            nn.Linear(1024, 2048), nn.ReLU(),
            nn.Linear(2048, 2048), nn.ReLU(),
        ).to('cuda:0')
        
        self.block2 = nn.Sequential(
            nn.Linear(2048, 1024), nn.ReLU(),
            nn.Linear(1024, 10),
        ).to('cuda:1')
    
    def forward(self, x):
        # Split batch into micro-batches
        splits = x.split(self.split_size, dim=0)
        
        outputs = []
        for split in splits:
            s = split.to('cuda:0')
            s = self.block1(s)
            s = s.to('cuda:1')
            s = self.block2(s)
            outputs.append(s)
        
        # Concatenate results
        return torch.cat(outputs, dim=0)

# With micro-batching:
# GPU 0: [MB1][MB2][MB3][MB4]
# GPU 1:      [MB1][MB2][MB3][MB4]
# Much better overlap!
```

---

## 11.6 Pipeline Parallelism

### What It Is
Pipeline parallelism is model parallelism done right. It splits the model into **stages**, splits data into **micro-batches**, and schedules them to maximize GPU utilization. Think of it as an assembly line in a factory — multiple products are being assembled at different stations simultaneously.

### Why It Matters
- Solves the GPU idle problem of naive model parallelism
- Reduces "pipeline bubble" (idle time) to a minimum
- Essential for training the largest models (100B+ parameters)

### Pipeline Schedules

```
GPipe Schedule (Fill-Drain):
──────────────────────────────────────────────────────
GPU 0: [F₁][F₂][F₃][F₄]________________[B₄][B₃][B₂][B₁]
GPU 1: ____[F₁][F₂][F₃][F₄]________[B₄][B₃][B₂][B₁]____
GPU 2: ________[F₁][F₂][F₃][F₄][B₄][B₃][B₂][B₁]________
                                    ^^^ pipeline bubble

1F1B Schedule (One Forward One Backward — better):
──────────────────────────────────────────────────────
GPU 0: [F₁][F₂][F₃][F₄][B₁][B₂][B₃][B₄]
GPU 1: ____[F₁][F₂][B₁][F₃][B₂][F₄][B₃][B₄]
GPU 2: ________[F₁][B₁][F₂][B₂][F₃][B₃][F₄][B₄]
                         ^^^ much smaller bubble!

F = Forward, B = Backward, number = micro-batch ID
```

### PyTorch Pipeline Parallelism

```python
"""
Pipeline parallelism using torch.distributed.pipelining (PyTorch 2.0+)
"""

import torch
import torch.nn as nn
from torch.distributed.pipelining import (
    pipe_split,
    pipeline,
    PipelineStage,
    ScheduleGPipe,
    Schedule1F1B,
)

class ModelWithSplits(nn.Module):
    """Define split points in the model"""
    def __init__(self):
        super().__init__()
        self.layer1 = nn.Linear(1024, 2048)
        self.layer2 = nn.Linear(2048, 2048)
        self.layer3 = nn.Linear(2048, 1024)
        self.layer4 = nn.Linear(1024, 10)
    
    def forward(self, x):
        x = torch.relu(self.layer1(x))
        x = torch.relu(self.layer2(x))
        pipe_split()  # ← Split point: layers before this → GPU 0, after → GPU 1
        x = torch.relu(self.layer3(x))
        x = self.layer4(x)
        return x

# Create pipeline
model = ModelWithSplits()
example_input = torch.randn(32, 1024)

# Trace and split into stages
pipe = pipeline(
    model,
    mb_args=(example_input,),
    chunks=4,  # Number of micro-batches
)

# Each rank handles one stage
# This is typically integrated with DDP for full parallelism
```

### Tensor Parallelism (for very large layers)

```python
"""
Tensor Parallelism — split INDIVIDUAL layers across GPUs.
Used for massive linear layers (e.g., LLM FFN layers with 4096→16384).
"""

import torch
import torch.nn as nn
import torch.distributed as dist

class ColumnParallelLinear(nn.Module):
    """
    Split linear layer columns across GPUs.
    Each GPU computes a portion of the output features.
    
    Full: Y = XW + b, W is [in, out]
    GPU 0: Y₀ = XW₀, W₀ is [in, out/N]
    GPU 1: Y₁ = XW₁, W₁ is [in, out/N]
    Result: Y = [Y₀ | Y₁]  (concatenate)
    """
    def __init__(self, in_features, out_features, world_size, rank):
        super().__init__()
        assert out_features % world_size == 0
        self.local_out = out_features // world_size
        self.linear = nn.Linear(in_features, self.local_out)
        self.rank = rank
        self.world_size = world_size
    
    def forward(self, x):
        local_output = self.linear(x)  # [B, local_out]
        
        # Gather outputs from all GPUs
        output_list = [torch.zeros_like(local_output) for _ in range(self.world_size)]
        dist.all_gather(output_list, local_output)
        return torch.cat(output_list, dim=-1)  # [B, out_features]


class RowParallelLinear(nn.Module):
    """
    Split linear layer rows across GPUs.
    Each GPU has all output features but only part of input.
    
    Full: Y = XW + b
    GPU 0: Y₀ = X₀W₀   (partial sum)
    GPU 1: Y₁ = X₁W₁   (partial sum)
    Result: Y = Y₀ + Y₁  (AllReduce sum)
    """
    def __init__(self, in_features, out_features, world_size, rank):
        super().__init__()
        assert in_features % world_size == 0
        self.local_in = in_features // world_size
        self.linear = nn.Linear(self.local_in, out_features)
        self.rank = rank
    
    def forward(self, x):
        # x is already split: [B, local_in]
        local_output = self.linear(x)  # [B, out_features]
        
        # Sum partial results across all GPUs
        dist.all_reduce(local_output, op=dist.ReduceOp.SUM)
        return local_output
```

---

## 11.7 DeepSpeed Integration

### What It Is
DeepSpeed is Microsoft's deep learning optimization library that integrates with PyTorch. It implements ZeRO (Zero Redundancy Optimizer) stages, mixed precision, gradient compression, and more. Think of it as a "turbocharger" for PyTorch — same engine, much more power.

### Why It Matters
- **ZeRO stages** — progressively more aggressive memory optimization
- **Offload to CPU/NVMe** — train models larger than total GPU memory
- **1-bit Adam** — 5x less communication for distributed training
- **Used to train** ChatGPT, BLOOM, and many other LLMs

### ZeRO Stages Explained

```
ZeRO Optimization Stages:
═════════════════════════════════════════════════════════════
               GPU Memory per GPU (for model with P params)
═════════════════════════════════════════════════════════════

No optimization:    [Parameters P][Gradients P][Optimizer 2P]  = 4P
                    ▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓

ZeRO Stage 1:      [Parameters P][Gradients P][Optimizer 2P/N]  
Shard optimizer     ▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓░░░░░░░░░░░░

ZeRO Stage 2:      [Parameters P][Grad P/N][Optimizer 2P/N]
+ Shard gradients   ▓▓▓▓▓▓▓▓▓▓▓▓░░░░░░░░░░░░░░░░░░░░░░░░

ZeRO Stage 3:      [Params P/N][Grad P/N][Optimizer 2P/N]
+ Shard parameters  ▓▓▓▓░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░

ZeRO-Infinity:      Even params offloaded to CPU/NVMe!
+ CPU/NVMe offload  ▓░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░

N = number of GPUs, ▓ = used, ░ = saved
```

### DeepSpeed Configuration

```json
// ds_config.json — DeepSpeed configuration file
{
    "train_batch_size": 256,
    "gradient_accumulation_steps": 4,
    "gradient_clipping": 1.0,
    
    "fp16": {
        "enabled": true,
        "loss_scale": 0,
        "loss_scale_window": 1000,
        "initial_scale_power": 16,
        "hysteresis": 2,
        "min_loss_scale": 1
    },
    
    "zero_optimization": {
        "stage": 2,
        "allgather_partitions": true,
        "allgather_bucket_size": 5e8,
        "overlap_comm": true,
        "reduce_scatter": true,
        "reduce_bucket_size": 5e8,
        "contiguous_gradients": true,
        
        "offload_optimizer": {
            "device": "cpu",
            "pin_memory": true
        }
    },
    
    "optimizer": {
        "type": "AdamW",
        "params": {
            "lr": 1e-4,
            "betas": [0.9, 0.999],
            "eps": 1e-8,
            "weight_decay": 0.01
        }
    },
    
    "scheduler": {
        "type": "WarmupLR",
        "params": {
            "warmup_min_lr": 0,
            "warmup_max_lr": 1e-4,
            "warmup_num_steps": 1000
        }
    },
    
    "activation_checkpointing": {
        "partition_activations": true,
        "contiguous_memory_optimization": true,
        "cpu_checkpointing": true
    }
}
```

### DeepSpeed Training Script

```python
"""
deepspeed_train.py — Training with DeepSpeed ZeRO

Launch: deepspeed --num_gpus=4 deepspeed_train.py --deepspeed_config ds_config.json
"""

import argparse
import torch
import torch.nn as nn
from torch.utils.data import DataLoader, TensorDataset
import deepspeed

class TransformerLM(nn.Module):
    def __init__(self, vocab_size=50000, dim=1024, layers=24, heads=16):
        super().__init__()
        self.embedding = nn.Embedding(vocab_size, dim)
        encoder_layer = nn.TransformerEncoderLayer(
            d_model=dim, nhead=heads, dim_feedforward=dim*4,
            batch_first=True, activation='gelu'
        )
        self.transformer = nn.TransformerEncoder(encoder_layer, num_layers=layers)
        self.lm_head = nn.Linear(dim, vocab_size, bias=False)
        
        # Weight tying — share embedding weights with output
        self.lm_head.weight = self.embedding.weight
    
    def forward(self, input_ids, labels=None):
        x = self.embedding(input_ids)
        x = self.transformer(x)
        logits = self.lm_head(x)
        
        loss = None
        if labels is not None:
            loss = nn.CrossEntropyLoss()(
                logits.view(-1, logits.size(-1)),
                labels.view(-1)
            )
        return loss, logits


def main():
    parser = argparse.ArgumentParser()
    parser.add_argument('--local_rank', type=int, default=-1)
    # DeepSpeed adds its own args
    parser = deepspeed.add_config_arguments(parser)
    args = parser.parse_args()
    
    # ═══════════════════════════════════════════
    # Create model (on CPU — DeepSpeed handles device placement)
    # ═══════════════════════════════════════════
    model = TransformerLM(vocab_size=50000, dim=1024, layers=24, heads=16)
    
    # ═══════════════════════════════════════════
    # Create dataset
    # ═══════════════════════════════════════════
    seq_len = 512
    dataset = TensorDataset(
        torch.randint(0, 50000, (5000, seq_len)),  # input_ids
        torch.randint(0, 50000, (5000, seq_len))   # labels (next token)
    )
    
    # ═══════════════════════════════════════════
    # Initialize DeepSpeed
    # ═══════════════════════════════════════════
    model_engine, optimizer, train_loader, lr_scheduler = deepspeed.initialize(
        args=args,
        model=model,
        training_data=dataset,
        model_parameters=model.parameters(),
    )
    # DeepSpeed creates optimizer, dataloader, and scheduler from config
    
    # ═══════════════════════════════════════════
    # Training loop — simplified by DeepSpeed
    # ═══════════════════════════════════════════
    for epoch in range(3):
        model_engine.train()
        total_loss = 0
        
        for step, (input_ids, labels) in enumerate(train_loader):
            input_ids = input_ids.to(model_engine.local_rank)
            labels = labels.to(model_engine.local_rank)
            
            # Forward
            loss, logits = model_engine(input_ids, labels)
            
            # Backward — DeepSpeed handles gradient scaling, accumulation, sync
            model_engine.backward(loss)
            
            # Step — DeepSpeed handles optimizer step, LR scheduling
            model_engine.step()
            
            total_loss += loss.item()
            
            if step % 50 == 0 and model_engine.local_rank == 0:
                print(f"Epoch {epoch}, Step {step}, Loss: {loss.item():.4f}")
        
        # Save checkpoint
        model_engine.save_checkpoint("checkpoints", tag=f"epoch_{epoch}")
    
    # Load checkpoint
    # model_engine.load_checkpoint("checkpoints", tag="epoch_2")


if __name__ == "__main__":
    main()
```

### ZeRO Stage 3 with CPU Offloading

```json
// ds_config_zero3.json — Maximum memory optimization
{
    "train_batch_size": 64,
    "gradient_accumulation_steps": 8,
    
    "bf16": {
        "enabled": true
    },
    
    "zero_optimization": {
        "stage": 3,
        
        "offload_param": {
            "device": "cpu",
            "pin_memory": true
        },
        "offload_optimizer": {
            "device": "cpu",
            "pin_memory": true
        },
        
        "overlap_comm": true,
        "contiguous_gradients": true,
        "sub_group_size": 1e9,
        "reduce_bucket_size": "auto",
        "stage3_prefetch_bucket_size": "auto",
        "stage3_param_persistence_threshold": "auto",
        "stage3_max_live_parameters": 1e9,
        "stage3_max_reuse_distance": 1e9,
        "stage3_gather_16bit_weights_on_model_save": true
    }
}
```

### DeepSpeed vs FSDP Comparison

| Feature | FSDP | DeepSpeed |
|---------|------|-----------|
| **Organization** | PyTorch native | Microsoft (external) |
| **Memory optimization** | Sharding (≈ ZeRO-3) | ZeRO Stages 1-3 + Infinity |
| **CPU offloading** | Basic support | Full (CPU + NVMe) |
| **Mixed precision** | FP16, BF16 | FP16, BF16, INT8 |
| **Communication** | AllGather/ReduceScatter | + 1-bit Adam, compression |
| **Activation checkpointing** | Manual | Automatic partitioning |
| **Ease of use** | PyTorch-native API | Config-file driven |
| **Community** | Growing | Large (HuggingFace default) |
| **Best for** | PyTorch-centric projects | LLM training, maximum scale |

---

## 11.8 Multi-Node Training

### What It Is
Multi-node training distributes training across **multiple physical machines** (nodes), each with multiple GPUs. This is how the largest models are trained — connecting dozens to thousands of GPUs across a data center.

### Network Topology

```
Data Center Network:

Node 0 (8x A100)              Node 1 (8x A100)
┌──────────────────┐          ┌──────────────────┐
│ GPU0 GPU1 GPU2 GPU3│◀═══════▶│ GPU0 GPU1 GPU2 GPU3│
│ GPU4 GPU5 GPU6 GPU7│  InfiniBand│ GPU4 GPU5 GPU6 GPU7│
│                    │  (200Gbps) │                    │
│ NVLink (600GB/s)  │          │ NVLink (600GB/s)  │
│ within node       │          │ within node       │
└──────────────────┘          └──────────────────┘
        │                              │
        └──────── Ethernet ────────────┘
                (10-100 Gbps)
```

### Multi-Node Setup with torchrun

```bash
# ═══════════════════════════════════════════
# Node 0 (master) — IP: 10.0.0.1
# ═══════════════════════════════════════════
torchrun \
    --nnodes=2 \
    --nproc_per_node=4 \
    --node_rank=0 \
    --master_addr=10.0.0.1 \
    --master_port=29500 \
    --rdzv_backend=c10d \
    train.py

# ═══════════════════════════════════════════
# Node 1 — IP: 10.0.0.2
# ═══════════════════════════════════════════
torchrun \
    --nnodes=2 \
    --nproc_per_node=4 \
    --node_rank=1 \
    --master_addr=10.0.0.1 \
    --master_port=29500 \
    --rdzv_backend=c10d \
    train.py
```

### Elastic Training (Fault Tolerant)

```bash
# Elastic training — handles node failures gracefully
# Min 2 nodes, max 4 nodes, current 3 nodes
torchrun \
    --nnodes=2:4 \
    --nproc_per_node=4 \
    --rdzv_backend=c10d \
    --rdzv_endpoint=10.0.0.1:29400 \
    --rdzv_id=my_job \
    --max_restarts=3 \
    train.py
```

### SLURM Job Script (HPC Clusters)

```bash
#!/bin/bash
#SBATCH --job-name=ddp_training
#SBATCH --nodes=4
#SBATCH --ntasks-per-node=1
#SBATCH --gpus-per-node=8
#SBATCH --cpus-per-task=32
#SBATCH --mem=256G
#SBATCH --time=24:00:00
#SBATCH --partition=gpu

# Get master address from first node
export MASTER_ADDR=$(scontrol show hostnames $SLURM_JOB_NODELIST | head -n 1)
export MASTER_PORT=29500

# Launch with srun
srun torchrun \
    --nnodes=$SLURM_NNODES \
    --nproc_per_node=8 \
    --rdzv_backend=c10d \
    --rdzv_endpoint=$MASTER_ADDR:$MASTER_PORT \
    train.py --epochs 100 --lr 1e-4
```

### Multi-Node Training Tips

```python
"""
Key adjustments for multi-node training
"""

import torch
import torch.distributed as dist

def setup_multi_node():
    """Setup with proper environment variables"""
    dist.init_process_group(
        backend='nccl',
        init_method='env://',  # Uses MASTER_ADDR and MASTER_PORT env vars
    )
    
    # IMPORTANT: Set NCCL environment variables for performance
    import os
    os.environ['NCCL_IB_DISABLE'] = '0'        # Enable InfiniBand
    os.environ['NCCL_NET_GDR_LEVEL'] = '5'      # GPU Direct RDMA
    os.environ['NCCL_SOCKET_IFNAME'] = 'eth0'   # Network interface
    os.environ['NCCL_DEBUG'] = 'INFO'            # Debug output


def scale_learning_rate(base_lr, world_size, warmup_steps=1000):
    """
    Linear scaling rule: LR scales with effective batch size.
    
    If base_lr works for batch_size B on 1 GPU,
    use base_lr * N for batch_size B on N GPUs
    (effective batch = B * N)
    
    ALWAYS use warmup with scaled LR to avoid training instability.
    """
    scaled_lr = base_lr * world_size
    
    # Use warmup scheduler
    from torch.optim.lr_scheduler import LambdaLR
    
    def warmup_lambda(step):
        if step < warmup_steps:
            return step / warmup_steps
        return 1.0
    
    # Apply to optimizer
    optimizer = torch.optim.AdamW(model.parameters(), lr=scaled_lr)
    scheduler = LambdaLR(optimizer, lr_lambda=warmup_lambda)
    return optimizer, scheduler
```

---

## 11.9 Debugging and Profiling Distributed Training

### What It Is
Debugging distributed training is notoriously hard — multiple processes, GPU memory issues, deadlocks, and communication errors. These tools and techniques help you find and fix problems.

### Debugging Environment Variables

```bash
# NCCL debugging
export NCCL_DEBUG=INFO          # Show NCCL communication info
export NCCL_DEBUG=WARN          # Only warnings and errors
export NCCL_DEBUG_SUBSYS=ALL    # All subsystems

# PyTorch distributed debugging
export TORCH_DISTRIBUTED_DEBUG=DETAIL  # Detailed logging
export TORCH_CPP_LOG_LEVEL=INFO        # C++ backend logs

# CUDA debugging
export CUDA_LAUNCH_BLOCKING=1   # Synchronous CUDA calls (for error location)
```

### Detecting Hangs and Deadlocks

```python
import torch.distributed as dist
import signal
import traceback

def timeout_handler(signum, frame):
    """Kill process if it hangs too long"""
    print(f"RANK {dist.get_rank()} TIMED OUT!")
    traceback.print_stack(frame)
    raise TimeoutError("Training step took too long")

# Set 5-minute timeout per step
signal.signal(signal.SIGALRM, timeout_handler)

for batch in dataloader:
    signal.alarm(300)  # 5 minutes timeout
    
    output = model(batch)
    loss.backward()
    optimizer.step()
    
    signal.alarm(0)  # Cancel timeout after successful step
```

### Memory Profiling

```python
import torch

def print_gpu_memory(tag=""):
    """Print current GPU memory usage"""
    if torch.cuda.is_available():
        allocated = torch.cuda.memory_allocated() / 1024**3
        reserved = torch.cuda.memory_reserved() / 1024**3
        max_allocated = torch.cuda.max_memory_allocated() / 1024**3
        print(f"[{tag}] GPU Memory: "
              f"Allocated={allocated:.2f}GB, "
              f"Reserved={reserved:.2f}GB, "
              f"Peak={max_allocated:.2f}GB")

# Use throughout training
print_gpu_memory("Before model creation")
model = LargeModel().cuda()
print_gpu_memory("After model creation")

output = model(input_data)
print_gpu_memory("After forward pass")

loss.backward()
print_gpu_memory("After backward pass")

# Reset peak stats
torch.cuda.reset_peak_memory_stats()
```

### PyTorch Profiler for Distributed

```python
import torch
from torch.profiler import profile, record_function, ProfilerActivity

# Profile one training step
with profile(
    activities=[ProfilerActivity.CPU, ProfilerActivity.CUDA],
    record_shapes=True,
    profile_memory=True,
    with_stack=True,
    schedule=torch.profiler.schedule(
        wait=1,     # Skip first step
        warmup=1,   # Warmup step
        active=3,   # Profile 3 steps
        repeat=1
    ),
    on_trace_ready=torch.profiler.tensorboard_trace_handler('./profiler_logs'),
) as prof:
    for step, (batch_x, batch_y) in enumerate(dataloader):
        if step >= 5:
            break
        
        with record_function("forward"):
            output = model(batch_x.cuda())
        
        with record_function("loss"):
            loss = criterion(output, batch_y.cuda())
        
        with record_function("backward"):
            optimizer.zero_grad()
            loss.backward()
        
        with record_function("optimizer"):
            optimizer.step()
        
        prof.step()

# Print summary
print(prof.key_averages().table(sort_by="cuda_time_total", row_limit=20))

# View in TensorBoard:
# tensorboard --logdir=./profiler_logs
```

### Communication Profiling

```python
import torch.distributed as dist
import time

def benchmark_allreduce(tensor_size_mb=100, device='cuda'):
    """Benchmark AllReduce communication speed"""
    size = int(tensor_size_mb * 1024 * 1024 / 4)  # Number of float32 elements
    tensor = torch.randn(size, device=device)
    
    # Warmup
    for _ in range(5):
        dist.all_reduce(tensor)
    torch.cuda.synchronize()
    
    # Benchmark
    start = time.perf_counter()
    num_iters = 20
    for _ in range(num_iters):
        dist.all_reduce(tensor)
    torch.cuda.synchronize()
    elapsed = time.perf_counter() - start
    
    # Calculate bandwidth
    # AllReduce transfers 2*(N-1)/N * data_size bytes
    world_size = dist.get_world_size()
    data_bytes = tensor.numel() * 4
    algo_bandwidth = data_bytes * num_iters / elapsed / 1e9  # GB/s
    bus_bandwidth = algo_bandwidth * 2 * (world_size - 1) / world_size
    
    if dist.get_rank() == 0:
        print(f"AllReduce {tensor_size_mb}MB: {elapsed/num_iters*1000:.1f}ms, "
              f"Bus BW: {bus_bandwidth:.1f} GB/s")
```

---

## 11.10 Common Mistakes

### Mistake 1: Forgetting `sampler.set_epoch(epoch)`
```python
# ❌ WRONG — all epochs use same data order (no real shuffling)
for epoch in range(10):
    for batch in dataloader:
        train_step(batch)

# ✅ CORRECT — different shuffle each epoch
for epoch in range(10):
    sampler.set_epoch(epoch)  # Seeds the shuffle with epoch number
    for batch in dataloader:
        train_step(batch)
```

### Mistake 2: Using `shuffle=True` with `DistributedSampler`
```python
# ❌ WRONG — shuffle and DistributedSampler conflict
loader = DataLoader(dataset, batch_size=32, shuffle=True, sampler=sampler)
# RuntimeError: sampler and shuffle are mutually exclusive

# ✅ CORRECT — shuffle is handled by the sampler
loader = DataLoader(dataset, batch_size=32, sampler=sampler)
# Control shuffle via DistributedSampler(dataset, shuffle=True)
```

### Mistake 3: All Ranks Printing/Logging/Saving
```python
# ❌ WRONG — 8 GPUs = 8 copies of every print
print(f"Loss: {loss.item()}")
torch.save(model.state_dict(), "model.pth")
wandb.log({"loss": loss})

# ✅ CORRECT — only rank 0 handles I/O
if dist.get_rank() == 0:
    print(f"Loss: {loss.item()}")
    torch.save(model.module.state_dict(), "model.pth")
    wandb.log({"loss": loss})
```

### Mistake 4: Not Wrapping Model Before Creating Optimizer
```python
# ❌ WRONG — optimizer references pre-DDP parameters
optimizer = torch.optim.Adam(model.parameters(), lr=1e-3)
model = DDP(model, device_ids=[local_rank])
# Optimizer points to wrong parameter objects!

# ✅ CORRECT — create optimizer AFTER DDP wrapping
model = DDP(model, device_ids=[local_rank])
optimizer = torch.optim.Adam(model.parameters(), lr=1e-3)
```

### Mistake 5: Saving `model` instead of `model.module`
```python
# ❌ WRONG — saves DDP wrapper, can't load without DDP
torch.save(model.state_dict(), "model.pth")
# Keys look like: "module.fc1.weight" instead of "fc1.weight"

# ✅ CORRECT — save the unwrapped model
torch.save(model.module.state_dict(), "model.pth")
```

### Mistake 6: Not Scaling Learning Rate
```python
# ❌ WRONG — same LR for 1 GPU and 8 GPUs
# Effective batch is 8x larger but LR stays the same → underfitting
optimizer = torch.optim.SGD(model.parameters(), lr=0.1)

# ✅ CORRECT — scale LR with world size + warmup
base_lr = 0.1
scaled_lr = base_lr * dist.get_world_size()
optimizer = torch.optim.SGD(model.parameters(), lr=scaled_lr)
# Add learning rate warmup for first 1000 steps!
```

### Mistake 7: Deadlocks from Conditional Code
```python
# ❌ WRONG — only some ranks execute this → other ranks wait forever
if dist.get_rank() == 0:
    dist.barrier()  # Only rank 0 reaches here → DEADLOCK

# ✅ CORRECT — ALL ranks must call collective operations
dist.barrier()  # All ranks call this
if dist.get_rank() == 0:
    # Do rank-0 only work (non-collective operations)
    save_model()
```

---

## 11.11 Interview Questions

### Q1: What's the difference between DataParallel and DistributedDataParallel?
**A:** `DataParallel` (DP) uses multi-threading within one process — it has a GPU 0 bottleneck (gathers all gradients there), suffers from the GIL, and only works on one machine. `DistributedDataParallel` (DDP) launches one process per GPU, uses NCCL ring AllReduce for efficient gradient sync (no single bottleneck), works across multiple machines, and is 2-3x faster. Always use DDP for anything beyond quick prototyping.

### Q2: Explain the AllReduce algorithm and why Ring AllReduce is efficient.
**A:** AllReduce computes a reduction (e.g., sum) of tensors across all processes and distributes the result to all. Ring AllReduce arranges GPUs in a logical ring — each GPU sends data to the next and receives from the previous. In 2 phases (scatter-reduce + allgather), each GPU transfers $2(N-1)/N$ of the data, which is nearly optimal regardless of the number of GPUs. The bandwidth cost is constant per GPU, making it highly scalable.

### Q3: What are ZeRO stages 1, 2, and 3?
**A:** ZeRO (Zero Redundancy Optimizer) progressively partitions training state across GPUs: **Stage 1** partitions optimizer states (e.g., Adam's momentum and variance) — saves ~4x memory. **Stage 2** adds gradient partitioning — saves ~8x. **Stage 3** also partitions model parameters — saves up to ~$N$x total where $N$ is GPU count. Each stage increases communication but reduces per-GPU memory, enabling larger models.

### Q4: When would you use FSDP vs DeepSpeed?
**A:** Use **FSDP** when: staying pure PyTorch (no external dependencies), moderate-scale training, or when team prefers PyTorch-native APIs. Use **DeepSpeed** when: training very large LLMs (100B+ parameters), need CPU/NVMe offloading (ZeRO-Infinity), need advanced features like 1-bit Adam, or using HuggingFace ecosystem (built-in DeepSpeed integration).

### Q5: How do you handle batch normalization in distributed training?
**A:** Standard BatchNorm computes statistics per-GPU, which is incorrect with small per-GPU batch sizes (different statistics on each GPU). Solutions: (1) Use `SyncBatchNorm` (`nn.SyncBatchNorm.convert_sync_batchnorm(model)`) to sync stats across GPUs. (2) Use `LayerNorm` or `GroupNorm` which are independent of batch size. (3) Use large enough per-GPU batch size that local statistics are representative.

### Q6: How does gradient accumulation work with DDP?
**A:** Use `model.no_sync()` context manager during accumulation steps to skip the AllReduce. Only on the final accumulation step (without `no_sync()`), gradients are synced across GPUs. This reduces communication from every micro-step to every $K$ micro-steps, improving throughput. Remember to divide the loss by the number of accumulation steps.

### Q7: What is pipeline parallelism and how does the 1F1B schedule improve over GPipe?
**A:** Pipeline parallelism splits the model into stages across GPUs and sends micro-batches through sequentially. **GPipe** does all forward passes first, then all backward passes — creating a large "pipeline bubble" where GPUs are idle. **1F1B** (One Forward One Backward) interleaves forward and backward passes: after the pipeline fills, each GPU alternates between forward and backward of different micro-batches. This reduces the bubble from $O(P)$ to $O(P-1)$ micro-batches ($P$ = pipeline stages) and also reduces peak memory since activations are freed sooner.

### Q8: How do you debug a distributed training hang?
**A:** (1) Set `NCCL_DEBUG=INFO` and `TORCH_DISTRIBUTED_DEBUG=DETAIL` for logging. (2) Set `CUDA_LAUNCH_BLOCKING=1` to make CUDA ops synchronous. (3) Check that ALL ranks reach every collective operation (barrier, allreduce). (4) Use `torch.distributed.monitored_barrier()` with a timeout to identify which rank is stuck. (5) Add timeouts using `signal.alarm()`. (6) Check network connectivity between nodes. (7) Verify NCCL version compatibility across nodes.

---

## 11.12 Quick Reference

### Essential Commands

| Task | Command |
|------|---------|
| Launch DDP (1 node, 4 GPU) | `torchrun --nproc_per_node=4 train.py` |
| Launch DDP (2 nodes) | `torchrun --nnodes=2 --node_rank=0 --master_addr=IP train.py` |
| Launch DeepSpeed | `deepspeed --num_gpus=4 train.py --deepspeed_config ds.json` |
| Convert SyncBatchNorm | `nn.SyncBatchNorm.convert_sync_batchnorm(model)` |
| NCCL debug | `export NCCL_DEBUG=INFO` |

### DDP Checklist

```
□ dist.init_process_group("nccl")
□ torch.cuda.set_device(local_rank)
□ model = model.to(device)
□ model = DDP(model, device_ids=[local_rank])
□ optimizer created AFTER DDP wrapping
□ DistributedSampler (NOT shuffle=True)
□ sampler.set_epoch(epoch) each epoch
□ Save model.module.state_dict() (not model.state_dict())
□ Only rank 0 does logging/saving
□ dist.barrier() before cleanup
□ dist.destroy_process_group() at the end
```

### Parallelism Decision Matrix

```
How to choose your parallelism strategy:
─────────────────────────────────────────────────────
Model fits on 1 GPU?
├── YES: Use DDP (data parallelism)
│   └── Need more memory? → Gradient accumulation + AMP
│
└── NO: How big is the model?
    ├── Fits on 1 node (multi-GPU): Use FSDP
    │   └── Sharding strategy?
    │       ├── Tight on memory → FULL_SHARD
    │       └── Prefer speed → SHARD_GRAD_OP
    │
    └── Needs multiple nodes:
        ├── Up to ~30B params → FSDP or DeepSpeed ZeRO-3
        ├── 30B-200B params → DeepSpeed ZeRO-3 + offload
        └── 200B+ params → Tensor + Pipeline + Data parallel
            (Megatron-LM / custom hybrid)
```

### Performance Optimization Checklist

| Optimization | Impact | Effort |
|-------------|--------|--------|
| Mixed precision (FP16/BF16) | 2x memory, 1.5-3x speed | Low (3 lines) |
| Gradient accumulation | Larger effective batch | Low |
| `torch.compile()` | 1.5-3x speed | Low (1 line) |
| `pin_memory=True` in DataLoader | 10-20% data loading speed | Trivial |
| `num_workers > 0` in DataLoader | Overlap data loading | Trivial |
| FSDP/ZeRO sharding | 4-8x memory reduction | Medium |
| CPU/NVMe offloading | Train beyond GPU memory | Medium |
| Activation checkpointing | 2-5x memory at ~20% speed cost | Medium |
| Tensor parallelism | Scale individual layers | High |
| Pipeline parallelism | Scale model depth | High |
| Custom CUDA kernels | Maximum performance | Very High |

### Memory Estimation Formula

For training a model with $P$ parameters using Adam optimizer:

$$\text{Memory (bytes)} = P \times (\underbrace{4}_{\text{params FP32}} + \underbrace{4}_{\text{gradients}} + \underbrace{4}_{\text{momentum}} + \underbrace{4}_{\text{variance}}) + \text{Activations}$$

$$\text{Memory (GB)} \approx \frac{16P}{10^9} + \text{Activations}$$

Example: 7B parameter model
- Parameters + Optimizer: $\frac{16 \times 7 \times 10^9}{10^9} = 112$ GB
- With FSDP on 8 GPUs: $112 / 8 = 14$ GB per GPU + activations
- With FP16/BF16: halve the parameter and gradient memory

> **Golden Rule:** When in doubt, start with DDP + mixed precision. Only add complexity (FSDP, DeepSpeed, pipeline parallelism) when you've confirmed that simpler approaches aren't enough.
