---
name: model-training
description: Train machine learning models end-to-end, covering data loading, preprocessing, architecture selection, training loops, validation, and checkpointing.
license: MIT
metadata:
  author: AI Agent Skills
  version: 1.0.0
---

# Model Training

This skill enables an AI agent to train machine learning models on structured or unstructured datasets. It covers the full training lifecycle: loading and preprocessing data, defining model architectures, configuring optimizers and loss functions, running training loops with validation, applying learning rate scheduling, and saving checkpoints. The agent can handle both classical ML and deep learning workflows across frameworks like PyTorch, TensorFlow, and scikit-learn.

## Workflow

1. **Load and inspect data:** Read the dataset from disk, database, or remote storage. Profile the data to understand feature distributions, class balance, missing values, and data types. Split into training, validation, and test sets using stratified sampling when class imbalance is present.

2. **Preprocess and transform:** Apply feature engineering such as normalization, standardization, tokenization (for text), or augmentation (for images). Build preprocessing pipelines that are reproducible and serializable so the same transforms apply at inference time.

3. **Define model architecture:** Select or construct the model architecture appropriate for the task. For classical ML, choose estimators like gradient boosting or SVMs. For deep learning, define layers, activation functions, and regularization such as dropout or weight decay. When transfer learning is applicable, load a pre-trained backbone and attach task-specific heads.

4. **Configure training:** Set the optimizer (Adam, SGD, AdamW), loss function (cross-entropy, MSE, focal loss), learning rate schedule (cosine annealing, step decay, warmup), and batch size. Enable mixed precision training with `torch.amp` or `tf.keras.mixed_precision` when training on GPUs to reduce memory usage and speed up computation.

5. **Execute training loop with validation:** Train for the specified number of epochs, logging training loss and metrics per batch or epoch. Evaluate on the validation set at regular intervals. Implement early stopping to halt training when validation performance plateaus for a configurable number of epochs (patience).

6. **Checkpoint and export:** Save model checkpoints at the best validation score and at regular intervals. Export the final model in a portable format (ONNX, TorchScript, SavedModel) for downstream deployment. Log all hyperparameters and metrics to an experiment tracker like MLflow or Weights & Biases.

## Supported Technologies

- **Frameworks:** PyTorch, TensorFlow/Keras, scikit-learn, XGBoost, LightGBM
- **Distributed training:** PyTorch DDP, Horovod, TensorFlow MirroredStrategy
- **Experiment tracking:** MLflow, Weights & Biases, TensorBoard
- **Mixed precision:** `torch.amp`, `tf.keras.mixed_precision`
- **Data loading:** PyTorch DataLoader, tf.data, pandas, Hugging Face Datasets

## Usage

Provide the agent with the dataset location, the target variable or task description, and any constraints (framework preference, compute budget, target metric). The agent will execute the full training workflow and return a trained model artifact along with evaluation metrics.

## Examples

### Example 1: Training a Text Classifier with PyTorch

```python
import torch
import torch.nn as nn
from torch.utils.data import DataLoader, TensorDataset
from sklearn.model_selection import train_test_split
from sklearn.preprocessing import LabelEncoder
from collections import Counter

# Simulated tokenized text data: 2000 samples, sequence length 50, vocab size 5000
X = torch.randint(0, 5000, (2000, 50))
y_raw = ["positive"] * 1000 + ["negative"] * 1000
le = LabelEncoder()
y = torch.tensor(le.fit_transform(y_raw), dtype=torch.long)

X_train, X_val, y_train, y_val = train_test_split(X, y, test_size=0.2, stratify=y, random_state=42)
train_loader = DataLoader(TensorDataset(X_train, y_train), batch_size=64, shuffle=True)
val_loader = DataLoader(TensorDataset(X_val, y_val), batch_size=64)

class TextClassifier(nn.Module):
    def __init__(self, vocab_size=5000, embed_dim=128, num_classes=2):
        super().__init__()
        self.embedding = nn.Embedding(vocab_size, embed_dim, padding_idx=0)
        self.lstm = nn.LSTM(embed_dim, 64, batch_first=True, bidirectional=True)
        self.dropout = nn.Dropout(0.3)
        self.fc = nn.Linear(128, num_classes)

    def forward(self, x):
        x = self.embedding(x)
        _, (hidden, _) = self.lstm(x)
        hidden = torch.cat((hidden[-2], hidden[-1]), dim=1)
        return self.fc(self.dropout(hidden))

device = torch.device("cuda" if torch.cuda.is_available() else "cpu")
model = TextClassifier().to(device)
optimizer = torch.optim.AdamW(model.parameters(), lr=1e-3, weight_decay=1e-2)
scheduler = torch.optim.lr_scheduler.CosineAnnealingLR(optimizer, T_max=10)
criterion = nn.CrossEntropyLoss()

best_val_acc, patience, patience_counter = 0.0, 3, 0
for epoch in range(10):
    model.train()
    for xb, yb in train_loader:
        xb, yb = xb.to(device), yb.to(device)
        loss = criterion(model(xb), yb)
        optimizer.zero_grad()
        loss.backward()
        optimizer.step()
    scheduler.step()

    model.eval()
    correct, total = 0, 0
    with torch.no_grad():
        for xb, yb in val_loader:
            xb, yb = xb.to(device), yb.to(device)
            correct += (model(xb).argmax(1) == yb).sum().item()
            total += yb.size(0)
    val_acc = correct / total
    print(f"Epoch {epoch+1}: val_acc={val_acc:.4f}")

    if val_acc > best_val_acc:
        best_val_acc = val_acc
        torch.save(model.state_dict(), "best_model.pt")
        patience_counter = 0
    else:
        patience_counter += 1
        if patience_counter >= patience:
            print("Early stopping triggered.")
            break
```

