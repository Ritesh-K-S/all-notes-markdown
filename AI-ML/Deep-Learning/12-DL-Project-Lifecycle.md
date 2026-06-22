# Chapter 12: Deep Learning Project Lifecycle

## Data Preparation, Training Pipelines, Experiment Tracking, and Model Registry

---

## Table of Contents
- [12.1 Overview — The DL Project Lifecycle](#121-overview--the-dl-project-lifecycle)
- [12.2 Data Preparation and Management](#122-data-preparation-and-management)
- [12.3 Training Pipelines — Production-Grade](#123-training-pipelines--production-grade)
- [12.4 Experiment Tracking](#124-experiment-tracking)
- [12.5 Model Registry and Versioning](#125-model-registry-and-versioning)
- [12.6 Debugging and Monitoring Training](#126-debugging-and-monitoring-training)
- [12.7 Reproducibility](#127-reproducibility)
- [12.8 From Prototype to Production](#128-from-prototype-to-production)
- [12.9 Common Mistakes](#129-common-mistakes)
- [12.10 Interview Questions](#1210-interview-questions)
- [12.11 Quick Reference](#1211-quick-reference)

---

## 12.1 Overview — The DL Project Lifecycle

### What It Is
The DL project lifecycle is the **end-to-end process** of building a deep learning system — from raw data to a deployed, monitored model in production. Think of it like building a house: you don't just stack bricks (write model code). You need blueprints (planning), foundation (data), construction (training), inspection (evaluation), and maintenance (monitoring).

### Why It Matters
- 85% of ML projects **never reach production** — not because of bad models, but bad engineering
- The model code is only ~10% of a real ML system
- Poor data management and tracking are the #1 cause of project failure
- Companies that master the lifecycle ship models 10× faster

### The Complete Lifecycle

```
┌─────────────────────────────────────────────────────────────────┐
│                    DL PROJECT LIFECYCLE                           │
│                                                                   │
│  ┌──────────┐   ┌──────────┐   ┌──────────┐   ┌──────────┐    │
│  │ 1. DATA  │──▶│ 2. MODEL │──▶│ 3. TRAIN │──▶│4. EVALUATE│    │
│  │          │   │  DESIGN  │   │          │   │           │    │
│  │• Collect │   │• Arch    │   │• Pipeline│   │• Metrics  │    │
│  │• Clean   │   │• Baseline│   │• Tracking│   │• Analysis │    │
│  │• Version │   │• Iterate │   │• Tune HP │   │• Bias/Fair│    │
│  └──────────┘   └──────────┘   └──────────┘   └──────────┘    │
│       ▲                                              │           │
│       │                                              ▼           │
│  ┌──────────┐   ┌──────────┐   ┌──────────┐   ┌──────────┐    │
│  │8. MONITOR│◀──│7. DEPLOY │◀──│6. PACKAGE│◀──│5. REGISTRY│    │
│  │          │   │          │   │          │   │           │    │
│  │• Drift   │   │• Serve   │   │• ONNX    │   │• Version  │    │
│  │• Perf    │   │• Scale   │   │• Docker  │   │• Stage    │    │
│  │• Retrain │   │• A/B Test│   │• Optimize│   │• Metadata │    │
│  └──────────┘   └──────────┘   └──────────┘   └──────────┘    │
│       │                                                          │
│       └──────────────── Feedback Loop ──────────────────────────┘
└─────────────────────────────────────────────────────────────────┘
```

---

## 12.2 Data Preparation and Management

### What It Is
Data preparation encompasses everything from collecting raw data to having clean, versioned, properly split datasets ready for training. It's the **foundation** — a model can never be better than its data.

### Why It Matters
- "Garbage in, garbage out" — no architecture can fix bad data
- Data issues cause **>60%** of model failures in production
- Proper versioning prevents the "which dataset did we use?" problem
- Good data pipelines save weeks of debugging

### Data Pipeline Architecture

```
Raw Data Sources                Processing                  Ready for Training
━━━━━━━━━━━━━━━━               ━━━━━━━━━━                  ━━━━━━━━━━━━━━━━━

┌─────────┐
│Databases│──┐
└─────────┘  │     ┌────────────────────┐      ┌─────────────────┐
┌─────────┐  │     │  Data Pipeline     │      │  Versioned      │
│  APIs   │──┼────▶│                    │─────▶│  Dataset        │
└─────────┘  │     │  • Validation      │      │                 │
┌─────────┐  │     │  • Cleaning        │      │  • train/       │
│  Files  │──┤     │  • Augmentation    │      │  • val/         │
└─────────┘  │     │  • Splitting       │      │  • test/        │
┌─────────┐  │     │  • Versioning      │      │  • metadata.json│
│Labeling │──┘     └────────────────────┘      └─────────────────┘
│Platform │
└─────────┘
```

### Code Example — Complete Data Pipeline

```python
import os
import json
import hashlib
import shutil
from pathlib import Path
from datetime import datetime
from typing import Dict, List, Tuple, Optional
import numpy as np
from PIL import Image
import torch
from torch.utils.data import Dataset, DataLoader, WeightedRandomSampler
from torchvision import transforms
from sklearn.model_selection import train_test_split

# ============================================================
# 1. DATA VALIDATION
# ============================================================

class DataValidator:
    """Validate data quality before training."""
    
    def __init__(self, data_dir: str):
        self.data_dir = Path(data_dir)
        self.issues = []
    
    def validate_images(self, extensions=('.jpg', '.jpeg', '.png', '.bmp')):
        """Check for corrupt, wrong-format, or problematic images."""
        print("Validating images...")
        
        valid_count = 0
        corrupt_count = 0
        size_issues = 0
        
        for img_path in self.data_dir.rglob('*'):
            if img_path.suffix.lower() not in extensions:
                continue
            
            try:
                img = Image.open(img_path)
                img.verify()  # verify file integrity
                
                # Re-open for size check (verify() closes the image)
                img = Image.open(img_path)
                w, h = img.size
                
                # Flag very small images
                if w < 32 or h < 32:
                    self.issues.append(f"TOO SMALL: {img_path} ({w}×{h})")
                    size_issues += 1
                
                # Flag extreme aspect ratios
                aspect = max(w, h) / min(w, h)
                if aspect > 10:
                    self.issues.append(f"EXTREME ASPECT: {img_path} (ratio={aspect:.1f})")
                
                valid_count += 1
                
            except Exception as e:
                self.issues.append(f"CORRUPT: {img_path} — {e}")
                corrupt_count += 1
        
        print(f"  Valid: {valid_count}, Corrupt: {corrupt_count}, Size issues: {size_issues}")
        return corrupt_count == 0
    
    def validate_class_balance(self, min_samples_per_class=10, max_imbalance_ratio=20):
        """Check class distribution."""
        class_counts = {}
        
        for class_dir in sorted(self.data_dir.iterdir()):
            if class_dir.is_dir():
                count = len(list(class_dir.iterdir()))
                class_counts[class_dir.name] = count
        
        if not class_counts:
            self.issues.append("No class directories found!")
            return False
        
        min_count = min(class_counts.values())
        max_count = max(class_counts.values())
        ratio = max_count / max(min_count, 1)
        
        print(f"  Classes: {len(class_counts)}")
        print(f"  Min samples: {min_count}, Max: {max_count}, Ratio: {ratio:.1f}")
        
        if min_count < min_samples_per_class:
            self.issues.append(f"Class with only {min_count} samples (minimum: {min_samples_per_class})")
        
        if ratio > max_imbalance_ratio:
            self.issues.append(f"Class imbalance ratio {ratio:.1f} exceeds {max_imbalance_ratio}")
        
        return ratio <= max_imbalance_ratio
    
    def validate_duplicates(self):
        """Find duplicate images using file hashing."""
        hashes = {}
        duplicates = []
        
        for img_path in self.data_dir.rglob('*.jpg'):
            file_hash = hashlib.md5(img_path.read_bytes()).hexdigest()
            
            if file_hash in hashes:
                duplicates.append((img_path, hashes[file_hash]))
            else:
                hashes[file_hash] = img_path
        
        if duplicates:
            print(f"  Found {len(duplicates)} duplicate pairs!")
            self.issues.extend([f"DUPLICATE: {a} == {b}" for a, b in duplicates[:5]])
        
        return len(duplicates) == 0
    
    def report(self) -> str:
        """Generate validation report."""
        report = f"\n{'='*60}\nDATA VALIDATION REPORT\n{'='*60}\n"
        report += f"Data directory: {self.data_dir}\n"
        report += f"Issues found: {len(self.issues)}\n\n"
        
        for issue in self.issues:
            report += f"  ⚠️  {issue}\n"
        
        if not self.issues:
            report += "  ✅ All checks passed!\n"
        
        return report


# ============================================================
# 2. DATA VERSIONING
# ============================================================

class DatasetVersionManager:
    """Simple dataset versioning (DVC-lite)."""
    
    def __init__(self, base_dir: str):
        self.base_dir = Path(base_dir)
        self.versions_dir = self.base_dir / '.versions'
        self.versions_dir.mkdir(exist_ok=True)
    
    def create_version(self, description: str) -> str:
        """Create a new version snapshot with metadata."""
        timestamp = datetime.now().strftime('%Y%m%d_%H%M%S')
        version_id = f"v_{timestamp}"
        
        # Compute dataset fingerprint
        file_list = sorted(self.base_dir.rglob('*'))
        file_list = [f for f in file_list if f.is_file() and '.versions' not in str(f)]
        
        hasher = hashlib.sha256()
        for f in file_list:
            hasher.update(str(f.relative_to(self.base_dir)).encode())
            hasher.update(str(f.stat().st_size).encode())
        
        fingerprint = hasher.hexdigest()[:12]
        
        # Save metadata
        metadata = {
            'version_id': version_id,
            'description': description,
            'timestamp': datetime.now().isoformat(),
            'fingerprint': fingerprint,
            'num_files': len(file_list),
            'total_size_mb': sum(f.stat().st_size for f in file_list) / 1e6,
            'file_manifest': [str(f.relative_to(self.base_dir)) for f in file_list[:100]],
        }
        
        version_file = self.versions_dir / f"{version_id}.json"
        version_file.write_text(json.dumps(metadata, indent=2))
        
        print(f"Created version: {version_id}")
        print(f"  Files: {len(file_list)}, Size: {metadata['total_size_mb']:.1f} MB")
        print(f"  Fingerprint: {fingerprint}")
        
        return version_id
    
    def list_versions(self) -> List[Dict]:
        """List all dataset versions."""
        versions = []
        for vf in sorted(self.versions_dir.glob('*.json')):
            versions.append(json.loads(vf.read_text()))
        return versions


# ============================================================
# 3. SMART DATA SPLITTING
# ============================================================

def create_stratified_split(
    data_dir: str,
    output_dir: str,
    train_ratio: float = 0.7,
    val_ratio: float = 0.15,
    test_ratio: float = 0.15,
    seed: int = 42,
):
    """
    Create stratified train/val/test split maintaining class proportions.
    Also handles group splitting (e.g., same patient's images stay together).
    """
    assert abs(train_ratio + val_ratio + test_ratio - 1.0) < 1e-6
    
    data_path = Path(data_dir)
    output_path = Path(output_dir)
    
    # Collect all samples with labels
    samples = []
    for class_dir in sorted(data_path.iterdir()):
        if class_dir.is_dir():
            for img_file in class_dir.iterdir():
                if img_file.suffix.lower() in ('.jpg', '.jpeg', '.png'):
                    samples.append({
                        'path': img_file,
                        'label': class_dir.name
                    })
    
    # Stratified split
    paths = [s['path'] for s in samples]
    labels = [s['label'] for s in samples]
    
    # First split: train vs (val+test)
    train_paths, temp_paths, train_labels, temp_labels = train_test_split(
        paths, labels, test_size=(val_ratio + test_ratio),
        stratify=labels, random_state=seed
    )
    
    # Second split: val vs test
    relative_test = test_ratio / (val_ratio + test_ratio)
    val_paths, test_paths, val_labels, test_labels = train_test_split(
        temp_paths, temp_labels, test_size=relative_test,
        stratify=temp_labels, random_state=seed
    )
    
    # Copy files to split directories
    splits = {
        'train': (train_paths, train_labels),
        'val': (val_paths, val_labels),
        'test': (test_paths, test_labels),
    }
    
    for split_name, (split_paths, split_labels) in splits.items():
        for path, label in zip(split_paths, split_labels):
            dest_dir = output_path / split_name / label
            dest_dir.mkdir(parents=True, exist_ok=True)
            shutil.copy2(path, dest_dir / path.name)
    
    # Save split metadata
    metadata = {
        'created': datetime.now().isoformat(),
        'seed': seed,
        'ratios': {'train': train_ratio, 'val': val_ratio, 'test': test_ratio},
        'counts': {
            'train': len(train_paths),
            'val': len(val_paths),
            'test': len(test_paths),
        },
        'class_distribution': {
            split: dict(zip(*np.unique(labels, return_counts=True)))
            for split, (_, labels) in splits.items()
        }
    }
    
    (output_path / 'split_metadata.json').write_text(json.dumps(metadata, indent=2))
    print(f"Split created: train={len(train_paths)}, val={len(val_paths)}, test={len(test_paths)}")
    
    return metadata


# ============================================================
# 4. PRODUCTION-READY DATASET CLASS
# ============================================================

class ProductionDataset(Dataset):
    """
    Production-quality dataset with:
    - Proper error handling for corrupt files
    - Caching for repeated access
    - Balanced sampling support
    - Comprehensive augmentation pipeline
    """
    
    def __init__(
        self,
        data_dir: str,
        split: str = 'train',
        img_size: int = 224,
        augment: bool = True,
        cache_in_memory: bool = False,
    ):
        self.data_dir = Path(data_dir) / split
        self.split = split
        self.cache_in_memory = cache_in_memory
        self.cache = {}
        
        # Collect samples
        self.samples = []
        self.classes = sorted([d.name for d in self.data_dir.iterdir() if d.is_dir()])
        self.class_to_idx = {c: i for i, c in enumerate(self.classes)}
        
        for class_name in self.classes:
            class_dir = self.data_dir / class_name
            for img_path in class_dir.iterdir():
                if img_path.suffix.lower() in ('.jpg', '.jpeg', '.png'):
                    self.samples.append((img_path, self.class_to_idx[class_name]))
        
        # Define transforms
        if augment and split == 'train':
            self.transform = transforms.Compose([
                transforms.RandomResizedCrop(img_size, scale=(0.8, 1.0)),
                transforms.RandomHorizontalFlip(),
                transforms.RandomRotation(15),
                transforms.ColorJitter(brightness=0.2, contrast=0.2, saturation=0.2, hue=0.1),
                transforms.RandomGrayscale(p=0.02),
                transforms.ToTensor(),
                transforms.Normalize([0.485, 0.456, 0.406], [0.229, 0.224, 0.225]),
                transforms.RandomErasing(p=0.1),  # CutOut-style augmentation
            ])
        else:
            self.transform = transforms.Compose([
                transforms.Resize(int(img_size * 1.14)),  # 256 for img_size=224
                transforms.CenterCrop(img_size),
                transforms.ToTensor(),
                transforms.Normalize([0.485, 0.456, 0.406], [0.229, 0.224, 0.225]),
            ])
    
    def __len__(self):
        return len(self.samples)
    
    def __getitem__(self, idx):
        img_path, label = self.samples[idx]
        
        # Check cache
        if self.cache_in_memory and idx in self.cache:
            return self.cache[idx], label
        
        try:
            img = Image.open(img_path).convert('RGB')
            img = self.transform(img)
        except Exception as e:
            # Handle corrupt images gracefully — return a random valid sample
            print(f"Warning: Failed to load {img_path}: {e}")
            return self.__getitem__((idx + 1) % len(self))
        
        if self.cache_in_memory:
            self.cache[idx] = img
        
        return img, label
    
    def get_sampler_weights(self) -> torch.Tensor:
        """Get sample weights for balanced sampling (handles class imbalance)."""
        labels = [label for _, label in self.samples]
        class_counts = np.bincount(labels)
        class_weights = 1.0 / class_counts
        sample_weights = torch.tensor([class_weights[label] for label in labels])
        return sample_weights


def create_dataloaders(
    data_dir: str,
    batch_size: int = 32,
    img_size: int = 224,
    num_workers: int = 4,
    balanced_sampling: bool = True,
) -> Dict[str, DataLoader]:
    """Create train/val/test dataloaders with best practices."""
    
    datasets_dict = {
        'train': ProductionDataset(data_dir, 'train', img_size, augment=True),
        'val': ProductionDataset(data_dir, 'val', img_size, augment=False),
        'test': ProductionDataset(data_dir, 'test', img_size, augment=False),
    }
    
    # Balanced sampler for training (handles class imbalance)
    train_sampler = None
    train_shuffle = True
    if balanced_sampling:
        weights = datasets_dict['train'].get_sampler_weights()
        train_sampler = WeightedRandomSampler(weights, len(weights), replacement=True)
        train_shuffle = False  # sampler handles this
    
    loaders = {
        'train': DataLoader(
            datasets_dict['train'],
            batch_size=batch_size,
            shuffle=train_shuffle,
            sampler=train_sampler,
            num_workers=num_workers,
            pin_memory=True,        # faster GPU transfer
            persistent_workers=True, # keep workers alive between epochs
            prefetch_factor=2,       # prefetch 2 batches per worker
            drop_last=True,          # avoid batch norm issues with last small batch
        ),
        'val': DataLoader(
            datasets_dict['val'],
            batch_size=batch_size * 2,  # can use larger batch for eval (no gradients)
            shuffle=False,
            num_workers=num_workers,
            pin_memory=True,
        ),
        'test': DataLoader(
            datasets_dict['test'],
            batch_size=batch_size * 2,
            shuffle=False,
            num_workers=num_workers,
            pin_memory=True,
        ),
    }
    
    # Print dataset info
    for split, ds in datasets_dict.items():
        print(f"  {split}: {len(ds)} samples, {len(ds.classes)} classes")
    
    return loaders
```

---

## 12.3 Training Pipelines — Production-Grade

### What It Is
A training pipeline is the **automated, reproducible system** that takes data + config and produces a trained model. It handles everything: data loading, model creation, training loop, checkpointing, logging, and error recovery.

### Why It Matters
- Reproducibility — anyone can re-run the exact same experiment
- Error recovery — resume from checkpoint after crashes
- Scalability — easily move from 1 GPU to multi-GPU/multi-node
- Maintainability — clean separation of concerns

### Code Example — Complete Training Pipeline

```python
import os
import time
import json
import yaml
import logging
from pathlib import Path
from dataclasses import dataclass, field, asdict
from typing import Optional, Dict, Any

import torch
import torch.nn as nn
import torch.optim as optim
from torch.utils.data import DataLoader
from torch.cuda.amp import GradScaler, autocast  # mixed precision

# ============================================================
# 1. CONFIGURATION (Type-safe, YAML-backed)
# ============================================================

@dataclass
class TrainingConfig:
    """All training hyperparameters in one place. Serializable to YAML."""
    
    # Model
    model_name: str = 'efficientnet_b0'
    num_classes: int = 10
    pretrained: bool = True
    
    # Optimization
    optimizer: str = 'adamw'
    lr: float = 3e-4
    weight_decay: float = 1e-2
    momentum: float = 0.9
    
    # Training
    epochs: int = 100
    batch_size: int = 64
    gradient_clip: float = 1.0
    accumulation_steps: int = 1  # gradient accumulation
    
    # LR Schedule
    scheduler: str = 'cosine'
    warmup_epochs: int = 5
    min_lr: float = 1e-6
    
    # Regularization
    dropout: float = 0.2
    label_smoothing: float = 0.1
    mixup_alpha: float = 0.2
    
    # Data
    data_dir: str = './data'
    img_size: int = 224
    num_workers: int = 4
    
    # System
    device: str = 'cuda'
    mixed_precision: bool = True
    seed: int = 42
    
    # Checkpointing
    checkpoint_dir: str = './checkpoints'
    save_every_n_epochs: int = 5
    keep_last_n: int = 3
    
    # Experiment
    experiment_name: str = 'default'
    run_name: Optional[str] = None
    tags: list = field(default_factory=list)
    
    @classmethod
    def from_yaml(cls, path: str) -> 'TrainingConfig':
        with open(path) as f:
            config_dict = yaml.safe_load(f)
        return cls(**config_dict)
    
    def to_yaml(self, path: str):
        with open(path, 'w') as f:
            yaml.dump(asdict(self), f, default_flow_style=False)


# ============================================================
# 2. TRAINING PIPELINE
# ============================================================

class Trainer:
    """Production-grade training pipeline."""
    
    def __init__(self, config: TrainingConfig):
        self.config = config
        self.device = torch.device(config.device if torch.cuda.is_available() else 'cpu')
        
        # Setup
        self._set_seed(config.seed)
        self._setup_logging()
        self._setup_model()
        self._setup_optimizer()
        self._setup_scheduler()
        
        # Mixed precision
        self.scaler = GradScaler() if config.mixed_precision else None
        
        # Tracking
        self.current_epoch = 0
        self.global_step = 0
        self.best_val_metric = 0
        self.train_history = []
        
        # Checkpoint directory
        self.ckpt_dir = Path(config.checkpoint_dir) / config.experiment_name
        self.ckpt_dir.mkdir(parents=True, exist_ok=True)
        
        # Save config
        config.to_yaml(str(self.ckpt_dir / 'config.yaml'))
    
    def _set_seed(self, seed: int):
        """Set all random seeds for reproducibility."""
        import random
        random.seed(seed)
        np.random.seed(seed)
        torch.manual_seed(seed)
        torch.cuda.manual_seed_all(seed)
        torch.backends.cudnn.deterministic = True
        torch.backends.cudnn.benchmark = False  # True is faster but non-deterministic
    
    def _setup_logging(self):
        """Configure logging."""
        log_dir = self.ckpt_dir / 'logs'
        log_dir.mkdir(exist_ok=True)
        
        logging.basicConfig(
            level=logging.INFO,
            format='%(asctime)s [%(levelname)s] %(message)s',
            handlers=[
                logging.FileHandler(log_dir / 'training.log'),
                logging.StreamHandler()
            ]
        )
        self.logger = logging.getLogger(__name__)
    
    def _setup_model(self):
        """Initialize model with pretrained weights."""
        import torchvision.models as models
        
        # Get model architecture
        model_fn = getattr(models, self.config.model_name)
        weights = 'IMAGENET1K_V1' if self.config.pretrained else None
        self.model = model_fn(weights=weights)
        
        # Replace classifier head
        if hasattr(self.model, 'classifier'):
            in_features = self.model.classifier[-1].in_features
            self.model.classifier[-1] = nn.Linear(in_features, self.config.num_classes)
        elif hasattr(self.model, 'fc'):
            in_features = self.model.fc.in_features
            self.model.fc = nn.Linear(in_features, self.config.num_classes)
        
        self.model = self.model.to(self.device)
        
        # Log model info
        total_params = sum(p.numel() for p in self.model.parameters())
        trainable_params = sum(p.numel() for p in self.model.parameters() if p.requires_grad)
        self.logger.info(f"Model: {self.config.model_name}")
        self.logger.info(f"Total params: {total_params:,}, Trainable: {trainable_params:,}")
    
    def _setup_optimizer(self):
        """Setup optimizer with proper weight decay handling."""
        # Don't apply weight decay to bias and norm layers
        no_decay = ['bias', 'bn', 'norm', 'LayerNorm']
        param_groups = [
            {
                'params': [p for n, p in self.model.named_parameters() 
                          if not any(nd in n for nd in no_decay) and p.requires_grad],
                'weight_decay': self.config.weight_decay
            },
            {
                'params': [p for n, p in self.model.named_parameters() 
                          if any(nd in n for nd in no_decay) and p.requires_grad],
                'weight_decay': 0.0
            },
        ]
        
        if self.config.optimizer == 'adamw':
            self.optimizer = optim.AdamW(param_groups, lr=self.config.lr)
        elif self.config.optimizer == 'sgd':
            self.optimizer = optim.SGD(
                param_groups, lr=self.config.lr, momentum=self.config.momentum
            )
        else:
            self.optimizer = optim.Adam(param_groups, lr=self.config.lr)
    
    def _setup_scheduler(self):
        """Setup learning rate scheduler."""
        if self.config.scheduler == 'cosine':
            self.scheduler = optim.lr_scheduler.CosineAnnealingLR(
                self.optimizer, T_max=self.config.epochs, eta_min=self.config.min_lr
            )
        elif self.config.scheduler == 'step':
            self.scheduler = optim.lr_scheduler.StepLR(
                self.optimizer, step_size=30, gamma=0.1
            )
        elif self.config.scheduler == 'plateau':
            self.scheduler = optim.lr_scheduler.ReduceLROnPlateau(
                self.optimizer, mode='max', patience=5, factor=0.5
            )
        else:
            self.scheduler = None
    
    def train_one_epoch(self, train_loader: DataLoader) -> Dict[str, float]:
        """Train for one epoch."""
        self.model.train()
        
        running_loss = 0.0
        correct = 0
        total = 0
        
        criterion = nn.CrossEntropyLoss(label_smoothing=self.config.label_smoothing)
        
        for batch_idx, (images, labels) in enumerate(train_loader):
            images = images.to(self.device, non_blocking=True)
            labels = labels.to(self.device, non_blocking=True)
            
            # Mixed precision forward pass
            if self.config.mixed_precision:
                with autocast():
                    outputs = self.model(images)
                    loss = criterion(outputs, labels)
                    loss = loss / self.config.accumulation_steps
                
                self.scaler.scale(loss).backward()
                
                # Gradient accumulation
                if (batch_idx + 1) % self.config.accumulation_steps == 0:
                    # Gradient clipping
                    self.scaler.unscale_(self.optimizer)
                    torch.nn.utils.clip_grad_norm_(
                        self.model.parameters(), self.config.gradient_clip
                    )
                    
                    self.scaler.step(self.optimizer)
                    self.scaler.update()
                    self.optimizer.zero_grad()
            else:
                outputs = self.model(images)
                loss = criterion(outputs, labels)
                loss = loss / self.config.accumulation_steps
                loss.backward()
                
                if (batch_idx + 1) % self.config.accumulation_steps == 0:
                    torch.nn.utils.clip_grad_norm_(
                        self.model.parameters(), self.config.gradient_clip
                    )
                    self.optimizer.step()
                    self.optimizer.zero_grad()
            
            # Track metrics
            running_loss += loss.item() * self.config.accumulation_steps
            _, predicted = outputs.max(1)
            total += labels.size(0)
            correct += predicted.eq(labels).sum().item()
            
            self.global_step += 1
        
        metrics = {
            'train_loss': running_loss / len(train_loader),
            'train_acc': 100.0 * correct / total,
            'lr': self.optimizer.param_groups[0]['lr'],
        }
        
        return metrics
    
    @torch.no_grad()
    def validate(self, val_loader: DataLoader) -> Dict[str, float]:
        """Evaluate on validation set."""
        self.model.eval()
        
        running_loss = 0.0
        correct = 0
        total = 0
        all_preds = []
        all_labels = []
        
        criterion = nn.CrossEntropyLoss()
        
        for images, labels in val_loader:
            images = images.to(self.device, non_blocking=True)
            labels = labels.to(self.device, non_blocking=True)
            
            if self.config.mixed_precision:
                with autocast():
                    outputs = self.model(images)
                    loss = criterion(outputs, labels)
            else:
                outputs = self.model(images)
                loss = criterion(outputs, labels)
            
            running_loss += loss.item()
            _, predicted = outputs.max(1)
            total += labels.size(0)
            correct += predicted.eq(labels).sum().item()
            
            all_preds.extend(predicted.cpu().numpy())
            all_labels.extend(labels.cpu().numpy())
        
        metrics = {
            'val_loss': running_loss / len(val_loader),
            'val_acc': 100.0 * correct / total,
        }
        
        return metrics
    
    def save_checkpoint(self, metrics: Dict[str, float], is_best: bool = False):
        """Save model checkpoint with all training state."""
        checkpoint = {
            'epoch': self.current_epoch,
            'global_step': self.global_step,
            'model_state_dict': self.model.state_dict(),
            'optimizer_state_dict': self.optimizer.state_dict(),
            'scheduler_state_dict': self.scheduler.state_dict() if self.scheduler else None,
            'scaler_state_dict': self.scaler.state_dict() if self.scaler else None,
            'best_val_metric': self.best_val_metric,
            'metrics': metrics,
            'config': asdict(self.config),
        }
        
        # Save periodic checkpoint
        ckpt_path = self.ckpt_dir / f'checkpoint_epoch{self.current_epoch}.pt'
        torch.save(checkpoint, ckpt_path)
        
        # Save best model separately
        if is_best:
            best_path = self.ckpt_dir / 'best_model.pt'
            torch.save(checkpoint, best_path)
            self.logger.info(f"  ★ New best model! Val acc: {metrics['val_acc']:.2f}%")
        
        # Cleanup old checkpoints (keep last N)
        self._cleanup_checkpoints()
    
    def _cleanup_checkpoints(self):
        """Remove old checkpoints, keeping only the most recent N."""
        checkpoints = sorted(
            self.ckpt_dir.glob('checkpoint_epoch*.pt'),
            key=lambda p: p.stat().st_mtime
        )
        
        while len(checkpoints) > self.config.keep_last_n:
            checkpoints[0].unlink()
            checkpoints.pop(0)
    
    def load_checkpoint(self, path: str):
        """Resume training from a checkpoint."""
        checkpoint = torch.load(path, map_location=self.device)
        
        self.model.load_state_dict(checkpoint['model_state_dict'])
        self.optimizer.load_state_dict(checkpoint['optimizer_state_dict'])
        
        if checkpoint['scheduler_state_dict'] and self.scheduler:
            self.scheduler.load_state_dict(checkpoint['scheduler_state_dict'])
        
        if checkpoint['scaler_state_dict'] and self.scaler:
            self.scaler.load_state_dict(checkpoint['scaler_state_dict'])
        
        self.current_epoch = checkpoint['epoch'] + 1
        self.global_step = checkpoint['global_step']
        self.best_val_metric = checkpoint['best_val_metric']
        
        self.logger.info(f"Resumed from epoch {self.current_epoch}")
    
    def fit(self, train_loader: DataLoader, val_loader: DataLoader):
        """Main training loop."""
        self.logger.info(f"\n{'='*60}")
        self.logger.info(f"Starting training: {self.config.experiment_name}")
        self.logger.info(f"{'='*60}")
        self.logger.info(f"Device: {self.device}")
        self.logger.info(f"Epochs: {self.config.epochs}")
        self.logger.info(f"Batch size: {self.config.batch_size}")
        self.logger.info(f"Learning rate: {self.config.lr}")
        self.logger.info(f"Mixed precision: {self.config.mixed_precision}")
        
        start_time = time.time()
        
        for epoch in range(self.current_epoch, self.config.epochs):
            self.current_epoch = epoch
            epoch_start = time.time()
            
            # Train
            train_metrics = self.train_one_epoch(train_loader)
            
            # Validate
            val_metrics = self.validate(val_loader)
            
            # Update scheduler
            if self.scheduler:
                if isinstance(self.scheduler, optim.lr_scheduler.ReduceLROnPlateau):
                    self.scheduler.step(val_metrics['val_acc'])
                else:
                    self.scheduler.step()
            
            # Combine metrics
            metrics = {**train_metrics, **val_metrics}
            self.train_history.append(metrics)
            
            # Check if best
            is_best = val_metrics['val_acc'] > self.best_val_metric
            if is_best:
                self.best_val_metric = val_metrics['val_acc']
            
            # Save checkpoint
            if (epoch + 1) % self.config.save_every_n_epochs == 0 or is_best:
                self.save_checkpoint(metrics, is_best)
            
            # Log progress
            epoch_time = time.time() - epoch_start
            self.logger.info(
                f"Epoch {epoch+1}/{self.config.epochs} "
                f"({epoch_time:.1f}s) — "
                f"Train Loss: {train_metrics['train_loss']:.4f}, "
                f"Train Acc: {train_metrics['train_acc']:.2f}%, "
                f"Val Acc: {val_metrics['val_acc']:.2f}%, "
                f"LR: {train_metrics['lr']:.2e}"
            )
        
        total_time = time.time() - start_time
        self.logger.info(f"\nTraining complete! Total time: {total_time/3600:.2f} hours")
        self.logger.info(f"Best validation accuracy: {self.best_val_metric:.2f}%")
        
        # Save final training history
        history_path = self.ckpt_dir / 'training_history.json'
        history_path.write_text(json.dumps(self.train_history, indent=2))


# ============================================================
# 3. PUTTING IT ALL TOGETHER
# ============================================================

def main():
    """Complete training pipeline execution."""
    
    # Load config
    config = TrainingConfig(
        model_name='efficientnet_b0',
        num_classes=10,
        epochs=50,
        lr=3e-4,
        batch_size=64,
        experiment_name='cifar10_efficientnet',
        mixed_precision=True,
    )
    
    # Create dataloaders
    loaders = create_dataloaders(
        data_dir=config.data_dir,
        batch_size=config.batch_size,
        img_size=config.img_size,
        num_workers=config.num_workers,
    )
    
    # Initialize trainer
    trainer = Trainer(config)
    
    # Optionally resume from checkpoint
    resume_path = Path(config.checkpoint_dir) / config.experiment_name / 'best_model.pt'
    if resume_path.exists():
        trainer.load_checkpoint(str(resume_path))
    
    # Train!
    trainer.fit(loaders['train'], loaders['val'])
    
    # Final evaluation on test set
    test_metrics = trainer.validate(loaders['test'])
    print(f"\nFinal Test Accuracy: {test_metrics['val_acc']:.2f}%")


if __name__ == "__main__":
    main()
```

---

## 12.4 Experiment Tracking

### What It Is
Experiment tracking is the practice of **logging everything** about each training run — hyperparameters, metrics, code version, data version, model artifacts — so you can compare experiments, reproduce results, and understand what worked.

### Why It Matters
- Without tracking, after 50 experiments you'll ask: "Which run got 94% accuracy? What were its settings?"
- Enables team collaboration — everyone sees what's been tried
- Required for auditing in regulated industries (healthcare, finance)
- Makes the difference between scientific ML and "throw stuff at the wall"

### Tools Comparison

| Tool | Best For | Pricing | Key Feature |
|------|----------|---------|-------------|
| **Weights & Biases (W&B)** | Teams, beautiful dashboards | Free (academic), paid (enterprise) | Best visualization |
| **MLflow** | Self-hosted, open-source needs | Free (open source) | Model registry built-in |
| **TensorBoard** | Quick local experiments | Free | Comes with PyTorch |
| **Neptune.ai** | Large teams, metadata management | Free tier + paid | Rich metadata |
| **Comet ML** | Automated tracking | Free tier + paid | Code change tracking |
| **DVC** | Data + model versioning | Free (open source) | Git for data |

### Code Example — Weights & Biases Integration

```python
import wandb
import torch
import torch.nn as nn
from pathlib import Path

class ExperimentTracker:
    """Unified experiment tracking interface (W&B + local)."""
    
    def __init__(self, config: dict, project_name: str, run_name: str = None):
        self.config = config
        
        # Initialize W&B
        self.run = wandb.init(
            project=project_name,
            name=run_name,
            config=config,
            tags=config.get('tags', []),
            notes=config.get('notes', ''),
        )
        
        # Log code
        wandb.run.log_code(".")  # tracks code changes
    
    def log_metrics(self, metrics: dict, step: int = None):
        """Log metrics for current step/epoch."""
        wandb.log(metrics, step=step)
    
    def log_model(self, model: nn.Module, name: str = "model"):
        """Log model architecture summary."""
        # Log model graph
        wandb.watch(model, log='all', log_freq=100)
    
    def log_images(self, images: torch.Tensor, predictions: list, labels: list, 
                   step: int, max_images: int = 16):
        """Log sample predictions as images."""
        wandb_images = []
        for i in range(min(max_images, len(images))):
            img = images[i].permute(1, 2, 0).cpu().numpy()
            # Denormalize
            img = img * [0.229, 0.224, 0.225] + [0.485, 0.456, 0.406]
            img = (img * 255).clip(0, 255).astype('uint8')
            
            caption = f"Pred: {predictions[i]}, True: {labels[i]}"
            wandb_images.append(wandb.Image(img, caption=caption))
        
        wandb.log({"predictions": wandb_images}, step=step)
    
    def log_confusion_matrix(self, y_true, y_pred, class_names):
        """Log confusion matrix."""
        wandb.log({
            "confusion_matrix": wandb.plot.confusion_matrix(
                y_true=y_true,
                preds=y_pred,
                class_names=class_names
            )
        })
    
    def log_artifact(self, path: str, name: str, artifact_type: str = "model"):
        """Log a file as a versioned artifact."""
        artifact = wandb.Artifact(name=name, type=artifact_type)
        artifact.add_file(path)
        self.run.log_artifact(artifact)
    
    def log_table(self, data: list, columns: list, name: str = "results"):
        """Log a data table."""
        table = wandb.Table(data=data, columns=columns)
        wandb.log({name: table})
    
    def finish(self):
        """End the run."""
        wandb.finish()


# --- Usage in Training Loop ---
def train_with_tracking():
    config = {
        'model': 'efficientnet_b0',
        'lr': 3e-4,
        'batch_size': 64,
        'epochs': 50,
        'optimizer': 'adamw',
        'weight_decay': 1e-2,
        'scheduler': 'cosine',
        'augmentation': 'medium',
        'notes': 'Baseline experiment with EfficientNet-B0',
        'tags': ['baseline', 'efficientnet', 'cifar10'],
    }
    
    # Initialize tracking
    tracker = ExperimentTracker(
        config=config,
        project_name='image-classification',
        run_name='efficientnet-b0-baseline'
    )
    
    # ... (model, optimizer, data setup) ...
    
    for epoch in range(config['epochs']):
        # Train
        train_metrics = {'train_loss': 0.5, 'train_acc': 85.0}  # placeholder
        
        # Validate
        val_metrics = {'val_loss': 0.6, 'val_acc': 82.0}  # placeholder
        
        # Log all metrics
        tracker.log_metrics({
            **train_metrics,
            **val_metrics,
            'epoch': epoch,
            'lr': 3e-4,  # current LR
        }, step=epoch)
        
        # Log sample predictions every 10 epochs
        if (epoch + 1) % 10 == 0:
            pass  # tracker.log_images(...)
    
    # Log final model
    # tracker.log_artifact('checkpoints/best_model.pt', 'best-model', 'model')
    
    tracker.finish()


# --- MLflow Alternative ---
def train_with_mlflow():
    """MLflow tracking — better for self-hosted/on-premise."""
    import mlflow
    import mlflow.pytorch
    
    # Set tracking URI (local or remote)
    mlflow.set_tracking_uri("http://localhost:5000")  # or file:///path/to/mlruns
    mlflow.set_experiment("image-classification")
    
    with mlflow.start_run(run_name="efficientnet-baseline"):
        # Log parameters
        mlflow.log_params({
            'model': 'efficientnet_b0',
            'lr': 3e-4,
            'batch_size': 64,
            'epochs': 50,
        })
        
        for epoch in range(50):
            # ... training ...
            
            # Log metrics
            mlflow.log_metrics({
                'train_loss': 0.5,
                'val_acc': 85.0,
            }, step=epoch)
        
        # Log model artifact
        # mlflow.pytorch.log_model(model, "model")
        
        # Log any file
        mlflow.log_artifact("config.yaml")
```

### What to Track — Comprehensive Checklist

```
ALWAYS TRACK:
━━━━━━━━━━━━━
☑ All hyperparameters (lr, batch_size, optimizer, scheduler, etc.)
☑ Training & validation metrics (loss, accuracy, per-class metrics)
☑ Learning rate over time
☑ Training time per epoch
☑ Hardware used (GPU model, memory)
☑ Dataset version / commit hash
☑ Code version (git commit SHA)
☑ Random seed
☑ Model checkpoint (best and last)
☑ Config file used

ALSO USEFUL:
━━━━━━━━━━━
☑ Gradient norms (detect exploding/vanishing)
☑ Sample predictions (visual sanity check)
☑ Confusion matrix (at end)
☑ Loss curves (overfitting detection)
☑ Parameter distributions (weight histograms)
☑ GPU utilization %
☑ Data augmentation examples
```

---

## 12.5 Model Registry and Versioning

### What It Is
A model registry is a **centralized store** for trained models with metadata, versioning, and stage management (development → staging → production). Think of it as "Git for models" — you can track which model version is deployed, roll back to previous versions, and audit the lineage of any model.

### Why It Matters
- Know exactly which model is in production at any time
- Roll back to previous version in seconds if new model fails
- Track model lineage: which data + code + hyperparameters produced it
- Required for compliance in regulated industries

### Model Stages

```
DEVELOPMENT ──→ STAGING ──→ PRODUCTION ──→ ARCHIVED
    │               │            │              │
    │  Experiments  │  Testing   │  Serving     │  Retired
    │  happening    │  A/B tests │  live traffic│  kept for
    │               │  shadow    │              │  audit
    │               │  mode      │              │
    ▼               ▼            ▼              ▼
  Many models    Top 2-3      1 model        Old versions
  being trained  candidates   (+ fallback)   (compliance)
```

### Code Example — Model Registry with MLflow

```python
import mlflow
from mlflow.tracking import MlflowClient
import torch
import json
from pathlib import Path
from datetime import datetime
from typing import Optional, Dict

class ModelRegistry:
    """
    Production model registry.
    Wraps MLflow model registry with additional functionality.
    """
    
    def __init__(self, tracking_uri: str = "http://localhost:5000"):
        mlflow.set_tracking_uri(tracking_uri)
        self.client = MlflowClient()
    
    def register_model(
        self,
        model: torch.nn.Module,
        model_name: str,
        metrics: Dict[str, float],
        config: Dict,
        description: str = "",
    ) -> str:
        """Register a trained model with full metadata."""
        
        with mlflow.start_run() as run:
            # Log parameters
            mlflow.log_params(config)
            
            # Log metrics
            mlflow.log_metrics(metrics)
            
            # Log model
            mlflow.pytorch.log_model(
                model,
                artifact_path="model",
                registered_model_name=model_name,
            )
            
            # Log additional metadata
            metadata = {
                'registered_at': datetime.now().isoformat(),
                'metrics': metrics,
                'config': config,
                'description': description,
                'pytorch_version': torch.__version__,
            }
            mlflow.log_dict(metadata, "metadata.json")
            
            run_id = run.info.run_id
        
        print(f"Model registered: {model_name} (run: {run_id})")
        return run_id
    
    def promote_model(self, model_name: str, version: int, stage: str):
        """
        Move model to a new stage.
        Stages: 'Staging', 'Production', 'Archived'
        """
        self.client.transition_model_version_stage(
            name=model_name,
            version=version,
            stage=stage,
        )
        print(f"Model {model_name} v{version} → {stage}")
    
    def get_production_model(self, model_name: str):
        """Load the current production model."""
        model_uri = f"models:/{model_name}/Production"
        model = mlflow.pytorch.load_model(model_uri)
        return model
    
    def compare_models(self, model_name: str) -> list:
        """Compare all versions of a model."""
        versions = self.client.search_model_versions(f"name='{model_name}'")
        
        comparison = []
        for v in versions:
            run = self.client.get_run(v.run_id)
            comparison.append({
                'version': v.version,
                'stage': v.current_stage,
                'metrics': run.data.metrics,
                'created': v.creation_timestamp,
            })
        
        return sorted(comparison, key=lambda x: x['version'])


# --- Simple File-Based Registry (No MLflow dependency) ---

class SimpleModelRegistry:
    """
    Lightweight model registry using just the filesystem.
    Good for small teams or when MLflow is overkill.
    """
    
    def __init__(self, registry_dir: str = "./model_registry"):
        self.registry_dir = Path(registry_dir)
        self.registry_dir.mkdir(parents=True, exist_ok=True)
        self.index_file = self.registry_dir / "registry_index.json"
        self._load_index()
    
    def _load_index(self):
        if self.index_file.exists():
            self.index = json.loads(self.index_file.read_text())
        else:
            self.index = {"models": {}}
    
    def _save_index(self):
        self.index_file.write_text(json.dumps(self.index, indent=2))
    
    def register(
        self,
        model: torch.nn.Module,
        model_name: str,
        version: str,
        metrics: Dict[str, float],
        config: Dict,
        tags: list = None,
    ):
        """Register a model version."""
        model_dir = self.registry_dir / model_name / version
        model_dir.mkdir(parents=True, exist_ok=True)
        
        # Save model weights
        torch.save(model.state_dict(), model_dir / "model.pt")
        
        # Save metadata
        metadata = {
            "model_name": model_name,
            "version": version,
            "registered_at": datetime.now().isoformat(),
            "metrics": metrics,
            "config": config,
            "tags": tags or [],
            "stage": "development",
        }
        (model_dir / "metadata.json").write_text(json.dumps(metadata, indent=2))
        
        # Update index
        if model_name not in self.index["models"]:
            self.index["models"][model_name] = {"versions": [], "production": None}
        
        self.index["models"][model_name]["versions"].append(version)
        self._save_index()
        
        print(f"✓ Registered: {model_name} v{version} (val_acc={metrics.get('val_acc', 'N/A')})")
    
    def promote_to_production(self, model_name: str, version: str):
        """Set a version as the production model."""
        # Verify it exists
        model_dir = self.registry_dir / model_name / version
        if not model_dir.exists():
            raise ValueError(f"Version {version} not found for {model_name}")
        
        # Update stage
        metadata_path = model_dir / "metadata.json"
        metadata = json.loads(metadata_path.read_text())
        metadata["stage"] = "production"
        metadata["promoted_at"] = datetime.now().isoformat()
        metadata_path.write_text(json.dumps(metadata, indent=2))
        
        # Update index
        old_prod = self.index["models"][model_name].get("production")
        self.index["models"][model_name]["production"] = version
        self._save_index()
        
        print(f"✓ {model_name} v{version} → PRODUCTION (was: {old_prod})")
    
    def load_production_model(self, model_name: str, model_class, **model_kwargs):
        """Load the current production model."""
        version = self.index["models"][model_name]["production"]
        if not version:
            raise ValueError(f"No production model for {model_name}")
        
        model_path = self.registry_dir / model_name / version / "model.pt"
        
        model = model_class(**model_kwargs)
        model.load_state_dict(torch.load(model_path, map_location='cpu'))
        model.eval()
        
        return model, version
    
    def list_models(self):
        """List all registered models and their versions."""
        for name, info in self.index["models"].items():
            prod = info.get("production", "none")
            versions = info.get("versions", [])
            print(f"\n{name}:")
            print(f"  Versions: {', '.join(versions)}")
            print(f"  Production: {prod}")


# Usage
registry = SimpleModelRegistry("./model_registry")

# After training
# registry.register(model, "image_classifier", "v1.0.0", 
#                   metrics={'val_acc': 94.5}, config={'lr': 3e-4, ...})
# registry.promote_to_production("image_classifier", "v1.0.0")
# model, version = registry.load_production_model("image_classifier", MyModel, num_classes=10)
```

---

## 12.6 Debugging and Monitoring Training

### What It Is
Training a deep learning model isn't "set and forget." You need to actively **monitor** the training process to catch issues early — gradient problems, overfitting, data issues, or hardware failures.

### Key Signals to Monitor

```
HEALTHY TRAINING:                    UNHEALTHY TRAINING:
━━━━━━━━━━━━━━━━━                   ━━━━━━━━━━━━━━━━━━━

Loss ▼                               Loss ▼
│╲                                   │     ╱────── (diverging!)
│  ╲                                 │    ╱
│   ╲____                            │   ╱
│        ╲___                        │  ╱
│            ╲___                    │ ╱
└──────────── epoch                  └──────────── epoch

Train/Val gap ▼                      Train/Val gap ▼
│  train ╲                           │  train ╲
│   ╲     val                        │         ╲___________
│    ╲  ╲                            │  val ──────────────── (overfitting!)
│     ╲╲                             │
│      ╲                             │
└──────────── epoch                  └──────────── epoch
```

### Code Example — Training Monitor

```python
import torch
import numpy as np
from collections import defaultdict
from typing import Dict, List
import warnings

class TrainingMonitor:
    """
    Monitor training health and detect common issues early.
    Attach to your training loop for real-time diagnostics.
    """
    
    def __init__(self, patience: int = 10, min_delta: float = 1e-4):
        self.patience = patience
        self.min_delta = min_delta
        self.history = defaultdict(list)
        self.gradient_history = []
        self.alerts = []
    
    def log_epoch(self, metrics: Dict[str, float]):
        """Log metrics for the current epoch."""
        for key, value in metrics.items():
            self.history[key].append(value)
        
        # Run diagnostics
        self._check_overfitting()
        self._check_loss_plateau()
        self._check_lr_too_high()
        self._check_nan()
    
    def log_gradients(self, model: torch.nn.Module):
        """Log gradient statistics (call after loss.backward())."""
        total_norm = 0
        param_norms = {}
        
        for name, param in model.named_parameters():
            if param.grad is not None:
                param_norm = param.grad.data.norm(2).item()
                total_norm += param_norm ** 2
                param_norms[name] = param_norm
        
        total_norm = total_norm ** 0.5
        self.gradient_history.append(total_norm)
        
        # Check for gradient issues
        if total_norm > 100:
            self.alerts.append(f"⚠️  EXPLODING GRADIENTS: norm={total_norm:.1f}")
        elif total_norm < 1e-7:
            self.alerts.append(f"⚠️  VANISHING GRADIENTS: norm={total_norm:.2e}")
        
        return total_norm
    
    def _check_overfitting(self):
        """Detect overfitting: train improves but val doesn't."""
        if len(self.history.get('train_loss', [])) < 10:
            return
        
        train_losses = self.history['train_loss'][-10:]
        val_losses = self.history.get('val_loss', [None])[-10:]
        
        if None in val_losses:
            return
        
        train_improving = train_losses[-1] < train_losses[0]
        val_worsening = val_losses[-1] > val_losses[0] * 1.05
        
        if train_improving and val_worsening:
            gap = val_losses[-1] - train_losses[-1]
            self.alerts.append(
                f"⚠️  OVERFITTING DETECTED: Train-Val gap = {gap:.4f}. "
                f"Consider: more augmentation, dropout, weight decay, or less capacity."
            )
    
    def _check_loss_plateau(self):
        """Detect loss plateau."""
        losses = self.history.get('val_loss', [])
        if len(losses) < self.patience:
            return
        
        recent = losses[-self.patience:]
        improvement = max(recent) - min(recent)
        
        if improvement < self.min_delta:
            self.alerts.append(
                f"⚠️  PLATEAU: Val loss hasn't improved in {self.patience} epochs. "
                f"Consider: reducing LR, changing optimizer, or stopping early."
            )
    
    def _check_lr_too_high(self):
        """Detect unstable training from too-high learning rate."""
        losses = self.history.get('train_loss', [])
        if len(losses) < 3:
            return
        
        # Loss increasing
        if losses[-1] > losses[-2] > losses[-3]:
            if losses[-1] > losses[-3] * 1.5:
                self.alerts.append(
                    f"⚠️  UNSTABLE TRAINING: Loss increasing rapidly. "
                    f"Reduce learning rate!"
                )
    
    def _check_nan(self):
        """Check for NaN values."""
        for key, values in self.history.items():
            if values and (np.isnan(values[-1]) or np.isinf(values[-1])):
                self.alerts.append(
                    f"🚨 NaN/Inf DETECTED in {key}! "
                    f"Check: gradient clipping, learning rate, data normalization."
                )
    
    def get_diagnostics(self) -> str:
        """Get current training diagnostics report."""
        report = "\n" + "="*50 + "\nTRAINING DIAGNOSTICS\n" + "="*50 + "\n"
        
        if self.alerts:
            report += "\n⚠️  ALERTS:\n"
            for alert in self.alerts[-5:]:  # last 5 alerts
                report += f"  {alert}\n"
        else:
            report += "\n✅ All checks passed — training looks healthy!\n"
        
        # Summary stats
        if self.gradient_history:
            report += f"\nGradient norm — "
            report += f"Mean: {np.mean(self.gradient_history[-100:]):.4f}, "
            report += f"Max: {np.max(self.gradient_history[-100:]):.4f}\n"
        
        return report


# Usage in training loop:
# monitor = TrainingMonitor(patience=10)
# 
# for epoch in range(epochs):
#     train_metrics = train_one_epoch(...)
#     val_metrics = validate(...)
#     
#     monitor.log_epoch({**train_metrics, **val_metrics})
#     
#     # Log gradients (in training step, after backward)
#     # grad_norm = monitor.log_gradients(model)
#     
#     if (epoch + 1) % 5 == 0:
#         print(monitor.get_diagnostics())
```

### Common Training Failure Modes

| Symptom | Likely Cause | Fix |
|---------|-------------|-----|
| Loss = NaN | LR too high, numerical overflow | Reduce LR, add gradient clipping, check for log(0) |
| Loss doesn't decrease | LR too low, bug in code | Increase LR, verify data loading, check labels |
| Val loss increases while train decreases | Overfitting | Add regularization (dropout, augment, weight decay) |
| Loss oscillates wildly | LR too high for batch size | Reduce LR or increase batch size |
| Train acc ~= random (1/num_classes) | Labels shuffled, model not learning | Verify data pipeline end-to-end |
| GPU util < 50% | Data loading bottleneck | Increase num_workers, use pin_memory |

---

## 12.7 Reproducibility

### What It Is
Reproducibility means getting the **exact same results** when re-running an experiment with the same settings. In deep learning, this is surprisingly hard due to GPU non-determinism, random initialization, data shuffling, etc.

### Why It Matters
- Scientific validity — results must be verifiable
- Debugging — need to reproduce a bug to fix it
- Compliance — auditors may require exact reproduction
- Collaboration — teammates need to verify your results

### Code Example — Reproducibility Toolkit

```python
import os
import random
import hashlib
import subprocess
import json
import platform
from datetime import datetime

import numpy as np
import torch

def set_deterministic(seed: int = 42):
    """
    Make training as deterministic as possible.
    Note: some operations remain non-deterministic on GPU.
    """
    # Python
    random.seed(seed)
    os.environ['PYTHONHASHSEED'] = str(seed)
    
    # NumPy
    np.random.seed(seed)
    
    # PyTorch
    torch.manual_seed(seed)
    torch.cuda.manual_seed(seed)
    torch.cuda.manual_seed_all(seed)  # for multi-GPU
    
    # CuDNN determinism (slower but reproducible)
    torch.backends.cudnn.deterministic = True
    torch.backends.cudnn.benchmark = False
    
    # PyTorch 2.0+ deterministic algorithms
    torch.use_deterministic_algorithms(True, warn_only=True)
    
    # Environment variable for CUDA determinism
    os.environ['CUBLAS_WORKSPACE_CONFIG'] = ':4096:8'


def get_environment_info() -> dict:
    """Capture full environment information for reproducibility."""
    info = {
        'timestamp': datetime.now().isoformat(),
        'platform': platform.platform(),
        'python_version': platform.python_version(),
        'pytorch_version': torch.__version__,
        'cuda_version': torch.version.cuda if torch.cuda.is_available() else 'N/A',
        'cudnn_version': str(torch.backends.cudnn.version()) if torch.cuda.is_available() else 'N/A',
        'gpu': torch.cuda.get_device_name(0) if torch.cuda.is_available() else 'N/A',
        'gpu_count': torch.cuda.device_count(),
        'numpy_version': np.__version__,
    }
    
    # Git information
    try:
        info['git_commit'] = subprocess.check_output(
            ['git', 'rev-parse', 'HEAD'], stderr=subprocess.DEVNULL
        ).decode().strip()
        info['git_branch'] = subprocess.check_output(
            ['git', 'rev-parse', '--abbrev-ref', 'HEAD'], stderr=subprocess.DEVNULL
        ).decode().strip()
        info['git_dirty'] = bool(subprocess.check_output(
            ['git', 'status', '--porcelain'], stderr=subprocess.DEVNULL
        ).decode().strip())
    except (subprocess.CalledProcessError, FileNotFoundError):
        info['git_commit'] = 'N/A'
    
    return info


def compute_data_hash(data_dir: str) -> str:
    """Compute a hash of the dataset for version tracking."""
    hasher = hashlib.sha256()
    
    for root, dirs, files in os.walk(data_dir):
        dirs.sort()  # ensure consistent ordering
        for filename in sorted(files):
            filepath = os.path.join(root, filename)
            # Hash filename + size (faster than hashing contents)
            hasher.update(filepath.encode())
            hasher.update(str(os.path.getsize(filepath)).encode())
    
    return hasher.hexdigest()[:16]


class ReproducibilityManager:
    """Manages all reproducibility artifacts for an experiment."""
    
    def __init__(self, experiment_dir: str, seed: int = 42):
        self.experiment_dir = experiment_dir
        self.seed = seed
        os.makedirs(experiment_dir, exist_ok=True)
        
        # Set deterministic mode
        set_deterministic(seed)
        
        # Save environment
        self.env_info = get_environment_info()
        self.env_info['seed'] = seed
    
    def save_experiment_card(self, config: dict, data_dir: str = None):
        """Save a complete 'experiment card' for reproducibility."""
        card = {
            'environment': self.env_info,
            'config': config,
            'data_hash': compute_data_hash(data_dir) if data_dir else 'N/A',
        }
        
        card_path = os.path.join(self.experiment_dir, 'experiment_card.json')
        with open(card_path, 'w') as f:
            json.dump(card, f, indent=2)
        
        print(f"Experiment card saved: {card_path}")
        return card
    
    def save_requirements(self):
        """Save Python package versions."""
        try:
            result = subprocess.run(
                ['pip', 'freeze'], capture_output=True, text=True
            )
            req_path = os.path.join(self.experiment_dir, 'requirements.txt')
            with open(req_path, 'w') as f:
                f.write(result.stdout)
        except Exception:
            pass


# Usage:
# repro = ReproducibilityManager('./experiments/exp_001', seed=42)
# repro.save_experiment_card(config=training_config, data_dir='./data')
# repro.save_requirements()
```

### Reproducibility Checklist

```
BEFORE TRAINING:
☑ Set all random seeds (Python, NumPy, PyTorch, CUDA)
☑ Set CUDNN deterministic mode
☑ Save git commit hash
☑ Save exact package versions (pip freeze)
☑ Save dataset hash/version
☑ Save full config (YAML/JSON)

DURING TRAINING:
☑ Use deterministic data loading (set worker seeds)
☑ Save checkpoints with optimizer state
☑ Log all random operations

AFTER TRAINING:
☑ Save experiment card with all metadata
☑ Verify reproducibility by running same config twice
☑ Document any known sources of non-determinism
```

> **Warning**: Perfect reproducibility across different GPU architectures is NOT possible. Results may differ between, e.g., A100 and V100 due to different floating-point implementations. Always report hardware in results.

---

## 12.8 From Prototype to Production

### The Production Readiness Checklist

```
PROTOTYPE → PRODUCTION GAP:
━━━━━━━━━━━━━━━━━━━━━━━━━━

Prototype (Jupyter):          Production (System):
├── Single file               ├── Modular codebase
├── Hardcoded paths           ├── Config-driven
├── No error handling         ├── Comprehensive error handling
├── Manual execution          ├── Automated pipelines
├── "It works on my machine"  ├── Containerized (Docker)
├── No monitoring             ├── Full observability
├── No tests                  ├── Unit + integration tests
└── Ad-hoc data handling      └── Data validation + versioning
```

### Model Export and Optimization

```python
import torch
import torch.onnx
import time

class ModelExporter:
    """Export models for production deployment."""
    
    @staticmethod
    def to_torchscript(model, sample_input, save_path):
        """
        Export to TorchScript (production inference).
        TorchScript removes Python dependency.
        """
        model.eval()
        
        # Method 1: Tracing (works for most models)
        traced = torch.jit.trace(model, sample_input)
        traced.save(save_path)
        
        # Verify
        with torch.no_grad():
            original_output = model(sample_input)
            traced_output = traced(sample_input)
            max_diff = (original_output - traced_output).abs().max().item()
            print(f"TorchScript export — Max diff: {max_diff:.2e}")
        
        return traced
    
    @staticmethod
    def to_onnx(model, sample_input, save_path, opset_version=13):
        """
        Export to ONNX (cross-framework interop).
        Can run on TensorRT, ONNX Runtime, OpenVINO.
        """
        model.eval()
        
        torch.onnx.export(
            model,
            sample_input,
            save_path,
            export_params=True,
            opset_version=opset_version,
            do_constant_folding=True,  # optimize constant ops
            input_names=['input'],
            output_names=['output'],
            dynamic_axes={
                'input': {0: 'batch_size'},   # dynamic batch
                'output': {0: 'batch_size'},
            }
        )
        
        # Verify with ONNX Runtime
        import onnxruntime as ort
        session = ort.InferenceSession(save_path)
        
        input_np = sample_input.numpy()
        ort_output = session.run(None, {'input': input_np})[0]
        
        with torch.no_grad():
            torch_output = model(sample_input).numpy()
        
        max_diff = np.abs(torch_output - ort_output).max()
        print(f"ONNX export — Max diff: {max_diff:.2e}")
    
    @staticmethod
    def benchmark_inference(model, sample_input, num_runs=100, device='cuda'):
        """Benchmark model inference speed."""
        model = model.to(device).eval()
        sample_input = sample_input.to(device)
        
        # Warmup
        with torch.no_grad():
            for _ in range(10):
                model(sample_input)
        
        # Benchmark
        if device == 'cuda':
            torch.cuda.synchronize()
        
        start = time.time()
        with torch.no_grad():
            for _ in range(num_runs):
                output = model(sample_input)
        
        if device == 'cuda':
            torch.cuda.synchronize()
        
        elapsed = (time.time() - start) / num_runs
        fps = sample_input.shape[0] / elapsed
        
        print(f"Inference benchmark ({device}):")
        print(f"  Batch size: {sample_input.shape[0]}")
        print(f"  Latency: {elapsed*1000:.2f} ms")
        print(f"  Throughput: {fps:.1f} images/sec")
        
        return elapsed, fps


# Usage:
# model = load_model(...)
# sample = torch.randn(1, 3, 224, 224)
# 
# ModelExporter.to_torchscript(model, sample, 'model_scripted.pt')
# ModelExporter.to_onnx(model, sample, 'model.onnx')
# ModelExporter.benchmark_inference(model, sample.expand(32, -1, -1, -1))
```

### Docker Template for DL Models

```python
# Dockerfile content (for reference):
dockerfile_content = """
# --- Dockerfile for DL Model Serving ---
FROM pytorch/pytorch:2.0.0-cuda11.7-cudnn8-runtime

WORKDIR /app

# Install system dependencies
RUN apt-get update && apt-get install -y --no-install-recommends \\
    libgl1-mesa-glx \\
    && rm -rf /var/lib/apt/lists/*

# Install Python dependencies
COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt

# Copy model and code
COPY model/ ./model/
COPY src/ ./src/
COPY config.yaml .

# Create non-root user (security!)
RUN useradd -m appuser && chown -R appuser /app
USER appuser

# Expose port for serving
EXPOSE 8000

# Health check
HEALTHCHECK --interval=30s --timeout=5s --retries=3 \\
    CMD curl -f http://localhost:8000/health || exit 1

# Start server
CMD ["python", "src/serve.py", "--config", "config.yaml"]
"""

print(dockerfile_content)
```

---

## 12.9 Common Mistakes

| Mistake | Problem | Fix |
|---------|---------|-----|
| **Training on test set** | Data leakage, inflated metrics | Strict train/val/test split, never touch test until final eval |
| **Not versioning data** | "What data did that model use?" | Use DVC or simple hash-based versioning |
| **No experiment tracking** | Lost in a sea of experiments | Use W&B, MLflow, or even spreadsheets — anything! |
| **Hardcoded paths** | Code breaks on other machines | Use config files, environment variables, relative paths |
| **No checkpointing** | Lose everything on crash | Save checkpoint every N epochs with optimizer state |
| **Ignoring data quality** | Model learns noise | Validate data before training (corrupt files, wrong labels) |
| **No monitoring in production** | Silent model degradation | Track prediction distributions, latency, error rates |
| **Deploying notebook code** | Unstructured, fragile | Refactor into proper Python modules before deployment |
| **No fallback model** | Single point of failure | Always keep previous model version as fallback |
| **Skip documentation** | Future you won't remember | Document decisions, failed approaches, and trade-offs |

### Pro Tips

> **Tip 1**: Create a **model card** for every production model — who trained it, what data, known limitations, bias analysis, intended use cases.

> **Tip 2**: Use **shadow deployments** — run new model in parallel with production model, compare outputs, but only serve from old model. Catches issues before they affect users.

> **Tip 3**: Implement **automated retraining triggers** — when data drift is detected or performance drops below threshold, automatically kick off retraining pipeline.

> **Tip 4**: The best teams spend 80% on data quality and 20% on modeling. Most beginners do the opposite.

---

## 12.10 Interview Questions

### Conceptual Questions

**Q1: Walk me through how you would set up a deep learning project from scratch for a production system.**
> 1) Start with data: validate quality, version it, create proper splits. 2) Set up experiment tracking (W&B/MLflow) from day 1. 3) Build a simple baseline first (pretrained model, minimal processing). 4) Create a reproducible training pipeline with config management. 5) Iterate on model/data/augmentation, tracking everything. 6) Once satisfied, register the best model, export to ONNX/TorchScript. 7) Deploy with monitoring (latency, drift, accuracy on served data). 8) Set up automated retraining when performance drops.

**Q2: How do you ensure reproducibility in deep learning experiments?**
> Set all random seeds (Python, NumPy, PyTorch), use `torch.backends.cudnn.deterministic = True`, save git commit hash and pip freeze, version your data (hash or DVC), save full config as YAML, save checkpoints with optimizer state. Accept that perfect reproducibility across different GPUs is impossible — document hardware. Use the same Docker image for different runs.

**Q3: What's the difference between model checkpointing and model registry?**
> Checkpointing saves model state during training for recovery (resume after crashes) — it's training infrastructure. A model registry stores finalized, validated models with metadata and stage management (dev/staging/production) — it's deployment infrastructure. Checkpoints are ephemeral; registered models are permanent with versioning and lineage.

**Q4: How do you handle class imbalance in a production DL pipeline?**
> Multi-pronged approach: 1) Weighted sampling (WeightedRandomSampler) during training. 2) Class-weighted loss function (higher penalty for minority class errors). 3) Data augmentation focused on minority classes. 4) Evaluation using class-balanced metrics (macro F1, not accuracy). 5) Consider SMOTE/synthetic data for tabular, not recommended for images. 6) Monitor per-class performance in production.

