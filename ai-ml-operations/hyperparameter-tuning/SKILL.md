---
name: hyperparameter-tuning
description: Optimize machine learning model hyperparameters using grid search, random search, Bayesian optimization, and Hyperband to maximize model performance within a compute budget.
license: MIT
metadata:
  author: AI Agent Skills
  version: 1.0.0
---

# Hyperparameter Tuning

This skill enables an AI agent to systematically search for optimal hyperparameter configurations for machine learning models. It covers defining search spaces, selecting search strategies (grid, random, Bayesian, Hyperband), running trials with cross-validation, applying early stopping to prune poor configurations, and analyzing results to identify the best-performing parameters. The agent balances exploration and exploitation to find strong configurations within a given computational budget.

## Workflow

1. **Define the search space:** Specify each hyperparameter with its type (categorical, integer, float) and range. Use log-uniform distributions for parameters that span orders of magnitude (e.g., learning rate from 1e-5 to 1e-1). Group related parameters and define conditional search spaces where certain parameters only apply when others take specific values.

2. **Select the search strategy:** Choose the tuning algorithm based on compute budget and search space size. Grid search is exhaustive but only feasible for small spaces. Random search is a strong baseline that scales better. Bayesian optimization (Tree-structured Parzen Estimators or Gaussian Processes) is most sample-efficient for expensive evaluations. Hyperband and ASHA combine early stopping with random search for deep learning workloads.

3. **Configure evaluation:** Set up k-fold cross-validation (typically 5-fold) for reliable performance estimates on small to medium datasets. For large datasets or expensive models, use a single holdout validation set. Define the objective metric to optimize (e.g., validation F1, AUC-ROC, RMSE) and whether to minimize or maximize it.

4. **Run trials with pruning:** Execute the search, launching trials in parallel when possible. Enable pruning to terminate underperforming trials early based on intermediate results (e.g., after a few epochs of training), freeing compute for more promising configurations.

5. **Analyze and select results:** Inspect the optimization history to understand which hyperparameters matter most (importance analysis). Visualize parameter interactions with contour plots or parallel coordinate plots. Select the best configuration and retrain the final model on the full training set with those parameters.

## Supported Technologies

- **Frameworks:** Optuna, Ray Tune, scikit-learn GridSearchCV/RandomizedSearchCV, Hyperopt, Keras Tuner
- **Pruning algorithms:** Median pruning, Hyperband (Successive Halving), ASHA
- **Bayesian methods:** TPE (Tree-structured Parzen Estimators), GP (Gaussian Process), CMA-ES
- **Visualization:** Optuna visualization (plotly), TensorBoard HParams, Weights & Biases Sweeps
- **Distributed execution:** Ray Tune cluster, Optuna with distributed storage (MySQL, PostgreSQL)

## Usage

Provide the agent with the model, dataset, the hyperparameters to tune with their ranges, a compute budget (number of trials or wall-clock time), and the target metric. The agent will execute the tuning workflow and return the best hyperparameter configuration along with performance analysis.

## Examples

### Example 1: Optuna Study for Tuning a Random Forest

```python
import optuna
from sklearn.ensemble import RandomForestClassifier
from sklearn.datasets import load_breast_cancer
from sklearn.model_selection import cross_val_score
import numpy as np

X, y = load_breast_cancer(return_X_y=True)

def objective(trial):
    params = {
        "n_estimators": trial.suggest_int("n_estimators", 50, 500, step=50),
        "max_depth": trial.suggest_int("max_depth", 3, 30),
        "min_samples_split": trial.suggest_int("min_samples_split", 2, 20),
        "min_samples_leaf": trial.suggest_int("min_samples_leaf", 1, 10),
        "max_features": trial.suggest_categorical("max_features", ["sqrt", "log2", None]),
        "criterion": trial.suggest_categorical("criterion", ["gini", "entropy"]),
    }
    clf = RandomForestClassifier(**params, random_state=42, n_jobs=-1)
    scores = cross_val_score(clf, X, y, cv=5, scoring="f1")
    return scores.mean()

study = optuna.create_study(direction="maximize", sampler=optuna.samplers.TPESampler(seed=42))
study.optimize(objective, n_trials=100, show_progress_bar=True)

print(f"Best F1: {study.best_value:.4f}")
print(f"Best params: {study.best_params}")

# Visualization
fig_importance = optuna.visualization.plot_param_importances(study)
fig_history = optuna.visualization.plot_optimization_history(study)
fig_contour = optuna.visualization.plot_contour(study, params=["n_estimators", "max_depth"])
```

