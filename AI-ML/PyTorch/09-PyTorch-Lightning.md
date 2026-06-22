# Chapter 09: PyTorch Lightning

## Table of Contents
- [What is PyTorch Lightning?](#what-is-pytorch-lightning)
- [Why Lightning Matters](#why-lightning-matters)
- [LightningModule — The Core Abstraction](#lightningmodule--the-core-abstraction)
- [Trainer — The Engine](#trainer--the-engine)
- [LightningDataModule](#lightningdatamodule)
- [Callbacks](#callbacks)
- [Logging and Experiment Tracking](#logging-and-experiment-tracking)
- [Checkpointing](#checkpointing)
- [Multi-GPU and Distributed Training](#multi-gpu-and-distributed-training)
- [Advanced Lightning Patterns](#advanced-lightning-patterns)
- [Common Mistakes](#common-mistakes)
- [Interview Questions](#interview-questions)
- [Quick Reference](#quick-reference)

---

## What is PyTorch Lightning?

**Simple Explanation:** PyTorch Lightning is like an autopilot system for deep learning. You still design the plane (your model and logic), but Lightning handles the boring-but-critical stuff: GPU management, distributed training, logging, checkpointing, mixed precision, and the training loop boilerplate.

**Formal Definition:** PyTorch Lightning is a lightweight wrapper around PyTorch that decouples the *science* (model, loss, optimizer) from the *engineering* (training loop, GPU management, distributed training). It enforces best practices without limiting flexibility.

```
┌────────────────────────────────────────────────────────────────┐
│              PURE PYTORCH vs PYTORCH LIGHTNING                   │
├───────────────────────────┬────────────────────────────────────┤
│     Pure PyTorch          │      PyTorch Lightning              │
├───────────────────────────┼────────────────────────────────────┤
│                           │                                    │
│  # YOUR code (~30%)      │  # YOUR code (LightningModule)     │
│  model = MyModel()       │  - __init__                        │
│  criterion = ...         │  - forward                         │
│  optimizer = ...         │  - training_step                   │
│                           │  - configure_optimizers             │
│  # BOILERPLATE (~70%)    │                                    │
│  for epoch in range(N):  │  # LIGHTNING handles (~70%)        │
│    model.train()         │  - Training loop                   │
│    for batch in loader:  │  - Validation loop                 │
│      x = x.to(device)   │  - GPU placement                   │
│      optimizer.zero_grad │  - Mixed precision                 │
│      output = model(x)  │  - Distributed training            │
│      loss = criterion()  │  - Logging                         │
│      loss.backward()     │  - Checkpointing                   │
│      optimizer.step()    │  - Early stopping                  │
│    model.eval()          │  - Progress bars                   │
│    with no_grad():       │  - Gradient clipping               │
│      for batch in val:   │  - Profiling                       │
│        ...               │  - Reproducibility                 │
└───────────────────────────┴────────────────────────────────────┘
```

### Installation

```python
# Install PyTorch Lightning
# pip install lightning

# Or the older package name (both work)
# pip install pytorch-lightning
```

---

## Why Lightning Matters

### Real-World Impact

| Aspect | Pure PyTorch | PyTorch Lightning |
|--------|-------------|-------------------|
| Training loop code | 50-200 lines | 0 lines (handled by Trainer) |
| Multi-GPU support | Hours of engineering | 1 flag: `devices=4` |
| Mixed precision | 10+ lines of code | 1 flag: `precision="16-mixed"` |
| Logging | Manual integration | Built-in (TensorBoard, W&B, etc.) |
| Checkpointing | Manual save/load | Automatic |
| Reproducibility | Easy to forget seeds | Built-in seed_everything |
| Code organization | Varies wildly | Standardized structure |
| Bug surface area | Large (custom loops) | Small (battle-tested loops) |

### When to Use Lightning

- **Production projects** — standardized code, easy handoffs
- **Research papers** — reproducible results, quick experiments
- **Team projects** — everyone reads the same structure
- **Scaling** — go from 1 GPU to 100 GPUs with minimal changes

### When NOT to Use Lightning

- Learning PyTorch fundamentals (write loops by hand first!)
- Extremely custom training logic that doesn't fit the step paradigm
- Simple scripts where the overhead isn't worth it

---

## LightningModule — The Core Abstraction

### What It Is

`LightningModule` is a drop-in replacement for `nn.Module` that organizes your code into well-defined methods. Think of it as a contract: "implement these methods, and Lightning handles the rest."

### Anatomy of a LightningModule

```python
import torch
import torch.nn as nn
import torch.nn.functional as F
import lightning as L  # Modern import (lightning >= 2.0)
# Alternative: import pytorch_lightning as pl

class ImageClassifier(L.LightningModule):
    """
    A complete image classifier with training, validation, and testing.
    
    LightningModule lifecycle:
    ┌───────────────────────────────────────────────────┐
    │  __init__          → Define model, loss, metrics  │
    │  setup()           → Called on every GPU           │
    │  configure_optimizers() → Return optimizer + scheduler │
    │  train_dataloader() → Return training dataloader   │
    │  training_step()   → Single training batch logic   │
    │  validation_step() → Single validation batch logic │
    │  test_step()       → Single test batch logic       │
    │  predict_step()    → Single prediction batch logic │
    │  on_*_epoch_end()  → Aggregate epoch metrics       │
    └───────────────────────────────────────────────────┘
    """
    
    def __init__(self, num_classes=10, learning_rate=1e-3):
        super().__init__()
        
        # save_hyperparameters() stores all __init__ args in self.hparams
        # They are automatically saved in checkpoints!
        self.save_hyperparameters()
        
        # Define the model architecture
        self.features = nn.Sequential(
            nn.Conv2d(3, 32, 3, padding=1),
            nn.BatchNorm2d(32),
            nn.ReLU(),
            nn.MaxPool2d(2),
            nn.Conv2d(32, 64, 3, padding=1),
            nn.BatchNorm2d(64),
            nn.ReLU(),
            nn.MaxPool2d(2),
            nn.Conv2d(64, 128, 3, padding=1),
            nn.BatchNorm2d(128),
            nn.ReLU(),
            nn.AdaptiveAvgPool2d(1),
        )
        
        self.classifier = nn.Sequential(
            nn.Flatten(),
            nn.Linear(128, 256),
            nn.ReLU(),
            nn.Dropout(0.3),
            nn.Linear(256, num_classes),
        )
        
        # Metrics (manual tracking)
        self.train_losses = []
        self.val_losses = []
    
    def forward(self, x):
        """
        Forward pass — used for inference/prediction.
        Keep this clean. Don't put loss computation here.
        """
        x = self.features(x)
        x = self.classifier(x)
        return x
    
    # ================================================================
    # TRAINING
    # ================================================================
    
    def training_step(self, batch, batch_idx):
        """
        Process a single training batch.
        Called by Lightning's training loop.
        
        Args:
            batch: Output from your DataLoader (images, labels)
            batch_idx: Index of this batch
        
        Returns:
            loss: The loss tensor (Lightning handles backward + optimizer)
        """
        images, labels = batch
        logits = self(images)  # Calls forward()
        loss = F.cross_entropy(logits, labels)
        
        # Calculate accuracy
        preds = logits.argmax(dim=1)
        acc = (preds == labels).float().mean()
        
        # Log metrics — Lightning handles aggregation!
        # on_step=True: log every batch
        # on_epoch=True: log the epoch average
        # prog_bar=True: show in progress bar
        self.log('train_loss', loss, on_step=True, on_epoch=True, prog_bar=True)
        self.log('train_acc', acc, on_step=False, on_epoch=True, prog_bar=True)
        
        return loss  # Lightning calls loss.backward() for you
    
    # ================================================================
    # VALIDATION
    # ================================================================
    
    def validation_step(self, batch, batch_idx):
        """
        Process a single validation batch.
        Lightning automatically:
        - Sets model.eval()
        - Disables gradients (torch.no_grad)
        - Disables dropout and uses running stats for BatchNorm
        """
        images, labels = batch
        logits = self(images)
        loss = F.cross_entropy(logits, labels)
        
        preds = logits.argmax(dim=1)
        acc = (preds == labels).float().mean()
        
        self.log('val_loss', loss, on_epoch=True, prog_bar=True)
        self.log('val_acc', acc, on_epoch=True, prog_bar=True)
    
    # ================================================================
    # TESTING
    # ================================================================
    
    def test_step(self, batch, batch_idx):
        """Same as validation_step but for the test set."""
        images, labels = batch
        logits = self(images)
        loss = F.cross_entropy(logits, labels)
        
        preds = logits.argmax(dim=1)
        acc = (preds == labels).float().mean()
        
        self.log('test_loss', loss, on_epoch=True)
        self.log('test_acc', acc, on_epoch=True)
    
    # ================================================================
    # PREDICTION
    # ================================================================
    
    def predict_step(self, batch, batch_idx):
        """For running predictions on new data."""
        images = batch[0] if isinstance(batch, (list, tuple)) else batch
        logits = self(images)
        return torch.softmax(logits, dim=1)
    
    # ================================================================
    # OPTIMIZER CONFIGURATION
    # ================================================================
    
    def configure_optimizers(self):
        """
        Define optimizer(s) and scheduler(s).
        
        Returns one of:
        - Single optimizer
        - Tuple of (optimizer_list, scheduler_list)
        - Dict with 'optimizer' and optional 'lr_scheduler'
        """
        optimizer = torch.optim.AdamW(
            self.parameters(),
            lr=self.hparams.learning_rate,
            weight_decay=0.01
        )
        
        scheduler = torch.optim.lr_scheduler.CosineAnnealingLR(
            optimizer,
            T_max=self.trainer.max_epochs,  # Access trainer config
            eta_min=1e-6
        )
        
        return {
            'optimizer': optimizer,
            'lr_scheduler': {
                'scheduler': scheduler,
                'interval': 'epoch',   # 'step' or 'epoch'
                'frequency': 1,
                'monitor': 'val_loss', # For ReduceLROnPlateau
            }
        }

# ============================================================
# USAGE
# ============================================================

model = ImageClassifier(num_classes=10, learning_rate=1e-3)

# Access saved hyperparameters
print(model.hparams)
# ImageClassifier(num_classes=10, learning_rate=0.001)
```

### Transfer Learning with LightningModule

```python
import torch
import torch.nn as nn
import torch.nn.functional as F
import lightning as L
from torchvision import models
from torchvision.models import ResNet50_Weights

class TransferLearningModel(L.LightningModule):
    """
    Fine-tuning a pretrained model with Lightning.
    Shows discriminative learning rates and gradual unfreezing.
    """
    
    def __init__(self, num_classes=10, lr_backbone=1e-5, lr_head=1e-3):
        super().__init__()
        self.save_hyperparameters()
        
        # Load pretrained backbone
        backbone = models.resnet50(weights=ResNet50_Weights.IMAGENET1K_V2)
        
        # Remove the original classifier
        self.backbone = nn.Sequential(*list(backbone.children())[:-1])
        
        # Freeze backbone initially
        for param in self.backbone.parameters():
            param.requires_grad = False
        
        # New classifier head
        self.head = nn.Sequential(
            nn.Flatten(),
            nn.Linear(2048, 512),
            nn.ReLU(),
            nn.Dropout(0.3),
            nn.Linear(512, num_classes)
        )
    
    def forward(self, x):
        features = self.backbone(x)
        return self.head(features)
    
    def training_step(self, batch, batch_idx):
        images, labels = batch
        logits = self(images)
        loss = F.cross_entropy(logits, labels)
        acc = (logits.argmax(1) == labels).float().mean()
        self.log_dict({'train_loss': loss, 'train_acc': acc}, prog_bar=True)
        return loss
    
    def validation_step(self, batch, batch_idx):
        images, labels = batch
        logits = self(images)
        loss = F.cross_entropy(logits, labels)
        acc = (logits.argmax(1) == labels).float().mean()
        self.log_dict({'val_loss': loss, 'val_acc': acc}, prog_bar=True)
    
    def configure_optimizers(self):
        # Discriminative learning rates
        optimizer = torch.optim.Adam([
            {'params': self.backbone.parameters(), 'lr': self.hparams.lr_backbone},
            {'params': self.head.parameters(), 'lr': self.hparams.lr_head},
        ])
        scheduler = torch.optim.lr_scheduler.ReduceLROnPlateau(
            optimizer, mode='min', factor=0.5, patience=3
        )
        return {
            'optimizer': optimizer,
            'lr_scheduler': {
                'scheduler': scheduler,
                'monitor': 'val_loss',
            }
        }
    
    def on_train_epoch_start(self):
        """Gradually unfreeze backbone layers."""
        if self.current_epoch == 5:
            # Unfreeze last layer group after 5 epochs
            for param in self.backbone[-1].parameters():
                param.requires_grad = True
            print("Unfroze backbone[-1]")
        
        elif self.current_epoch == 10:
            # Unfreeze everything
            for param in self.backbone.parameters():
                param.requires_grad = True
            print("Unfroze entire backbone")
```

---

## Trainer — The Engine

### What It Is

The `Trainer` is Lightning's training engine. It runs the training loop, handles GPUs, applies callbacks, logs metrics, and saves checkpoints — all configured through simple flags.

### Basic Usage

```python
import lightning as L

model = ImageClassifier(num_classes=10, learning_rate=1e-3)

# ============================================================
# BASIC TRAINER
# ============================================================
trainer = L.Trainer(
    max_epochs=20,
    accelerator='auto',   # Automatically detect GPU/CPU/TPU
    devices='auto',        # Use all available devices
)

# Train
trainer.fit(model, train_dataloaders=train_loader, val_dataloaders=val_loader)

# Test
trainer.test(model, dataloaders=test_loader)

# Predict
predictions = trainer.predict(model, dataloaders=test_loader)
```

### Trainer Flags — The Complete Guide

```python
trainer = L.Trainer(
    # ====== DURATION ======
    max_epochs=100,            # Maximum number of epochs
    min_epochs=10,             # Minimum epochs (override early stopping)
    max_steps=-1,              # Max training steps (-1 = no limit)
    max_time="00:12:00:00",    # Max training time (DD:HH:MM:SS)
    
    # ====== HARDWARE ======
    accelerator='gpu',         # 'cpu', 'gpu', 'tpu', 'auto'
    devices=1,                 # Number of GPUs: 1, 2, [0,1], 'auto'
    strategy='auto',           # 'ddp', 'fsdp', 'deepspeed', 'auto'
    precision='16-mixed',      # '32', '16-mixed', 'bf16-mixed'
    
    # ====== TRAINING TWEAKS ======
    gradient_clip_val=1.0,     # Gradient clipping by norm
    gradient_clip_algorithm='norm',  # 'norm' or 'value'
    accumulate_grad_batches=4, # Gradient accumulation
    
    # ====== VALIDATION ======
    val_check_interval=1.0,    # Validate every N epochs (1.0) or steps (int)
    check_val_every_n_epoch=1, # Validate every N epochs
    num_sanity_val_steps=2,    # Run N validation batches before training
    
    # ====== LOGGING ======
    logger=True,               # Enable default TensorBoard logger
    log_every_n_steps=50,      # Log every N training steps
    enable_progress_bar=True,  # Show progress bar
    
    # ====== CHECKPOINTING ======
    enable_checkpointing=True, # Auto-save checkpoints
    default_root_dir='./logs', # Where to save logs and checkpoints
    
    # ====== DEBUGGING ======
    fast_dev_run=False,        # Run 1 train + 1 val batch (for debugging)
    overfit_batches=0,         # Overfit on N batches (sanity check)
    limit_train_batches=1.0,   # Use only X% of training data
    limit_val_batches=1.0,     # Use only X% of validation data
    deterministic=True,        # Reproducible results (slower)
    detect_anomaly=False,      # Detect NaN/Inf (very slow)
    
    # ====== CALLBACKS ======
    callbacks=[],              # List of callback objects
    
    # ====== PERFORMANCE ======
    benchmark=True,            # cuDNN benchmark (faster, non-deterministic)
)
```

### Debugging with Trainer

```python
# Quick smoke test — runs 1 batch of train + val + test
trainer = L.Trainer(fast_dev_run=True)
trainer.fit(model, train_loader, val_loader)

# Overfit on a few batches (if this doesn't overfit, your model/data has a bug)
trainer = L.Trainer(overfit_batches=10, max_epochs=100)
trainer.fit(model, train_loader)

# Use only 10% of data for fast iteration
trainer = L.Trainer(limit_train_batches=0.1, limit_val_batches=0.1)
trainer.fit(model, train_loader, val_loader)

# Profile your code
trainer = L.Trainer(profiler='simple')  # or 'advanced' or 'pytorch'
trainer.fit(model, train_loader, val_loader)
```

---

## LightningDataModule

### What It Is

**Simple Explanation:** A `LightningDataModule` packages all your data logic in one place — downloading, preprocessing, splitting, and creating DataLoaders. It's a portable, reusable data recipe.

### Why It Matters

- **Reproducibility**: Same data splits every time
- **Portability**: Share data setup across experiments
- **Clean separation**: Data logic separate from model logic
- **Distributed-ready**: Handles multi-GPU data loading correctly

### Complete DataModule Example

```python
import lightning as L
from torch.utils.data import DataLoader, random_split
from torchvision import datasets, transforms

class CIFAR10DataModule(L.LightningDataModule):
    """
    Encapsulates everything about the CIFAR-10 dataset.
    
    DataModule lifecycle:
    ┌──────────────────────────────────────────────────┐
    │  prepare_data()    → Download, preprocess (1 GPU)│
    │  setup(stage)      → Split, transform (all GPUs) │
    │  train_dataloader() → Return training loader     │
    │  val_dataloader()   → Return validation loader   │
    │  test_dataloader()  → Return test loader         │
    │  predict_dataloader() → Return prediction loader │
    │  teardown(stage)   → Clean up                    │
    └──────────────────────────────────────────────────┘
    """
    
    def __init__(self, data_dir='./data', batch_size=32, num_workers=4):
        super().__init__()
        self.save_hyperparameters()
        
        self.data_dir = data_dir
        self.batch_size = batch_size
        self.num_workers = num_workers
        
        # Define transforms
        self.train_transform = transforms.Compose([
            transforms.RandomCrop(32, padding=4),
            transforms.RandomHorizontalFlip(),
            transforms.ColorJitter(0.2, 0.2, 0.2),
            transforms.ToTensor(),
            transforms.Normalize((0.4914, 0.4822, 0.4465),
                               (0.2470, 0.2435, 0.2616)),
        ])
        
        self.test_transform = transforms.Compose([
            transforms.ToTensor(),
            transforms.Normalize((0.4914, 0.4822, 0.4465),
                               (0.2470, 0.2435, 0.2616)),
        ])
    
    def prepare_data(self):
        """
        Download data. Called ONCE on a single GPU.
        DO NOT set state here (self.x = ...) — it won't be available
        on other GPUs in distributed training!
        """
        datasets.CIFAR10(self.data_dir, train=True, download=True)
        datasets.CIFAR10(self.data_dir, train=False, download=True)
    
    def setup(self, stage=None):
        """
        Set up datasets. Called on EVERY GPU.
        
        Args:
            stage: 'fit', 'validate', 'test', or 'predict'
        """
        if stage == 'fit' or stage is None:
            full_train = datasets.CIFAR10(
                self.data_dir, train=True, transform=self.train_transform
            )
            # Split into train and validation
            self.train_dataset, self.val_dataset = random_split(
                full_train, [45000, 5000],
                generator=torch.Generator().manual_seed(42)  # Reproducible split
            )
            # Override transform for validation split
            self.val_dataset.dataset.transform = self.test_transform
        
        if stage == 'test' or stage is None:
            self.test_dataset = datasets.CIFAR10(
                self.data_dir, train=False, transform=self.test_transform
            )
    
    def train_dataloader(self):
        return DataLoader(
            self.train_dataset,
            batch_size=self.batch_size,
            shuffle=True,
            num_workers=self.num_workers,
            pin_memory=True,       # Faster GPU transfer
            persistent_workers=True # Keep workers alive between epochs
        )
    
    def val_dataloader(self):
        return DataLoader(
            self.test_dataset,
            batch_size=self.batch_size * 2,  # Can use larger batch for val
            shuffle=False,
            num_workers=self.num_workers,
            pin_memory=True,
        )
    
    def test_dataloader(self):
        return DataLoader(
            self.test_dataset,
            batch_size=self.batch_size * 2,
            shuffle=False,
            num_workers=self.num_workers,
        )

# ============================================================
# USAGE
# ============================================================

import torch

dm = CIFAR10DataModule(batch_size=64, num_workers=4)
model = ImageClassifier(num_classes=10, learning_rate=1e-3)

trainer = L.Trainer(max_epochs=20, accelerator='auto')
trainer.fit(model, datamodule=dm)  # Pass DataModule, not loaders!
trainer.test(model, datamodule=dm)
```

---

## Callbacks

### What Callbacks Are

**Simple Explanation:** Callbacks are like event handlers — pieces of code that run at specific points during training. "When epoch ends, do X. When validation improves, do Y."

### Callback Lifecycle

```
┌─────────────────────────────────────────────────────────────┐
│                    CALLBACK HOOKS ORDER                      │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│  on_fit_start()                                             │
│  │                                                          │
│  ├── on_train_epoch_start()                                 │
│  │   ├── on_train_batch_start()                             │
│  │   │   └── training_step()                                │
│  │   ├── on_train_batch_end()                               │
│  │   │   ... (repeat for all batches)                       │
│  │   └── on_train_epoch_end()                               │
│  │                                                          │
│  ├── on_validation_epoch_start()                            │
│  │   ├── on_validation_batch_start()                        │
│  │   │   └── validation_step()                              │
│  │   ├── on_validation_batch_end()                          │
│  │   │   ... (repeat)                                       │
│  │   └── on_validation_epoch_end()                          │
│  │                                                          │
│  │   ... (repeat for all epochs)                            │
│  │                                                          │
│  on_fit_end()                                               │
└─────────────────────────────────────────────────────────────┘
```

### Built-In Callbacks

```python
import lightning as L
from lightning.pytorch.callbacks import (
    EarlyStopping,
    ModelCheckpoint,
    LearningRateMonitor,
    RichProgressBar,
    StochasticWeightAveraging,
    GradientAccumulationScheduler,
)

# ============================================================
# EARLY STOPPING
# ============================================================
early_stop = EarlyStopping(
    monitor='val_loss',       # Metric to monitor
    patience=10,              # Epochs to wait before stopping
    mode='min',               # 'min' for loss, 'max' for accuracy
    min_delta=0.001,          # Minimum change to qualify as improvement
    verbose=True,
)

# ============================================================
# MODEL CHECKPOINTING
# ============================================================
checkpoint = ModelCheckpoint(
    dirpath='checkpoints/',
    filename='best-{epoch:02d}-{val_loss:.4f}',  # Dynamic name
    monitor='val_loss',
    mode='min',
    save_top_k=3,             # Save 3 best models
    save_last=True,           # Also save the last model
    every_n_epochs=1,         # Check every epoch
    save_weights_only=False,  # Save full model (not just weights)
)

# ============================================================
# LEARNING RATE MONITOR
# ============================================================
lr_monitor = LearningRateMonitor(logging_interval='step')

# ============================================================
# STOCHASTIC WEIGHT AVERAGING (free accuracy boost!)
# ============================================================
swa = StochasticWeightAveraging(
    swa_lrs=1e-4,             # SWA learning rate
    swa_epoch_start=15,       # Start SWA at epoch 15
)

# ============================================================
# USE THEM ALL
# ============================================================
trainer = L.Trainer(
    max_epochs=50,
    callbacks=[early_stop, checkpoint, lr_monitor, swa],
)
trainer.fit(model, datamodule=dm)

# Load best model after training
best_model_path = checkpoint.best_model_path
print(f"Best model saved at: {best_model_path}")
```

### Custom Callback

```python
import lightning as L
import time

class PrintCallback(L.Callback):
    """Custom callback that prints training summary."""
    
    def on_fit_start(self, trainer, pl_module):
        """Called when training starts."""
        self.start_time = time.time()
        print(f"Training started with {trainer.max_epochs} epochs")
        total_params = sum(p.numel() for p in pl_module.parameters())
        trainable = sum(p.numel() for p in pl_module.parameters() if p.requires_grad)
        print(f"Parameters: {trainable:,} trainable / {total_params:,} total")
    
    def on_train_epoch_end(self, trainer, pl_module):
        """Called at the end of each training epoch."""
        metrics = trainer.callback_metrics
        epoch = trainer.current_epoch
        elapsed = time.time() - self.start_time
        
        train_loss = metrics.get('train_loss_epoch', 0)
        val_loss = metrics.get('val_loss', 0)
        val_acc = metrics.get('val_acc', 0)
        
        print(f"\nEpoch {epoch} ({elapsed:.0f}s) | "
              f"Train Loss: {train_loss:.4f} | "
              f"Val Loss: {val_loss:.4f} | "
              f"Val Acc: {val_acc:.4f}")
    
    def on_fit_end(self, trainer, pl_module):
        """Called when training finishes."""
        total_time = time.time() - self.start_time
        print(f"\nTraining complete in {total_time:.1f}s")

# Usage
trainer = L.Trainer(callbacks=[PrintCallback()])
```

### Freeze/Unfreeze Callback (Gradual Unfreezing)

```python
import lightning as L

class BackboneFineTuning(L.Callback):
    """Gradually unfreeze backbone layers during training."""
    
    def __init__(self, unfreeze_at_epoch=5):
        super().__init__()
        self.unfreeze_at_epoch = unfreeze_at_epoch
    
    def on_train_epoch_start(self, trainer, pl_module):
        if trainer.current_epoch == self.unfreeze_at_epoch:
            # Unfreeze backbone
            for param in pl_module.backbone.parameters():
                param.requires_grad = True
            
            # Add backbone params to optimizer with small LR
            optimizer = trainer.optimizers[0]
            optimizer.add_param_group({
                'params': pl_module.backbone.parameters(),
                'lr': 1e-5,
            })
            print(f"Epoch {self.unfreeze_at_epoch}: Backbone unfrozen!")
```

---

## Logging and Experiment Tracking

### What It Is

Lightning integrates with multiple logging backends so you can track metrics, hyperparameters, and artifacts across experiments.

### Built-In Loggers

```python
import lightning as L
from lightning.pytorch.loggers import (
    TensorBoardLogger,
    CSVLogger,
    WandbLogger,
)

# ============================================================
# TENSORBOARD (default, no extra install needed)
# ============================================================
tb_logger = TensorBoardLogger(
    save_dir='logs/',
    name='my_experiment',
    version='v1',
)

# ============================================================
# CSV LOGGER (simple, no dependencies)
# ============================================================
csv_logger = CSVLogger(
    save_dir='logs/',
    name='csv_experiment',
)

# ============================================================
# WEIGHTS & BIASES (most popular for teams)
# ============================================================
# pip install wandb
wandb_logger = WandbLogger(
    project='my_project',
    name='experiment_1',
    log_model='all',  # Log model checkpoints to W&B
)

# ============================================================
# USE MULTIPLE LOGGERS
# ============================================================
trainer = L.Trainer(
    logger=[tb_logger, csv_logger],  # Log to both!
    log_every_n_steps=50,
)
```

### Logging in LightningModule

```python
class MyModel(L.LightningModule):
    
    def training_step(self, batch, batch_idx):
        images, labels = batch
        logits = self(images)
        loss = F.cross_entropy(logits, labels)
        
        # ============================================================
        # LOGGING METHODS
        # ============================================================
        
        # Single metric
        self.log('train_loss', loss)
        
        # Multiple metrics at once
        self.log_dict({
            'train_loss': loss,
            'train_acc': (logits.argmax(1) == labels).float().mean(),
        }, prog_bar=True)
        
        # Log with specific settings
        self.log('train_loss', loss,
                 on_step=True,      # Log at each step
                 on_epoch=True,     # Also log epoch average
                 prog_bar=True,     # Show in progress bar
                 logger=True,       # Send to logger
                 batch_size=images.size(0))  # For correct averaging
        
        return loss
    
    def on_validation_epoch_end(self):
        """Log custom artifacts at end of validation."""
        # Log images (TensorBoard)
        if hasattr(self.logger, 'experiment'):
            # Access the underlying logger experiment
            pass
    
    def on_train_start(self):
        """Log hyperparameters."""
        self.logger.log_hyperparams(self.hparams)
```

### Viewing TensorBoard Logs

```python
# Launch TensorBoard from terminal:
# tensorboard --logdir=logs/

# Or from a notebook:
# %load_ext tensorboard
# %tensorboard --logdir=logs/
```

---

## Checkpointing

### What It Is

Checkpoints save the complete state of your training — model weights, optimizer state, epoch number, metrics — so you can resume training or load the best model.

### Automatic Checkpointing

```python
from lightning.pytorch.callbacks import ModelCheckpoint

# Save the best model based on validation loss
checkpoint_callback = ModelCheckpoint(
    dirpath='checkpoints/',
    filename='best-{epoch:02d}-{val_loss:.4f}',
    monitor='val_loss',
    mode='min',
    save_top_k=3,         # Keep 3 best checkpoints
    save_last=True,       # Always save the latest
    verbose=True,
)

trainer = L.Trainer(callbacks=[checkpoint_callback])
trainer.fit(model, datamodule=dm)

# Get path to best model
print(f"Best: {checkpoint_callback.best_model_path}")
print(f"Best score: {checkpoint_callback.best_model_score}")
```

### Loading from Checkpoints

```python
# ============================================================
# METHOD 1: Load for inference
# ============================================================
model = ImageClassifier.load_from_checkpoint(
    'checkpoints/best-epoch=15-val_loss=0.2100.ckpt'
)
model.eval()

# Hyperparameters are automatically restored!
print(model.hparams)

# ============================================================
# METHOD 2: Resume training
# ============================================================
model = ImageClassifier(num_classes=10)  # Create fresh model
trainer = L.Trainer(max_epochs=50)

# Resume from checkpoint (restores everything: weights, optimizer, epoch, etc.)
trainer.fit(model, datamodule=dm, ckpt_path='checkpoints/last.ckpt')

# ============================================================
# METHOD 3: Load only weights (ignore optimizer state)
# ============================================================
checkpoint = torch.load('checkpoints/best.ckpt')
model.load_state_dict(checkpoint['state_dict'])

# ============================================================
# METHOD 4: Load with modified hyperparameters
# ============================================================
model = ImageClassifier.load_from_checkpoint(
    'checkpoints/best.ckpt',
    num_classes=20,        # Override original num_classes
    learning_rate=1e-4,    # Override original LR
)
```

### What's Inside a Checkpoint?

```python
import torch

checkpoint = torch.load('checkpoints/best.ckpt', weights_only=False)

print("Keys in checkpoint:")
for key in checkpoint.keys():
    print(f"  {key}")

# Typical keys:
# epoch                  → Current epoch number
# global_step            → Total training steps
# pytorch-lightning_version → Lightning version
# state_dict             → Model weights
# loops                  → Training loop state
# callbacks              → Callback states
# optimizer_states       → Optimizer state (momentum, etc.)
# lr_schedulers          → Scheduler state
# hyper_parameters       → Your saved hyperparameters
```

---

## Multi-GPU and Distributed Training

### What It Is

Lightning makes multi-GPU and distributed training trivial — often requiring just one or two flag changes.

### Single GPU → Multi-GPU

```python
# ============================================================
# SINGLE GPU (default)
# ============================================================
trainer = L.Trainer(accelerator='gpu', devices=1)

# ============================================================
# MULTI-GPU: Data Parallel (simplest, but slower)
# ============================================================
# DON'T use dp strategy - it's deprecated and slow
# Lightning uses ddp by default for multi-GPU

# ============================================================
# MULTI-GPU: Distributed Data Parallel (recommended)
# ============================================================
trainer = L.Trainer(
    accelerator='gpu',
    devices=4,             # Use 4 GPUs
    strategy='ddp',        # Distributed Data Parallel
)

# ============================================================
# SPECIFIC GPUs
# ============================================================
trainer = L.Trainer(
    accelerator='gpu',
    devices=[0, 2],        # Use GPU 0 and GPU 2
)

# ============================================================
# ALL AVAILABLE GPUs
# ============================================================
trainer = L.Trainer(
    accelerator='gpu',
    devices='auto',        # Use all GPUs
)
```

### How DDP Works

```
┌──────────────────────────────────────────────────────────────┐
│           DISTRIBUTED DATA PARALLEL (DDP)                     │
├──────────────────────────────────────────────────────────────┤
│                                                               │
│  Full Dataset                                                │
│  ┌─────────────────────────────────────────────┐             │
│  │ Batch 1 │ Batch 2 │ Batch 3 │ Batch 4 │... │             │
│  └────┬────┴────┬────┴────┬────┴────┬────┘    │             │
│       │         │         │         │                        │
│       ▼         ▼         ▼         ▼                        │
│    GPU 0     GPU 1     GPU 2     GPU 3                       │
│   ┌──────┐ ┌──────┐ ┌──────┐ ┌──────┐                      │
│   │Model │ │Model │ │Model │ │Model │  ← Same model copy   │
│   │Copy  │ │Copy  │ │Copy  │ │Copy  │                       │
│   └──┬───┘ └──┬───┘ └──┬───┘ └──┬───┘                      │
│      │        │        │        │                            │
│      ▼        ▼        ▼        ▼                            │
│   Forward  Forward  Forward  Forward   ← Independent        │
│   Backward Backward Backward Backward  ← Independent        │
│      │        │        │        │                            │
│      └────────┴────────┴────────┘                            │
│                    │                                         │
│              All-Reduce                                      │
│         (average gradients)                                  │
│                    │                                         │
│              Update Weights                                  │
│         (identical on all GPUs)                              │
└──────────────────────────────────────────────────────────────┘
```

### FSDP (Fully Sharded Data Parallel) — For Very Large Models

```python
# FSDP shards model weights across GPUs (not just data)
# Needed when model doesn't fit on a single GPU

trainer = L.Trainer(
    accelerator='gpu',
    devices=4,
    strategy='fsdp',       # Fully Sharded Data Parallel
    precision='16-mixed',  # Almost always used with FSDP
)

# With custom FSDP config
from lightning.pytorch.strategies import FSDPStrategy

strategy = FSDPStrategy(
    sharding_strategy='FULL_SHARD',  # Shard everything
    activation_checkpointing_policy={  # Which layers to checkpoint
        nn.TransformerEncoderLayer,
    },
)

trainer = L.Trainer(
    accelerator='gpu',
    devices=4,
    strategy=strategy,
    precision='16-mixed',
)
```

### Strategy Comparison

| Strategy | Use When | Memory | Speed | Complexity |
|----------|----------|--------|-------|------------|
| Single GPU | Model fits on 1 GPU | Baseline | Fastest | None |
| DDP | Model fits on 1 GPU, want faster training | Same per GPU | Near-linear scaling | Low |
| FSDP | Model too big for 1 GPU | Sharded (much less) | Good scaling | Medium |
| DeepSpeed | Very large models, advanced optimization | Sharded | Excellent | High |

---

## Advanced Lightning Patterns

### 1. Multiple Optimizers (GANs)

```python
class GAN(L.LightningModule):
    def __init__(self, latent_dim=100):
        super().__init__()
        self.save_hyperparameters()
        self.automatic_optimization = False  # Manual optimization for GANs!
        
        self.generator = nn.Sequential(
            nn.Linear(latent_dim, 256),
            nn.ReLU(),
            nn.Linear(256, 784),
            nn.Tanh()
        )
        
        self.discriminator = nn.Sequential(
            nn.Linear(784, 256),
            nn.LeakyReLU(0.2),
            nn.Linear(256, 1),
            nn.Sigmoid()
        )
    
    def training_step(self, batch, batch_idx):
        real_images, _ = batch
        real_images = real_images.view(real_images.size(0), -1)
        batch_size = real_images.size(0)
        
        # Get optimizers
        opt_g, opt_d = self.optimizers()
        
        # ====== Train Discriminator ======
        z = torch.randn(batch_size, self.hparams.latent_dim, device=self.device)
        fake_images = self.generator(z).detach()  # Detach to not update G
        
        real_preds = self.discriminator(real_images)
        fake_preds = self.discriminator(fake_images)
        
        d_loss = -torch.mean(torch.log(real_preds + 1e-8) + 
                            torch.log(1 - fake_preds + 1e-8))
        
        opt_d.zero_grad()
        self.manual_backward(d_loss)
        opt_d.step()
        
        # ====== Train Generator ======
        z = torch.randn(batch_size, self.hparams.latent_dim, device=self.device)
        fake_images = self.generator(z)
        fake_preds = self.discriminator(fake_images)
        
        g_loss = -torch.mean(torch.log(fake_preds + 1e-8))
        
        opt_g.zero_grad()
        self.manual_backward(g_loss)
        opt_g.step()
        
        self.log_dict({'d_loss': d_loss, 'g_loss': g_loss}, prog_bar=True)
    
    def configure_optimizers(self):
        opt_g = torch.optim.Adam(self.generator.parameters(), lr=2e-4)
        opt_d = torch.optim.Adam(self.discriminator.parameters(), lr=2e-4)
        return [opt_g, opt_d]  # Return list of optimizers
```

### 2. Using torchmetrics for Proper Metric Tracking

```python
import lightning as L
import torch.nn.functional as F
import torchmetrics

class MetricModel(L.LightningModule):
    """Using torchmetrics for proper distributed metric computation."""
    
    def __init__(self, num_classes=10):
        super().__init__()
        self.save_hyperparameters()
        self.model = nn.Linear(784, num_classes)
        
        # torchmetrics handles distributed aggregation correctly!
        self.train_acc = torchmetrics.Accuracy(task='multiclass', num_classes=num_classes)
        self.val_acc = torchmetrics.Accuracy(task='multiclass', num_classes=num_classes)
        self.val_f1 = torchmetrics.F1Score(task='multiclass', num_classes=num_classes, average='macro')
        self.val_confusion = torchmetrics.ConfusionMatrix(task='multiclass', num_classes=num_classes)
    
    def training_step(self, batch, batch_idx):
        x, y = batch
        logits = self.model(x.view(x.size(0), -1))
        loss = F.cross_entropy(logits, y)
        
        preds = logits.argmax(dim=1)
        self.train_acc(preds, y)  # Update metric state
        
        self.log('train_loss', loss, prog_bar=True)
        self.log('train_acc', self.train_acc, on_step=False, on_epoch=True, prog_bar=True)
        
        return loss
    
    def validation_step(self, batch, batch_idx):
        x, y = batch
        logits = self.model(x.view(x.size(0), -1))
        loss = F.cross_entropy(logits, y)
        
        preds = logits.argmax(dim=1)
        self.val_acc(preds, y)
        self.val_f1(preds, y)
        self.val_confusion(preds, y)
        
        self.log('val_loss', loss, prog_bar=True)
        self.log('val_acc', self.val_acc, on_epoch=True, prog_bar=True)
        self.log('val_f1', self.val_f1, on_epoch=True)

    def configure_optimizers(self):
        return torch.optim.Adam(self.parameters(), lr=1e-3)
```

### 3. Hyperparameter Tuning with Optuna

```python
import optuna
import lightning as L

def objective(trial):
    """Optuna objective function using Lightning."""
    
    # Suggest hyperparameters
    lr = trial.suggest_float('lr', 1e-5, 1e-2, log=True)
    dropout = trial.suggest_float('dropout', 0.1, 0.5)
    hidden_size = trial.suggest_categorical('hidden_size', [128, 256, 512])
    batch_size = trial.suggest_categorical('batch_size', [32, 64, 128])
    
    # Create model and data with suggested params
    model = ImageClassifier(
        num_classes=10,
        learning_rate=lr,
    )
    dm = CIFAR10DataModule(batch_size=batch_size)
    
    trainer = L.Trainer(
        max_epochs=20,
        accelerator='auto',
        callbacks=[
            L.pytorch.callbacks.EarlyStopping(monitor='val_loss', patience=5),
        ],
        enable_progress_bar=False,  # Quiet for HPO
    )
    
    trainer.fit(model, datamodule=dm)
    
    # Return the metric to optimize
    return trainer.callback_metrics['val_loss'].item()

# Run HPO
study = optuna.create_study(direction='minimize')
study.optimize(objective, n_trials=50)

print(f"Best params: {study.best_params}")
print(f"Best val_loss: {study.best_value:.4f}")
```

### 4. CLI Integration (Production-Ready)

```python
# In train.py:
import lightning as L
from lightning.pytorch.cli import LightningCLI

class MyModel(L.LightningModule):
    def __init__(self, lr: float = 1e-3, hidden: int = 256):
        super().__init__()
        self.save_hyperparameters()
        self.model = nn.Linear(784, hidden)
        # ...

class MyData(L.LightningDataModule):
    def __init__(self, batch_size: int = 32, data_dir: str = './data'):
        super().__init__()
        self.save_hyperparameters()
        # ...

# This automatically creates a CLI!
cli = LightningCLI(MyModel, MyData)

# Now run from command line:
# python train.py fit --model.lr=0.001 --data.batch_size=64 --trainer.max_epochs=50
# python train.py test --ckpt_path=best.ckpt
# python train.py predict --ckpt_path=best.ckpt
```

---

## Common Mistakes

### 1. Setting State in `prepare_data()`

```python
# ❌ WRONG: prepare_data runs on ONE GPU only
class BadDataModule(L.LightningDataModule):
    def prepare_data(self):
        self.train_data = load_data()  # NOT available on other GPUs!

# ✅ CORRECT: Set state in setup(), which runs on ALL GPUs
class GoodDataModule(L.LightningDataModule):
    def prepare_data(self):
        download_data()  # Only download/preprocess
    
    def setup(self, stage=None):
        self.train_data = load_data()  # Available on all GPUs
```

### 2. Calling `.backward()` Instead of `self.manual_backward()`

```python
# ❌ WRONG in manual optimization mode:
loss.backward()  # Skips Lightning's gradient scaling!

# ✅ CORRECT:
self.manual_backward(loss)  # Handles AMP scaling, etc.
```

### 3. Forgetting `self.log()` batch_size for Variable-Length Data

```python
# ❌ WRONG: Averages will be incorrect with variable batch sizes
self.log('val_loss', loss)  # Assumes fixed batch size

# ✅ CORRECT: Specify batch_size for correct weighted averaging
self.log('val_loss', loss, batch_size=images.size(0))
```

### 4. Not Using `sync_dist=True` in Multi-GPU

```python
# ❌ WRONG: Each GPU logs its own metric, not synchronized
self.log('val_acc', acc)

# ✅ CORRECT: Synchronize across GPUs
self.log('val_acc', acc, sync_dist=True)

# Better: use torchmetrics (handles sync automatically)
```

### 5. Storing Tensors in Lists Across Steps

```python
# ❌ WRONG: Memory leak! Tensors accumulate on GPU
class BadModel(L.LightningModule):
    def __init__(self):
        super().__init__()
        self.all_preds = []  # Grows forever!
    
    def validation_step(self, batch, batch_idx):
        preds = self(batch[0])
        self.all_preds.append(preds)  # GPU memory leak!

# ✅ CORRECT: Detach and move to CPU, or use torchmetrics
class GoodModel(L.LightningModule):
    def validation_step(self, batch, batch_idx):
        preds = self(batch[0])
        return preds.detach().cpu()  # Or just use torchmetrics
```

### 6. Not Calling `super().__init__()` or `self.save_hyperparameters()`

```python
# ❌ WRONG: Forgetting super().__init__()
class BadModel(L.LightningModule):
    def __init__(self, lr=1e-3):
        self.lr = lr  # Crash! Module not initialized

# ✅ CORRECT
class GoodModel(L.LightningModule):
    def __init__(self, lr=1e-3):
        super().__init__()
        self.save_hyperparameters()  # Saves lr to self.hparams and checkpoints
```

---

## Interview Questions

### Q1: What is the difference between `prepare_data()` and `setup()` in LightningDataModule?

**Answer:**
- `prepare_data()`: Called once on a single process/GPU. Used for downloading/preprocessing data that should only happen once. DO NOT set instance state here (no `self.x = ...`).
- `setup(stage)`: Called on every GPU/process. Used for creating dataset objects, splits, and transforms. Safe to set instance state.

This separation prevents data corruption in distributed training where multiple processes would try to download simultaneously.

### Q2: When would you use `self.automatic_optimization = False`?

**Answer:** Manual optimization is needed when:
1. **GANs** — Two models with separate optimizers and alternating training
2. **Reinforcement Learning** — Custom optimization schedules
3. **Research** — Need control over when gradients are applied
4. **Multiple losses** — Need to backward through different losses at different times

With manual optimization, you call `self.manual_backward(loss)`, `optimizer.zero_grad()`, and `optimizer.step()` yourself.

### Q3: How does Lightning handle distributed training under the hood?

**Answer:** Lightning uses DDP (DistributedDataParallel) by default:
1. Spawns N processes (one per GPU)
2. Each process gets a full model copy
3. Data is sharded via `DistributedSampler` (each GPU sees different batches)
4. Forward and backward passes run independently on each GPU
5. Gradients are synchronized via all-reduce (averaged across GPUs)
6. All GPUs apply the same weight update

Lightning handles the sampler, process spawning, gradient sync, and metric aggregation automatically.

### Q4: What is the difference between DDP and FSDP?

**Answer:**

| Aspect | DDP | FSDP |
|--------|-----|------|
| Model copy | Full model on each GPU | Model sharded across GPUs |
| Memory per GPU | Full model + gradients + optimizer | 1/N model + 1/N gradients + 1/N optimizer |
| Communication | All-reduce gradients | All-gather weights, reduce-scatter gradients |
| Best for | Models that fit on 1 GPU | Models too large for 1 GPU |
| Complexity | Low | Medium |
| Speed | Very fast | Slightly slower due to communication |

### Q5: How do you resume training from a checkpoint in Lightning?

**Answer:**
```python
# Resume everything (model, optimizer, epoch, step):
trainer.fit(model, datamodule=dm, ckpt_path='path/to/checkpoint.ckpt')

# Load only model weights (new training run):
model = MyModel.load_from_checkpoint('path/to/checkpoint.ckpt')
trainer.fit(model, datamodule=dm)  # Starts from epoch 0
```

The `ckpt_path` approach restores the full training state, while `load_from_checkpoint` only loads weights and hyperparameters.

### Q6: What is `save_hyperparameters()` and why is it important?

**Answer:** `self.save_hyperparameters()` stores all `__init__` arguments in `self.hparams` and includes them in checkpoints. Benefits:
1. Hyperparameters are logged automatically
2. `load_from_checkpoint()` reconstructs the model with correct args
3. CLI integration reads them for command-line arguments
4. Experiment tracking captures them

Without it, you'd need to manually track and pass constructor arguments when loading.

### Q7: How would you implement a custom learning rate scheduler in Lightning?

**Answer:**
```python
def configure_optimizers(self):
    optimizer = torch.optim.Adam(self.parameters(), lr=1e-3)
    scheduler = torch.optim.lr_scheduler.CosineAnnealingLR(optimizer, T_max=10)
    
    return {
        'optimizer': optimizer,
        'lr_scheduler': {
            'scheduler': scheduler,
            'interval': 'epoch',    # or 'step'
            'frequency': 1,
            'monitor': 'val_loss',  # For ReduceLROnPlateau
            'strict': True,
        }
    }
```

Key options: `interval` controls step-level vs epoch-level scheduling. `monitor` is needed for metric-based schedulers like ReduceLROnPlateau.

---

## Quick Reference

### LightningModule Methods Cheat Sheet

| Method | When It's Called | What to Do |
|--------|-----------------|------------|
| `__init__` | Object creation | Define model, loss, metrics |
| `forward` | `self(x)` or `model(x)` | Pure inference logic |
| `training_step` | Each training batch | Forward + loss + logging |
| `validation_step` | Each validation batch | Forward + metrics |
| `test_step` | Each test batch | Forward + final metrics |
| `predict_step` | Each prediction batch | Forward only |
| `configure_optimizers` | Before training starts | Return optimizer(s) + scheduler(s) |
| `on_train_epoch_start` | Start of each train epoch | Scheduled changes (unfreeze, etc.) |
| `on_train_epoch_end` | End of each train epoch | Aggregate custom metrics |

### Trainer Flags Cheat Sheet

| Flag | Example | Purpose |
|------|---------|---------|
| `max_epochs` | `100` | Training duration |
| `accelerator` | `'gpu'` | Hardware type |
| `devices` | `4` or `[0,1]` | Number/IDs of devices |
| `strategy` | `'ddp'` | Distributed strategy |
| `precision` | `'16-mixed'` | Mixed precision |
| `gradient_clip_val` | `1.0` | Gradient clipping |
| `accumulate_grad_batches` | `4` | Gradient accumulation |
| `fast_dev_run` | `True` | 1-batch debug run |
| `overfit_batches` | `10` | Overfit sanity check |
| `deterministic` | `True` | Reproducibility |
| `profiler` | `'simple'` | Performance profiling |

### Migration from Pure PyTorch

```
Pure PyTorch                    →  Lightning
─────────────────────────────────────────────────────
model = MyNet()                 →  class MyNet(L.LightningModule)
criterion = nn.CrossEntropyLoss →  F.cross_entropy in training_step
optimizer = Adam(...)           →  configure_optimizers()
for epoch in range(N):          →  trainer.fit()
  model.train()                 →  (automatic)
  for batch in loader:          →  training_step(batch, idx)
    x = x.to(device)           →  (automatic)
    optimizer.zero_grad()       →  (automatic)
    loss = criterion(model(x))  →  return loss
    loss.backward()             →  (automatic)
    optimizer.step()            →  (automatic)
  model.eval()                  →  (automatic)
  with torch.no_grad():        →  (automatic)
    validate(...)               →  validation_step(batch, idx)
  save_checkpoint(...)          →  ModelCheckpoint callback
```

### Complete Training Template

```python
import torch
import torch.nn as nn
import torch.nn.functional as F
import lightning as L
from lightning.pytorch.callbacks import ModelCheckpoint, EarlyStopping, LearningRateMonitor

# 1. Define Model
class MyModel(L.LightningModule):
    def __init__(self, lr=1e-3):
        super().__init__()
        self.save_hyperparameters()
        self.model = nn.Sequential(...)
    
    def forward(self, x):
        return self.model(x)
    
    def training_step(self, batch, batch_idx):
        x, y = batch
        loss = F.cross_entropy(self(x), y)
        self.log('train_loss', loss, prog_bar=True)
        return loss
    
    def validation_step(self, batch, batch_idx):
        x, y = batch
        loss = F.cross_entropy(self(x), y)
        acc = (self(x).argmax(1) == y).float().mean()
        self.log_dict({'val_loss': loss, 'val_acc': acc}, prog_bar=True)
    
    def configure_optimizers(self):
        opt = torch.optim.AdamW(self.parameters(), lr=self.hparams.lr)
        sch = torch.optim.lr_scheduler.CosineAnnealingLR(opt, T_max=self.trainer.max_epochs)
        return {'optimizer': opt, 'lr_scheduler': sch}

# 2. Define Data
class MyData(L.LightningDataModule):
    def __init__(self, batch_size=32):
        super().__init__()
        self.save_hyperparameters()
    def prepare_data(self): ...
    def setup(self, stage=None): ...
    def train_dataloader(self): ...
    def val_dataloader(self): ...

# 3. Train
model = MyModel(lr=1e-3)
dm = MyData(batch_size=64)

trainer = L.Trainer(
    max_epochs=50,
    accelerator='auto',
    precision='16-mixed',
    callbacks=[
        ModelCheckpoint(monitor='val_loss', mode='min', save_top_k=3),
        EarlyStopping(monitor='val_loss', patience=10),
        LearningRateMonitor(),
    ],
)
trainer.fit(model, datamodule=dm)
trainer.test(model, datamodule=dm)
```