**Q5: You notice your model's accuracy dropped 5% in production. How do you debug this?**
> 1) Check for data drift — has the input distribution changed? (Compare feature distributions). 2) Check for data quality issues — corrupt data, missing features, upstream pipeline changes. 3) Check infrastructure — model serving correctly? Memory issues? 4) Look at which classes/segments degraded. 5) Compare recent predictions to training data distribution. 6) If data drift confirmed, evaluate if retraining on recent data fixes it. 7) Check if a dependency updated (library version change).

**Q6: What metrics would you monitor for a deployed image classification model?**
> **System metrics**: latency (p50, p95, p99), throughput, GPU utilization, memory usage, error rate. **Model metrics**: prediction confidence distribution (sudden drop = drift), class distribution of predictions (sudden shift = data change), accuracy on sampled+labeled production data. **Data metrics**: input feature distributions, image resolution/quality stats, null/corrupt input rate. Alert if any metric deviates >2σ from baseline.

---

## 12.11 Quick Reference

### Project Structure Template

```
my_dl_project/
├── configs/
│   ├── base.yaml              # default config
│   ├── experiment_001.yaml    # override for specific experiment
│   └── production.yaml        # production settings
├── data/
│   ├── raw/                   # untouched original data
│   ├── processed/             # cleaned, split data
│   └── data_version.json      # dataset metadata
├── src/
│   ├── data/
│   │   ├── dataset.py         # Dataset class
│   │   ├── transforms.py      # augmentation pipelines
│   │   └── validation.py      # data quality checks
│   ├── models/
│   │   ├── architectures.py   # model definitions
│   │   └── losses.py          # custom loss functions
│   ├── training/
│   │   ├── trainer.py         # training loop
│   │   └── callbacks.py       # logging, checkpointing
│   ├── evaluation/
│   │   ├── metrics.py         # evaluation functions
│   │   └── visualization.py   # result plotting
│   └── serving/
│       ├── export.py          # ONNX/TorchScript export
│       └── serve.py           # inference server
├── notebooks/                  # exploration only
├── tests/                      # unit tests
├── scripts/
│   ├── train.py               # entry point
│   └── evaluate.py            # evaluation script
├── Dockerfile
├── requirements.txt
├── Makefile                    # common commands
└── README.md
```