### Example 2: Ray Tune for Neural Network with Early Stopping

```python
import torch
import torch.nn as nn
from torch.utils.data import DataLoader, TensorDataset, random_split
from ray import tune
from ray.tune.schedulers import ASHAScheduler
from ray.air import session
import numpy as np

def train_nn(config):
    X = torch.randn(2000, 20)
    y = (X[:, 0] + X[:, 1] * 2 > 0).long()
    dataset = TensorDataset(X, y)
    train_set, val_set = random_split(dataset, [1600, 400])
    train_loader = DataLoader(train_set, batch_size=config["batch_size"], shuffle=True)
    val_loader = DataLoader(val_set, batch_size=256)

    model = nn.Sequential(
        nn.Linear(20, config["hidden_size"]),
        nn.ReLU(),
        nn.Dropout(config["dropout"]),
        nn.Linear(config["hidden_size"], config["hidden_size"] // 2),
        nn.ReLU(),
        nn.Linear(config["hidden_size"] // 2, 2),
    )
    optimizer = torch.optim.Adam(model.parameters(), lr=config["lr"], weight_decay=config["weight_decay"])
    criterion = nn.CrossEntropyLoss()

    for epoch in range(50):
        model.train()
        for xb, yb in train_loader:
            loss = criterion(model(xb), yb)
            optimizer.zero_grad()
            loss.backward()
            optimizer.step()

        model.eval()
        correct, total = 0, 0
        with torch.no_grad():
            for xb, yb in val_loader:
                correct += (model(xb).argmax(1) == yb).sum().item()
                total += yb.size(0)
        session.report({"val_accuracy": correct / total})

search_space = {
    "hidden_size": tune.choice([64, 128, 256]),
    "lr": tune.loguniform(1e-4, 1e-1),
    "dropout": tune.uniform(0.1, 0.5),
    "batch_size": tune.choice([32, 64, 128]),
    "weight_decay": tune.loguniform(1e-5, 1e-2),
}

scheduler = ASHAScheduler(max_t=50, grace_period=5, reduction_factor=3)
result = tune.run(
    train_nn,
    config=search_space,
    num_samples=50,
    scheduler=scheduler,
    metric="val_accuracy",
    mode="max",
    resources_per_trial={"cpu": 2},
)

print(f"Best config: {result.best_config}")
print(f"Best val accuracy: {result.best_result['val_accuracy']:.4f}")
```

## Best Practices

- **Use log-uniform distributions** for learning rate, weight decay, and regularization strength since optimal values often span multiple orders of magnitude.
- **Start with random search** to quickly identify promising regions of the search space before switching to Bayesian optimization for fine-grained exploration.
- **Enable early stopping / pruning** to avoid wasting compute on configurations that clearly underperform after a few epochs.
- **Always use cross-validation** for the objective score on small datasets (< 50k samples) to reduce variance in performance estimates and avoid overfitting to a single validation split.
- **Run hyperparameter importance analysis** after tuning to understand which parameters actually matter — often only 2-3 parameters drive most of the performance difference.
- **Set a compute budget upfront** (number of trials, GPU-hours, or wall-clock time) and choose the search strategy that makes the best use of that budget.

## Edge Cases

- **Huge search spaces (> 10 dimensions):** Bayesian optimization degrades with high dimensionality. Use random search or Hyperband as a first pass, then run Bayesian optimization on the top 3-5 most important parameters identified from the first pass.
- **Noisy objectives:** When cross-validation scores have high variance, a single trial result is unreliable. Increase the number of CV folds, use repeated k-fold, or average over multiple seeds before comparing configurations.
- **Correlated hyperparameters:** Some hyperparameters interact strongly (e.g., learning rate and batch size). Use Optuna's contour plots or fANOVA importance to detect interactions and consider tuning correlated groups together.
- **Expensive evaluations (> 1 hour per trial):** Use multi-fidelity methods like Hyperband that train with small budgets first and only promote promising configurations to full training. Also consider surrogate benchmarks or smaller proxy datasets for initial screening.
- **Categorical explosion:** When multiple categorical hyperparameters create a combinatorial explosion, use conditional search spaces to prune invalid combinations and reduce the effective space size.