### Example 2: Fine-Tuning a Pre-Trained Model with Hugging Face Transformers

```python
from datasets import load_dataset
from transformers import AutoTokenizer, AutoModelForSequenceClassification, TrainingArguments, Trainer
import numpy as np
from sklearn.metrics import accuracy_score, f1_score

dataset = load_dataset("imdb")
tokenizer = AutoTokenizer.from_pretrained("distilbert-base-uncased")

def tokenize(batch):
    return tokenizer(batch["text"], padding="max_length", truncation=True, max_length=256)

tokenized = dataset.map(tokenize, batched=True)
tokenized.set_format("torch", columns=["input_ids", "attention_mask", "label"])

model = AutoModelForSequenceClassification.from_pretrained("distilbert-base-uncased", num_labels=2)

def compute_metrics(eval_pred):
    preds = np.argmax(eval_pred.predictions, axis=1)
    return {"accuracy": accuracy_score(eval_pred.label_ids, preds), "f1": f1_score(eval_pred.label_ids, preds)}

training_args = TrainingArguments(
    output_dir="./results", num_train_epochs=3, per_device_train_batch_size=16,
    per_device_eval_batch_size=32, eval_strategy="epoch", save_strategy="epoch",
    load_best_model_at_end=True, metric_for_best_model="f1", fp16=True,
    learning_rate=2e-5, weight_decay=0.01, warmup_steps=500, logging_steps=100,
)

trainer = Trainer(model=model, args=training_args, train_dataset=tokenized["train"],
                  eval_dataset=tokenized["test"], compute_metrics=compute_metrics)
trainer.train()
trainer.save_model("./best_model")
```

## Best Practices

- **Always stratify splits** when dealing with imbalanced datasets to ensure each split reflects the true class distribution.
- **Use learning rate warmup** for fine-tuning pre-trained models to avoid catastrophic forgetting in early training steps.
- **Enable mixed precision** (`fp16` or `bf16`) on GPU training to cut memory usage roughly in half and accelerate throughput.
- **Log everything** to an experiment tracker — hyperparameters, metrics per epoch, system resource usage, and the git hash of the training code.
- **Save checkpoints frequently** and always keep the best-validation checkpoint to avoid losing progress from crashes or overtraining.
- **Validate on a held-out set** that was never used during training or hyperparameter selection to get an unbiased estimate of generalization.

## Edge Cases

- **Small datasets (< 1000 samples):** Use k-fold cross-validation instead of a single train/val split. Prefer transfer learning or pre-trained models over training from scratch.
- **Extreme class imbalance (> 100:1 ratio):** Use oversampling (SMOTE), class-weighted loss functions, or focal loss. Evaluation should rely on F1, precision-recall AUC, or Matthews correlation coefficient rather than accuracy.
- **Training divergence or NaN loss:** Reduce the learning rate, apply gradient clipping (`torch.nn.utils.clip_grad_norm_`), check for data issues like infinite values, or disable mixed precision to rule out numerical instability.
- **Out-of-memory errors:** Reduce batch size, enable gradient accumulation, use mixed precision, or switch to gradient checkpointing to trade compute for memory.
- **Non-stationary data (concept drift):** Implement periodic retraining on fresh data, use time-based train/val splits rather than random splits, and monitor production metrics for degradation.