### Key Tools Summary

| Stage | Tool | Purpose |
|-------|------|---------|
| Data versioning | DVC | Track data + model files with Git |
| Experiment tracking | W&B / MLflow | Log metrics, params, artifacts |
| Training | PyTorch Lightning | Boilerplate-free training |
| HP Tuning | Optuna | Smart hyperparameter search |
| Model export | ONNX / TorchScript | Production-ready inference |
| Serving | TorchServe / Triton | Model serving at scale |
| Monitoring | Prometheus + Grafana | System & model metrics |
| CI/CD | GitHub Actions | Automated testing & deployment |
| Containers | Docker | Reproducible environments |

### One-Page Cheat Sheet

```
DL PROJECT LIFECYCLE — ESSENTIALS:
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

1. DATA:    Validate → Clean → Version → Split (stratified)
2. TRAIN:   Config-driven → Mixed precision → Gradient clip → Checkpoint
3. TRACK:   Log EVERYTHING → Compare experiments → Reproduce
4. EVAL:    Per-class metrics → Confusion matrix → Error analysis
5. DEPLOY:  Export (ONNX) → Containerize → Monitor → Fallback ready
6. MONITOR: Drift detection → Alert → Auto-retrain trigger

Key Commands:
  Train:    python scripts/train.py --config configs/exp_001.yaml
  Eval:     python scripts/evaluate.py --checkpoint best_model.pt
  Export:   python src/serving/export.py --format onnx
  Serve:    docker run -p 8000:8000 my-model:v1.2.0
```

---

*End of Deep Learning Chapter 12 — You now have a complete toolkit for taking DL models from idea to production.*
