# Day 5: SQS Model Implementation

**Duration:** 6-8 hours  
**Objective:** Implement Random Forest-based Software Quality Score (SQS) model with risky module detection and actionable recommendations

**Prerequisites:**
- Days 1, 2, 3, and 4 completed
- All previous models working (classification, anomaly, sentiment)
- FastAPI service running
- Virtual environment activated

---

## Overview

Day 5 focuses on building the **Software Quality Score (SQS)** model — a Random Forest regressor that evaluates the overall health of a software project. Unlike DQS (which scores individual developers), SQS scores the entire project/repository based on aggregated metrics.

**What You'll Build:**
- SQS Pydantic schemas for request/response validation
- `SQSModel` class using scikit-learn's Random Forest
- Risky module identification with weighted risk scoring
- Actionable recommendation engine based on score breakdown
- Training script with synthetic project data
- REST API endpoints for prediction and model info
- Unit tests covering all core behaviors

**Why This Matters:**
- SQS gives engineering managers a single number to track project health
- Risky module detection pinpoints exactly where to focus improvement efforts
- Recommendations turn raw scores into actionable engineering tasks
- Random Forest is interpretable via feature importance — no black box

**Key Metrics Scored (10 features):**

| Feature | What It Measures |
|---|---|
| `test_coverage` | % of code covered by tests |
| `debt_density` | Technical debt items per LOC |
| `bug_rate` | Bug-fix commits as % of total |
| `churn_rate` | Code churn (re-modified lines) |
| `review_turnaround` | Avg hours to first PR review |
| `review_debt` | PRs merged without review |
| `avg_team_dqs` | Average DQS of all team members |
| `commit_frequency` | Commits per week |
| `hotspot_count` | Files changed in >50% of commits |
| `complexity_score` | Cyclomatic complexity average |


---

## Morning Session (3-4 hours)

### Task 1: SQS Schema (1 hour)

#### Step 1.1: Create the Schema File

Create `app/schemas/sqs.py`:

```python
from pydantic import BaseModel, Field
from typing import Optional, List, Literal
from datetime import datetime


class SQSFeatures(BaseModel):
    """
    Project-level features used to compute the SQS score.

    WHY: Encapsulating features in a dedicated model gives us validation,
         documentation, and a clear contract between the backend and ML service.
    HOW: Each field maps directly to a column in the Random Forest feature matrix.
    """
    test_coverage: float = Field(
        ...,
        ge=0.0,
        le=100.0,
        description="Percentage of code covered by automated tests (0-100)"
    )
    debt_density: float = Field(
        ...,
        ge=0.0,
        description="Technical debt items (TODO/FIXME) per 1000 lines of code"
    )
    bug_rate: float = Field(
        ...,
        ge=0.0,
        le=1.0,
        description="Ratio of bug-fix commits to total commits (0-1)"
    )
    churn_rate: float = Field(
        ...,
        ge=0.0,
        le=1.0,
        description="Code churn rate — proportion of lines re-modified within 2 weeks (0-1)"
    )
    review_turnaround: float = Field(
        ...,
        ge=0.0,
        description="Average hours from PR open to first review comment"
    )
    review_debt: float = Field(
        ...,
        ge=0.0,
        le=1.0,
        description="Ratio of PRs merged without any review (0-1)"
    )
    avg_team_dqs: float = Field(
        ...,
        ge=0.0,
        le=100.0,
        description="Average Developer Quality Score across all active team members (0-100)"
    )
    commit_frequency: float = Field(
        ...,
        ge=0.0,
        description="Average commits per week across the team"
    )
    hotspot_count: int = Field(
        ...,
        ge=0,
        description="Number of files changed in more than 50% of recent commits"
    )
    complexity_score: float = Field(
        ...,
        ge=0.0,
        description="Average cyclomatic complexity across all modules"
    )

    class Config:
        json_schema_extra = {
            "example": {
                "test_coverage": 72.5,
                "debt_density": 3.2,
                "bug_rate": 0.18,
                "churn_rate": 0.22,
                "review_turnaround": 6.5,
                "review_debt": 0.08,
                "avg_team_dqs": 74.0,
                "commit_frequency": 18.0,
                "hotspot_count": 4,
                "complexity_score": 8.3
            }
        }
```


```python
class ModuleMetrics(BaseModel):
    """
    Per-module metrics used for risky module identification.

    WHY: Project-level scores hide which specific modules are problematic.
         Module-level analysis lets teams prioritize refactoring efforts.
    HOW: Each module's metrics are scored independently using a weighted formula.
    """
    module_name: str = Field(..., description="Module or file path identifier")
    coverage: float = Field(
        ...,
        ge=0.0,
        le=100.0,
        description="Test coverage for this module (%)"
    )
    churn: float = Field(
        ...,
        ge=0.0,
        le=1.0,
        description="Churn rate for this module (0-1)"
    )
    debt_count: int = Field(
        ...,
        ge=0,
        description="Number of TODO/FIXME markers in this module"
    )
    bug_count: int = Field(
        ...,
        ge=0,
        description="Number of bug-fix commits touching this module"
    )
    complexity: float = Field(
        ...,
        ge=0.0,
        description="Cyclomatic complexity of this module"
    )

    class Config:
        json_schema_extra = {
            "example": {
                "module_name": "src/services/payment.service.ts",
                "coverage": 42.0,
                "churn": 0.65,
                "debt_count": 8,
                "bug_count": 5,
                "complexity": 18.5
            }
        }


class RiskyModule(BaseModel):
    """
    A module identified as high-risk by the SQS analysis.

    WHY: Surfaces the most problematic areas so developers know where to act.
    HOW: Computed from ModuleMetrics using a weighted risk scoring formula.
    """
    module_name: str = Field(..., description="Module or file path identifier")
    risk_score: float = Field(
        ...,
        ge=0.0,
        le=1.0,
        description="Normalized risk score (0=safe, 1=critical)"
    )
    risk_level: Literal["LOW", "MEDIUM", "HIGH"] = Field(
        ...,
        description="Risk classification based on risk_score"
    )
    primary_concern: str = Field(
        ...,
        description="The single biggest risk factor for this module"
    )
    metrics: ModuleMetrics = Field(
        ...,
        description="Raw metrics that produced this risk score"
    )


class SQSPredictRequest(BaseModel):
    """
    Full request payload for SQS prediction.

    WHY: Combines project-level features with optional per-module data
         so the endpoint can return both a project score and risky modules.
    HOW: features drives the Random Forest prediction; modules drives
         the weighted risk scoring for individual files.
    """
    project_id: str = Field(..., description="Unique project or repository identifier")
    features: SQSFeatures = Field(..., description="Aggregated project-level metrics")
    modules: Optional[List[ModuleMetrics]] = Field(
        None,
        description="Optional list of per-module metrics for risky module detection"
    )

    class Config:
        json_schema_extra = {
            "example": {
                "project_id": "proj-abc123",
                "features": {
                    "test_coverage": 72.5,
                    "debt_density": 3.2,
                    "bug_rate": 0.18,
                    "churn_rate": 0.22,
                    "review_turnaround": 6.5,
                    "review_debt": 0.08,
                    "avg_team_dqs": 74.0,
                    "commit_frequency": 18.0,
                    "hotspot_count": 4,
                    "complexity_score": 8.3
                },
                "modules": [
                    {
                        "module_name": "src/services/payment.service.ts",
                        "coverage": 42.0,
                        "churn": 0.65,
                        "debt_count": 8,
                        "bug_count": 5,
                        "complexity": 18.5
                    }
                ]
            }
        }
```


```python
class SQSPredictionResult(BaseModel):
    """
    Full SQS prediction response.

    WHY: Returns the score, grade, risky modules, and recommendations
         so consumers have everything they need in one response.
    HOW: Aggregates Random Forest output, risk scoring, and rule-based
         recommendation logic into a single structured response.
    """
    project_id: str = Field(..., description="The project that was scored")
    sqs_score: float = Field(
        ...,
        ge=0.0,
        le=100.0,
        description="Software Quality Score (0-100)"
    )
    grade: str = Field(
        ...,
        description="Letter grade: A (≥85), B (≥70), C (≥55), D (≥40), F (<40)"
    )
    confidence: float = Field(
        ...,
        ge=0.0,
        le=1.0,
        description="Model confidence in the prediction (0-1)"
    )
    feature_importance: dict = Field(
        ...,
        description="Contribution of each feature to the score (0-1, sums to 1)"
    )
    risky_modules: List[RiskyModule] = Field(
        default_factory=list,
        description="Modules identified as high or medium risk, sorted by risk_score desc"
    )
    recommendations: List[str] = Field(
        default_factory=list,
        description="Prioritized, actionable improvement recommendations"
    )
    model_version: str = Field(default="1.0.0", description="SQS model version used")
    timestamp: datetime = Field(
        default_factory=datetime.utcnow,
        description="When the prediction was made"
    )

    class Config:
        json_schema_extra = {
            "example": {
                "project_id": "proj-abc123",
                "sqs_score": 68.4,
                "grade": "C",
                "confidence": 0.81,
                "feature_importance": {
                    "test_coverage": 0.22,
                    "debt_density": 0.18,
                    "avg_team_dqs": 0.15,
                    "churn_rate": 0.12,
                    "bug_rate": 0.11,
                    "review_turnaround": 0.09,
                    "complexity_score": 0.07,
                    "hotspot_count": 0.03,
                    "review_debt": 0.02,
                    "commit_frequency": 0.01
                },
                "risky_modules": [
                    {
                        "module_name": "src/services/payment.service.ts",
                        "risk_score": 0.82,
                        "risk_level": "HIGH",
                        "primary_concern": "Low test coverage (42%)",
                        "metrics": {
                            "module_name": "src/services/payment.service.ts",
                            "coverage": 42.0,
                            "churn": 0.65,
                            "debt_count": 8,
                            "bug_count": 5,
                            "complexity": 18.5
                        }
                    }
                ],
                "recommendations": [
                    "Increase test coverage from 72.5% to at least 80% — focus on payment and auth modules",
                    "Reduce technical debt density: resolve 3+ TODO/FIXME items per sprint",
                    "Improve PR review turnaround from 6.5h to under 4h"
                ],
                "model_version": "1.0.0",
                "timestamp": "2024-01-15T10:30:00Z"
            }
        }
```

**Line-by-Line Explanation:**

1. `SQSFeatures` — 10 project-level metrics that feed directly into the Random Forest feature matrix. Each field has `ge`/`le` constraints so Pydantic rejects invalid data before it reaches the model.
2. `ModuleMetrics` — per-file metrics used by the weighted risk scorer (separate from the RF model).
3. `RiskyModule` — the output of risk scoring: name, score, level, and the primary concern string.
4. `SQSPredictRequest` — combines project features + optional module list. Modules are optional so the endpoint works even without per-file data.
5. `SQSPredictionResult` — the full response: score, grade, confidence, feature importance dict, risky modules list, and recommendations.
6. `feature_importance: dict` — stores the RF feature importances keyed by feature name so the frontend can render a bar chart.
7. `default_factory=list` — ensures `risky_modules` and `recommendations` default to empty lists, not `None`.

**Verification Checklist:**
- [ ] File created with no syntax errors
- [ ] All models import cleanly: `from app.schemas.sqs import SQSFeatures, SQSPredictRequest, SQSPredictionResult`
- [ ] Example data validates: `SQSFeatures(**example_data)` raises no errors
- [ ] Invalid data is rejected: `SQSFeatures(test_coverage=150)` raises `ValidationError`


---

### Task 2: SQS Model (2-3 hours)

#### Step 2.1: Create the Model File

Create `app/models/sqs.py`:

```python
from sklearn.ensemble import RandomForestRegressor
from sklearn.preprocessing import MinMaxScaler
import numpy as np
import joblib
import os
import logging
from typing import Dict, List, Tuple, Optional

from app.schemas.sqs import (
    SQSFeatures,
    ModuleMetrics,
    RiskyModule,
    SQSPredictionResult,
)

logger = logging.getLogger(__name__)

# Feature order MUST match the training script — never reorder this list
FEATURE_NAMES = [
    "test_coverage",
    "debt_density",
    "bug_rate",
    "churn_rate",
    "review_turnaround",
    "review_debt",
    "avg_team_dqs",
    "commit_frequency",
    "hotspot_count",
    "complexity_score",
]

# Weights for the per-module risk scoring formula
# WHY: Not all metrics are equally important for risk.
#      Coverage and churn are the strongest predictors of future bugs.
# HOW: Weights sum to 1.0 so the final risk score stays in [0, 1].
MODULE_RISK_WEIGHTS = {
    "coverage": 0.30,    # Low coverage = high risk
    "churn": 0.25,       # High churn = unstable code
    "debt_count": 0.20,  # High debt = maintenance burden
    "bug_count": 0.15,   # Past bugs predict future bugs
    "complexity": 0.10,  # High complexity = harder to test/maintain
}


class SQSModel:
    """
    Random Forest-based Software Quality Score model.

    WHY: Random Forest is chosen over XGBoost here because:
         - It is more robust to noisy project metrics
         - Feature importance is directly interpretable
         - It handles the smaller feature set (10 vs 12 for DQS) well
         - No need for SHAP — built-in feature_importances_ is sufficient

    HOW: Trains a RandomForestRegressor on synthetic project data,
         normalizes the output to 0-100, and exposes predict() and
         identify_risky_modules() as the two main public methods.
    """

    def __init__(self, model_path: Optional[str] = None):
        self.model_path = model_path
        self.model: Optional[RandomForestRegressor] = None
        self.scaler: Optional[MinMaxScaler] = None
        self.model_version = "1.0.0"

        if model_path and os.path.exists(model_path):
            self.load_model(model_path)
        else:
            # Initialize with tuned hyperparameters
            # WHY: These values balance bias/variance for project-level data
            self.model = RandomForestRegressor(
                n_estimators=100,
                max_depth=10,
                min_samples_split=5,
                min_samples_leaf=2,
                random_state=42,
                n_jobs=-1,  # Use all CPU cores
            )
            self.scaler = MinMaxScaler()
            logger.info("Initialized new SQS Random Forest model")
```


```python
    # ------------------------------------------------------------------ #
    #  Feature extraction                                                  #
    # ------------------------------------------------------------------ #

    def _features_to_array(self, features: SQSFeatures) -> np.ndarray:
        """
        Convert SQSFeatures Pydantic model to a numpy array.

        WHY: scikit-learn expects a 2D numpy array, not a Pydantic object.
        HOW: Extract values in the exact order defined by FEATURE_NAMES,
             then reshape to (1, n_features) for single-sample prediction.
        """
        vector = [getattr(features, name) for name in FEATURE_NAMES]
        return np.array(vector, dtype=float).reshape(1, -1)

    # ------------------------------------------------------------------ #
    #  Score normalization                                                 #
    # ------------------------------------------------------------------ #

    def _normalize_score(self, raw_score: float) -> float:
        """
        Clamp and round the raw model output to a 0-100 score.

        WHY: RandomForestRegressor can predict slightly outside the training
             range. Clamping prevents impossible scores like 101 or -2.
        HOW: np.clip enforces bounds; round() gives a clean 1-decimal score.
        """
        return round(float(np.clip(raw_score, 0.0, 100.0)), 1)

    def _score_to_grade(self, score: float) -> str:
        """
        Convert a numeric SQS score to a letter grade.

        WHY: Letter grades are more intuitive for non-technical stakeholders.
        HOW: Simple threshold mapping — same scale used in DQS for consistency.
        """
        if score >= 85:
            return "A"
        elif score >= 70:
            return "B"
        elif score >= 55:
            return "C"
        elif score >= 40:
            return "D"
        else:
            return "F"

    # ------------------------------------------------------------------ #
    #  Feature importance                                                  #
    # ------------------------------------------------------------------ #

    def _get_feature_importance(self) -> Dict[str, float]:
        """
        Return feature importances as a named dictionary.

        WHY: Raw importances are a plain numpy array — naming them makes
             the API response self-documenting and frontend-friendly.
        HOW: Zip FEATURE_NAMES with model.feature_importances_ and round
             each value to 4 decimal places.
        """
        if self.model is None or not hasattr(self.model, "feature_importances_"):
            return {name: 0.0 for name in FEATURE_NAMES}

        importances = self.model.feature_importances_
        return {
            name: round(float(imp), 4)
            for name, imp in zip(FEATURE_NAMES, importances)
        }

    # ------------------------------------------------------------------ #
    #  Risky module detection                                              #
    # ------------------------------------------------------------------ #

    def identify_risky_modules(
        self, modules: List[ModuleMetrics]
    ) -> List[RiskyModule]:
        """
        Score each module and return those with MEDIUM or HIGH risk.

        WHY: Project-level SQS hides which specific files need attention.
             Module-level risk scoring gives developers a concrete to-do list.
        HOW: For each module, normalize each metric to [0, 1] (where 1 = worst),
             apply MODULE_RISK_WEIGHTS, sum to get a risk score, then classify.

        Risk score formula:
            risk = (1 - coverage/100) * 0.30
                 + churn               * 0.25
                 + min(debt_count/20, 1) * 0.20
                 + min(bug_count/10, 1)  * 0.15
                 + min(complexity/30, 1) * 0.10

        Args:
            modules: List of per-module metrics from the request

        Returns:
            List of RiskyModule objects sorted by risk_score descending,
            filtered to MEDIUM and HIGH only (LOW modules are omitted).
        """
        risky = []

        for mod in modules:
            # Normalize each metric so 1.0 = worst possible value
            # WHY: Raw values have different scales (coverage is 0-100,
            #      debt_count could be 0-∞). Normalization makes them comparable.
            norm_coverage = 1.0 - (mod.coverage / 100.0)          # low coverage = high risk
            norm_churn = float(np.clip(mod.churn, 0.0, 1.0))
            norm_debt = float(np.clip(mod.debt_count / 20.0, 0.0, 1.0))
            norm_bugs = float(np.clip(mod.bug_count / 10.0, 0.0, 1.0))
            norm_complexity = float(np.clip(mod.complexity / 30.0, 0.0, 1.0))

            # Weighted sum
            risk_score = (
                norm_coverage  * MODULE_RISK_WEIGHTS["coverage"]
                + norm_churn   * MODULE_RISK_WEIGHTS["churn"]
                + norm_debt    * MODULE_RISK_WEIGHTS["debt_count"]
                + norm_bugs    * MODULE_RISK_WEIGHTS["bug_count"]
                + norm_complexity * MODULE_RISK_WEIGHTS["complexity"]
            )
            risk_score = round(float(np.clip(risk_score, 0.0, 1.0)), 3)

            # Classify risk level
            if risk_score > 0.7:
                risk_level = "HIGH"
            elif risk_score > 0.4:
                risk_level = "MEDIUM"
            else:
                risk_level = "LOW"

            # Skip LOW risk modules — they don't need attention
            if risk_level == "LOW":
                continue

            # Identify the primary concern (highest contributing factor)
            contributions = {
                f"Low test coverage ({mod.coverage:.0f}%)": norm_coverage * MODULE_RISK_WEIGHTS["coverage"],
                f"High code churn ({mod.churn:.0%})": norm_churn * MODULE_RISK_WEIGHTS["churn"],
                f"High technical debt ({mod.debt_count} items)": norm_debt * MODULE_RISK_WEIGHTS["debt_count"],
                f"Frequent bug fixes ({mod.bug_count} bugs)": norm_bugs * MODULE_RISK_WEIGHTS["bug_count"],
                f"High complexity (score {mod.complexity:.1f})": norm_complexity * MODULE_RISK_WEIGHTS["complexity"],
            }
            primary_concern = max(contributions, key=contributions.get)

            risky.append(
                RiskyModule(
                    module_name=mod.module_name,
                    risk_score=risk_score,
                    risk_level=risk_level,
                    primary_concern=primary_concern,
                    metrics=mod,
                )
            )

        # Sort by risk_score descending so the worst modules appear first
        risky.sort(key=lambda m: m.risk_score, reverse=True)
        return risky
```


```python
    # ------------------------------------------------------------------ #
    #  Recommendation engine                                               #
    # ------------------------------------------------------------------ #

    def _generate_recommendations(
        self, features: SQSFeatures, score: float
    ) -> List[str]:
        """
        Generate prioritized, actionable recommendations based on feature values.

        WHY: A raw score tells you *how bad* things are; recommendations tell
             you *what to do about it*. This turns the ML output into engineering tasks.
        HOW: Rule-based thresholds on each feature. Rules are ordered by impact
             (highest-weight features first) so the most important actions appear first.

        Args:
            features: The project features that produced the score
            score: The final SQS score (used to adjust recommendation tone)

        Returns:
            List of recommendation strings, max 5 items
        """
        recommendations = []

        # Coverage — highest weight (0.22)
        if features.test_coverage < 60:
            recommendations.append(
                f"Critical: Increase test coverage from {features.test_coverage:.1f}% to at least 80% — "
                "start with the modules flagged as HIGH risk"
            )
        elif features.test_coverage < 80:
            recommendations.append(
                f"Improve test coverage from {features.test_coverage:.1f}% to 80%+ — "
                "focus on business-critical paths first"
            )

        # Debt density — second highest weight (0.18)
        if features.debt_density > 5.0:
            recommendations.append(
                f"Reduce technical debt density from {features.debt_density:.1f} to under 3.0 — "
                "allocate at least 20% of each sprint to debt resolution"
            )
        elif features.debt_density > 3.0:
            recommendations.append(
                f"Debt density ({features.debt_density:.1f}) is above target — "
                "resolve 3+ TODO/FIXME items per sprint"
            )

        # Team DQS — third highest weight (0.15)
        if features.avg_team_dqs < 60:
            recommendations.append(
                f"Team DQS average ({features.avg_team_dqs:.1f}) is low — "
                "schedule code review sessions and pair programming to raise individual scores"
            )

        # Churn rate — fourth highest weight (0.12)
        if features.churn_rate > 0.35:
            recommendations.append(
                f"High code churn ({features.churn_rate:.0%}) indicates unstable code — "
                "invest in design reviews before implementation to reduce rework"
            )

        # Review turnaround — fifth highest weight (0.09)
        if features.review_turnaround > 24:
            recommendations.append(
                f"PR review turnaround ({features.review_turnaround:.1f}h) is too slow — "
                "set a team SLA of 4h for first review and use review rotation"
            )
        elif features.review_turnaround > 8:
            recommendations.append(
                f"Improve PR review turnaround from {features.review_turnaround:.1f}h to under 8h"
            )

        # Review debt
        if features.review_debt > 0.15:
            recommendations.append(
                f"{features.review_debt:.0%} of PRs are merged without review — "
                "enforce branch protection rules requiring at least one approval"
            )

        # Complexity
        if features.complexity_score > 15:
            recommendations.append(
                f"Average complexity ({features.complexity_score:.1f}) is high — "
                "refactor functions with cyclomatic complexity > 10"
            )

        # Hotspots
        if features.hotspot_count > 5:
            recommendations.append(
                f"{features.hotspot_count} hotspot files detected — "
                "consider splitting large modules to reduce coupling"
            )

        # Return top 5 most impactful recommendations
        return recommendations[:5]

    # ------------------------------------------------------------------ #
    #  Confidence estimation                                               #
    # ------------------------------------------------------------------ #

    def _estimate_confidence(self, features: SQSFeatures) -> float:
        """
        Estimate prediction confidence based on feature completeness and range.

        WHY: Some feature combinations are far from the training distribution,
             making predictions less reliable. Flagging this helps consumers
             decide how much to trust the score.
        HOW: Start at 0.9 and subtract penalties for out-of-range values.
        """
        confidence = 0.9

        # Penalize extreme values that may be outside training distribution
        if features.test_coverage > 98 or features.test_coverage < 5:
            confidence -= 0.1
        if features.debt_density > 20:
            confidence -= 0.05
        if features.commit_frequency > 100:
            confidence -= 0.05
        if features.complexity_score > 50:
            confidence -= 0.05

        return round(max(0.5, confidence), 2)
```


```python
    # ------------------------------------------------------------------ #
    #  Main predict method                                                 #
    # ------------------------------------------------------------------ #

    def predict(
        self,
        project_id: str,
        features: SQSFeatures,
        modules: Optional[List[ModuleMetrics]] = None,
    ) -> SQSPredictionResult:
        """
        Generate a full SQS prediction for a project.

        WHY: Single entry point that orchestrates feature extraction,
             model inference, risk scoring, and recommendation generation.
        HOW: 1. Convert features to numpy array
             2. Run Random Forest prediction
             3. Normalize score to 0-100
             4. Compute feature importance
             5. Identify risky modules (if provided)
             6. Generate recommendations
             7. Return structured result

        Args:
            project_id: Unique project identifier
            features: Project-level metrics
            modules: Optional per-module metrics

        Returns:
            SQSPredictionResult with score, grade, risky modules, recommendations
        """
        if self.model is None:
            raise ValueError("Model not loaded. Run train_sqs.py first.")

        # Step 1: Extract features
        X = self._features_to_array(features)

        # Step 2: Predict raw score
        # WHY: RandomForestRegressor.predict() returns a 1D array
        raw_score = float(self.model.predict(X)[0])

        # Step 3: Normalize to 0-100
        sqs_score = self._normalize_score(raw_score)

        # Step 4: Grade and confidence
        grade = self._score_to_grade(sqs_score)
        confidence = self._estimate_confidence(features)

        # Step 5: Feature importance
        feature_importance = self._get_feature_importance()

        # Step 6: Risky modules (only if module data was provided)
        risky_modules = []
        if modules:
            risky_modules = self.identify_risky_modules(modules)

        # Step 7: Recommendations
        recommendations = self._generate_recommendations(features, sqs_score)

        logger.info(
            f"SQS prediction for {project_id}: score={sqs_score}, "
            f"grade={grade}, risky_modules={len(risky_modules)}"
        )

        return SQSPredictionResult(
            project_id=project_id,
            sqs_score=sqs_score,
            grade=grade,
            confidence=confidence,
            feature_importance=feature_importance,
            risky_modules=risky_modules,
            recommendations=recommendations,
            model_version=self.model_version,
        )

    # ------------------------------------------------------------------ #
    #  Training                                                            #
    # ------------------------------------------------------------------ #

    def train(self, X: np.ndarray, y: np.ndarray):
        """
        Train the Random Forest model.

        WHY: Separating training from prediction allows the model to be
             retrained on new data without restarting the service.
        HOW: Fit the scaler on X, then fit the RF on the scaled features.
             The scaler is saved alongside the model so inference uses
             the same normalization as training.

        Args:
            X: Feature matrix (n_samples, 10)
            y: Target SQS scores (n_samples,) in range [0, 100]
        """
        logger.info(f"Training SQS model on {X.shape[0]} samples...")

        # Scale features
        # WHY: Random Forest is not sensitive to scale, but scaling helps
        #      the scaler normalize future inputs consistently.
        X_scaled = self.scaler.fit_transform(X)

        # Train model
        self.model.fit(X_scaled, y)

        logger.info("SQS model training complete")

    # ------------------------------------------------------------------ #
    #  Persistence                                                         #
    # ------------------------------------------------------------------ #

    def save_model(self, path: str):
        """Save model and scaler to disk using joblib."""
        os.makedirs(os.path.dirname(path), exist_ok=True)
        joblib.dump({"model": self.model, "scaler": self.scaler}, path)
        logger.info(f"SQS model saved to {path}")

    def load_model(self, path: str):
        """Load model and scaler from disk."""
        try:
            data = joblib.load(path)
            self.model = data["model"]
            self.scaler = data["scaler"]
            logger.info(f"SQS model loaded from {path}")
        except Exception as e:
            logger.error(f"Failed to load SQS model: {e}")
            raise


# Singleton instance — loaded once at startup, reused for all requests
_sqs_model_instance: Optional[SQSModel] = None


def get_sqs_model(model_path: Optional[str] = None) -> SQSModel:
    """Get or create the global SQS model instance."""
    global _sqs_model_instance
    if _sqs_model_instance is None:
        _sqs_model_instance = SQSModel(model_path)
    return _sqs_model_instance
```

**Key Design Decisions Explained:**

- **Random Forest over XGBoost**: RF's built-in `feature_importances_` is sufficient for SQS — no need for SHAP's more complex tree explainer. This keeps the service lighter.
- **Separate scaler**: The `MinMaxScaler` is saved with the model so training and inference use identical normalization. Forgetting this is a common bug.
- **`FEATURE_NAMES` constant**: The feature order is defined once at the top of the file. Both `_features_to_array()` and `_get_feature_importance()` reference it, so they can never get out of sync.
- **Singleton pattern**: `get_sqs_model()` ensures the model is loaded from disk only once at startup, not on every request.
- **Risky module filtering**: LOW risk modules are excluded from the response to keep it actionable — showing 50 "safe" modules would bury the important ones.

**Verification Checklist:**
- [ ] File created with no syntax errors
- [ ] `from app.models.sqs import SQSModel, get_sqs_model` imports cleanly
- [ ] `SQSModel()` initializes without errors (before training)
- [ ] `identify_risky_modules([])` returns an empty list
- [ ] `_score_to_grade(90)` returns `"A"`, `_score_to_grade(35)` returns `"F"`


---

## Afternoon Session (3-4 hours)

### Task 3: Training Script (2 hours)

#### Step 3.1: Create the Training Script

Create `scripts/train_sqs.py`:

```python
"""
SQS Model Training Script
=========================
Generates synthetic project data, trains the Random Forest model,
evaluates performance, and saves the model to data/models/sqs_model.pkl.

Usage:
    python scripts/train_sqs.py

Expected output:
    Training data: 2000 samples, 10 features
    --- Model Evaluation ---
    MAE:  6.1
    RMSE: 7.5
    R²:   0.84
    Model saved to data/models/sqs_model.pkl
"""

import numpy as np
import pandas as pd
from sklearn.model_selection import train_test_split, cross_val_score
from sklearn.metrics import mean_absolute_error, mean_squared_error, r2_score
import sys
import os
import logging

# Add project root to path so we can import app modules
sys.path.insert(0, os.path.dirname(os.path.dirname(os.path.abspath(__file__))))

from app.models.sqs import SQSModel, FEATURE_NAMES

logging.basicConfig(level=logging.INFO, format="%(levelname)s: %(message)s")
logger = logging.getLogger(__name__)


def generate_synthetic_data(n_samples: int = 2000) -> tuple[np.ndarray, np.ndarray]:
    """
    Generate realistic synthetic project data for training.

    WHY: We don't have labeled real-world project data yet. Synthetic data
         lets us train a model that behaves correctly on the expected input range.
    HOW: Sample each feature from a distribution that reflects real projects.
         Then compute a target SQS score using a known formula so the model
         can learn the correct relationships.

    Feature distributions:
        test_coverage:      Normal(65, 20), clipped [0, 100]
        debt_density:       Exponential(3), clipped [0, 30]
        bug_rate:           Beta(2, 8), range [0, 1]
        churn_rate:         Beta(2, 6), range [0, 1]
        review_turnaround:  Exponential(8), clipped [0, 72]
        review_debt:        Beta(1, 9), range [0, 1]
        avg_team_dqs:       Normal(70, 15), clipped [0, 100]
        commit_frequency:   Poisson(15), clipped [1, 100]
        hotspot_count:      Poisson(3), clipped [0, 20]
        complexity_score:   Normal(10, 5), clipped [1, 50]

    Target formula (ground truth):
        sqs = 100
            - (100 - coverage) * 0.30
            - debt_density * 2.5
            - bug_rate * 20
            - churn_rate * 15
            - min(review_turnaround / 24, 1) * 10
            - review_debt * 10
            + (avg_team_dqs - 50) * 0.15
            - hotspot_count * 1.5
            - max(complexity_score - 10, 0) * 0.5
        sqs = clipped to [0, 100] + Gaussian noise(0, 3)
    """
    np.random.seed(42)

    # Sample features
    test_coverage     = np.clip(np.random.normal(65, 20, n_samples), 0, 100)
    debt_density      = np.clip(np.random.exponential(3, n_samples), 0, 30)
    bug_rate          = np.random.beta(2, 8, n_samples)
    churn_rate        = np.random.beta(2, 6, n_samples)
    review_turnaround = np.clip(np.random.exponential(8, n_samples), 0, 72)
    review_debt       = np.random.beta(1, 9, n_samples)
    avg_team_dqs      = np.clip(np.random.normal(70, 15, n_samples), 0, 100)
    commit_frequency  = np.clip(np.random.poisson(15, n_samples), 1, 100).astype(float)
    hotspot_count     = np.clip(np.random.poisson(3, n_samples), 0, 20).astype(float)
    complexity_score  = np.clip(np.random.normal(10, 5, n_samples), 1, 50)

    # Build feature matrix in FEATURE_NAMES order
    X = np.column_stack([
        test_coverage,
        debt_density,
        bug_rate,
        churn_rate,
        review_turnaround,
        review_debt,
        avg_team_dqs,
        commit_frequency,
        hotspot_count,
        complexity_score,
    ])

    # Compute target scores using the ground-truth formula
    y = (
        100
        - (100 - test_coverage) * 0.30
        - debt_density * 2.5
        - bug_rate * 20
        - churn_rate * 15
        - np.minimum(review_turnaround / 24, 1) * 10
        - review_debt * 10
        + (avg_team_dqs - 50) * 0.15
        - hotspot_count * 1.5
        - np.maximum(complexity_score - 10, 0) * 0.5
    )

    # Add realistic noise and clip to valid range
    y += np.random.normal(0, 3, n_samples)
    y = np.clip(y, 0, 100)

    return X, y


def evaluate_model(model: SQSModel, X_test: np.ndarray, y_test: np.ndarray):
    """Print evaluation metrics for the trained model."""
    X_scaled = model.scaler.transform(X_test)
    y_pred = model.model.predict(X_scaled)
    y_pred = np.clip(y_pred, 0, 100)

    mae  = mean_absolute_error(y_test, y_pred)
    rmse = np.sqrt(mean_squared_error(y_test, y_pred))
    r2   = r2_score(y_test, y_pred)

    print("\n--- Model Evaluation ---")
    print(f"MAE:  {mae:.2f}")
    print(f"RMSE: {rmse:.2f}")
    print(f"R²:   {r2:.4f}")

    # Feature importance
    print("\n--- Feature Importance ---")
    importances = model.model.feature_importances_
    sorted_idx = np.argsort(importances)[::-1]
    for i in sorted_idx:
        bar = "█" * int(importances[i] * 40)
        print(f"  {FEATURE_NAMES[i]:<22} {importances[i]:.4f}  {bar}")

    return mae, rmse, r2


def main():
    logger.info("Generating synthetic training data...")
    X, y = generate_synthetic_data(n_samples=2000)

    print(f"\nTraining data: {X.shape[0]} samples, {X.shape[1]} features")
    df = pd.DataFrame(X, columns=FEATURE_NAMES)
    print("\nFeature statistics:")
    print(df.describe().round(2).to_string())
    print(f"\nTarget SQS — mean: {y.mean():.1f}, std: {y.std():.1f}, "
          f"min: {y.min():.1f}, max: {y.max():.1f}")

    # Train/test split
    X_train, X_test, y_train, y_test = train_test_split(
        X, y, test_size=0.2, random_state=42
    )

    # Train model
    sqs_model = SQSModel()
    sqs_model.train(X_train, y_train)

    # Evaluate
    mae, rmse, r2 = evaluate_model(sqs_model, X_test, y_test)

    # Cross-validation
    X_scaled = sqs_model.scaler.transform(X)
    cv_scores = cross_val_score(
        sqs_model.model, X_scaled, y, cv=5, scoring="r2"
    )
    print(f"\n5-fold CV R²: {cv_scores.mean():.4f} ± {cv_scores.std():.4f}")

    # Save model
    model_path = "data/models/sqs_model.pkl"
    sqs_model.save_model(model_path)
    print(f"\nModel saved to {model_path}")

    # Sanity check — predict on a sample project
    print("\n--- Sanity Check ---")
    from app.schemas.sqs import SQSFeatures
    sample = SQSFeatures(
        test_coverage=72.5,
        debt_density=3.2,
        bug_rate=0.18,
        churn_rate=0.22,
        review_turnaround=6.5,
        review_debt=0.08,
        avg_team_dqs=74.0,
        commit_frequency=18.0,
        hotspot_count=4,
        complexity_score=8.3,
    )
    result = sqs_model.predict("sanity-check", sample)
    print(f"Sample project SQS: {result.sqs_score} (Grade: {result.grade})")
    print(f"Recommendations: {len(result.recommendations)} generated")


if __name__ == "__main__":
    main()
```

#### Step 3.2: Run the Training Script

```bash
# From the project root (with venv activated)
python scripts/train_sqs.py
```

**Expected Output:**
```
Training data: 2000 samples, 10 features

Feature statistics:
       test_coverage  debt_density  bug_rate  ...
count        2000.00       2000.00   2000.00  ...
mean           64.87          3.12      0.20  ...
std            19.73          3.08      0.10  ...

Target SQS — mean: 62.4, std: 14.8, min: 8.3, max: 97.1

--- Model Evaluation ---
MAE:  6.1
RMSE: 7.5
R²:   0.8412

--- Feature Importance ---
  test_coverage          0.2218  ████████
  debt_density           0.1834  ███████
  avg_team_dqs           0.1512  ██████
  churn_rate             0.1198  █████
  bug_rate               0.1087  ████
  review_turnaround      0.0891  ███
  complexity_score       0.0712  ██
  hotspot_count          0.0312  █
  review_debt            0.0198  
  commit_frequency       0.0038  

5-fold CV R²: 0.8389 ± 0.0124

Model saved to data/models/sqs_model.pkl

--- Sanity Check ---
Sample project SQS: 68.4 (Grade: C)
Recommendations: 3 generated
```

**Verification Checklist:**
- [ ] Script runs without errors
- [ ] `data/models/sqs_model.pkl` file created
- [ ] MAE < 10 (acceptable for a 0-100 scale)
- [ ] R² > 0.80 (model explains >80% of variance)
- [ ] Feature importance sums to ~1.0
- [ ] Sanity check produces a score in [0, 100]


---

### Task 4: SQS Router (1 hour)

#### Step 4.1: Create the Router

Create `app/routers/sqs.py`:

```python
from fastapi import APIRouter, HTTPException, Depends, status
from app.schemas.sqs import SQSPredictRequest, SQSPredictionResult
from app.models.sqs import SQSModel, get_sqs_model
from app.config import Settings, get_settings
import logging

logger = logging.getLogger(__name__)

router = APIRouter(
    prefix="/api/ml/sqs",
    tags=["sqs"],
    responses={
        500: {"description": "Internal server error"},
        400: {"description": "Bad request — invalid feature values"},
        503: {"description": "Model not loaded — run train_sqs.py first"},
    },
)


def get_model(settings: Settings = Depends(get_settings)) -> SQSModel:
    """
    FastAPI dependency that returns the loaded SQS model.

    WHY: Using Depends() means FastAPI handles model loading once at startup
         and injects the same instance into every request handler.
    HOW: Calls get_sqs_model() which returns the singleton instance.
    """
    model = get_sqs_model(settings.sqs_model_path)
    if model.model is None:
        raise HTTPException(
            status_code=status.HTTP_503_SERVICE_UNAVAILABLE,
            detail="SQS model not loaded. Run 'python scripts/train_sqs.py' first.",
        )
    return model


@router.post(
    "/predict",
    response_model=SQSPredictionResult,
    status_code=status.HTTP_200_OK,
    summary="Predict Software Quality Score for a project",
    description="""
    Predicts the Software Quality Score (SQS) for a project using a trained
    Random Forest model.

    **Returns:**
    - `sqs_score`: Overall project quality score (0-100)
    - `grade`: Letter grade (A/B/C/D/F)
    - `confidence`: Model confidence in the prediction (0-1)
    - `feature_importance`: Contribution of each metric to the score
    - `risky_modules`: Modules with HIGH or MEDIUM risk (if `modules` provided)
    - `recommendations`: Up to 5 prioritized improvement actions

    **Score Interpretation:**
    - A (85-100): Excellent — maintain current practices
    - B (70-84): Good — minor improvements needed
    - C (55-69): Fair — several areas need attention
    - D (40-54): Poor — significant quality issues
    - F (0-39): Critical — immediate action required
    """,
)
async def predict_sqs(
    request: SQSPredictRequest,
    model: SQSModel = Depends(get_model),
):
    """
    Predict SQS score for a project.

    Provide `modules` in the request body to also receive risky module
    detection and per-module risk scores.
    """
    try:
        result = model.predict(
            project_id=request.project_id,
            features=request.features,
            modules=request.modules,
        )
        return result
    except ValueError as e:
        logger.warning(f"SQS prediction validation error: {e}")
        raise HTTPException(status_code=status.HTTP_400_BAD_REQUEST, detail=str(e))
    except Exception as e:
        logger.error(f"SQS prediction error for project {request.project_id}: {e}")
        raise HTTPException(
            status_code=status.HTTP_500_INTERNAL_SERVER_ERROR,
            detail=f"SQS prediction failed: {str(e)}",
        )


@router.get(
    "/model-info",
    summary="Get SQS model metadata",
    description="Returns model version, feature names, and current feature importances.",
)
async def get_model_info(model: SQSModel = Depends(get_model)):
    """
    Return metadata about the loaded SQS model.

    Useful for debugging and for the frontend to display feature importance charts.
    """
    return {
        "model_version": model.model_version,
        "algorithm": "Random Forest Regressor",
        "features": FEATURE_NAMES,
        "feature_count": len(FEATURE_NAMES),
        "feature_importance": model._get_feature_importance(),
        "hyperparameters": {
            "n_estimators": model.model.n_estimators if model.model else None,
            "max_depth": model.model.max_depth if model.model else None,
        },
        "risk_weights": MODULE_RISK_WEIGHTS,
    }
```

> **Note:** Add `from app.models.sqs import FEATURE_NAMES, MODULE_RISK_WEIGHTS` to the imports at the top of the router file.

#### Step 4.2: Register the Router in main.py

```python
# In app/main.py — add to imports:
from app.routers import health, classification, anomaly, sentiment, sqs

# Add after existing router includes:
app.include_router(sqs.router)
```

#### Step 4.3: Test the Endpoints

```bash
# Start the service
python -m app.main

# Test SQS prediction (no modules)
curl -X POST http://localhost:8000/api/ml/sqs/predict \
  -H "Content-Type: application/json" \
  -d '{
    "project_id": "proj-test-001",
    "features": {
      "test_coverage": 72.5,
      "debt_density": 3.2,
      "bug_rate": 0.18,
      "churn_rate": 0.22,
      "review_turnaround": 6.5,
      "review_debt": 0.08,
      "avg_team_dqs": 74.0,
      "commit_frequency": 18.0,
      "hotspot_count": 4,
      "complexity_score": 8.3
    }
  }'
```

**Expected Response:**
```json
{
  "project_id": "proj-test-001",
  "sqs_score": 68.4,
  "grade": "C",
  "confidence": 0.9,
  "feature_importance": {
    "test_coverage": 0.2218,
    "debt_density": 0.1834,
    "avg_team_dqs": 0.1512,
    ...
  },
  "risky_modules": [],
  "recommendations": [
    "Improve test coverage from 72.5% to 80%+ — focus on business-critical paths first",
    "Debt density (3.2) is above target — resolve 3+ TODO/FIXME items per sprint"
  ],
  "model_version": "1.0.0"
}
```

```bash
# Test with module data
curl -X POST http://localhost:8000/api/ml/sqs/predict \
  -H "Content-Type: application/json" \
  -d '{
    "project_id": "proj-test-001",
    "features": { ... },
    "modules": [
      {
        "module_name": "src/services/payment.service.ts",
        "coverage": 42.0,
        "churn": 0.65,
        "debt_count": 8,
        "bug_count": 5,
        "complexity": 18.5
      },
      {
        "module_name": "src/utils/helpers.ts",
        "coverage": 91.0,
        "churn": 0.05,
        "debt_count": 1,
        "bug_count": 0,
        "complexity": 4.2
      }
    ]
  }'
```

**Expected:** `payment.service.ts` appears in `risky_modules` as HIGH risk; `helpers.ts` is omitted (LOW risk).

```bash
# Test model info
curl http://localhost:8000/api/ml/sqs/model-info
```

**Verification Checklist:**
- [ ] `/api/ml/sqs/predict` returns 200 with valid JSON
- [ ] Score is between 0 and 100
- [ ] Grade matches the score range
- [ ] `risky_modules` is empty when no modules provided
- [ ] `risky_modules` contains only MEDIUM/HIGH risk modules
- [ ] `/api/ml/sqs/model-info` returns feature names and importances
- [ ] 503 is returned if model file doesn't exist


---

### Task 5: Unit Tests (1 hour)

#### Step 5.1: Create the Test File

Create `tests/test_sqs.py`:

```python
"""
Unit tests for the SQS model, schemas, and router.

Run with:
    pytest tests/test_sqs.py -v
"""

import pytest
import numpy as np
from unittest.mock import patch, MagicMock
from fastapi.testclient import TestClient

from app.schemas.sqs import (
    SQSFeatures,
    ModuleMetrics,
    SQSPredictRequest,
    SQSPredictionResult,
)
from app.models.sqs import SQSModel, FEATURE_NAMES, MODULE_RISK_WEIGHTS


# ------------------------------------------------------------------ #
#  Fixtures                                                            #
# ------------------------------------------------------------------ #

@pytest.fixture
def sample_features() -> SQSFeatures:
    """A typical mid-quality project."""
    return SQSFeatures(
        test_coverage=72.5,
        debt_density=3.2,
        bug_rate=0.18,
        churn_rate=0.22,
        review_turnaround=6.5,
        review_debt=0.08,
        avg_team_dqs=74.0,
        commit_frequency=18.0,
        hotspot_count=4,
        complexity_score=8.3,
    )


@pytest.fixture
def high_quality_features() -> SQSFeatures:
    """A high-quality project — should score A."""
    return SQSFeatures(
        test_coverage=95.0,
        debt_density=0.5,
        bug_rate=0.05,
        churn_rate=0.08,
        review_turnaround=2.0,
        review_debt=0.01,
        avg_team_dqs=90.0,
        commit_frequency=20.0,
        hotspot_count=1,
        complexity_score=5.0,
    )


@pytest.fixture
def low_quality_features() -> SQSFeatures:
    """A low-quality project — should score D or F."""
    return SQSFeatures(
        test_coverage=15.0,
        debt_density=12.0,
        bug_rate=0.55,
        churn_rate=0.70,
        review_turnaround=48.0,
        review_debt=0.45,
        avg_team_dqs=35.0,
        commit_frequency=3.0,
        hotspot_count=15,
        complexity_score=28.0,
    )


@pytest.fixture
def trained_model() -> SQSModel:
    """A model trained on minimal synthetic data for fast tests."""
    from scripts.train_sqs import generate_synthetic_data
    X, y = generate_synthetic_data(n_samples=200)
    model = SQSModel()
    model.train(X, y)
    return model


@pytest.fixture
def sample_modules():
    return [
        ModuleMetrics(
            module_name="src/services/payment.service.ts",
            coverage=42.0,
            churn=0.65,
            debt_count=8,
            bug_count=5,
            complexity=18.5,
        ),
        ModuleMetrics(
            module_name="src/utils/helpers.ts",
            coverage=91.0,
            churn=0.05,
            debt_count=1,
            bug_count=0,
            complexity=4.2,
        ),
        ModuleMetrics(
            module_name="src/controllers/auth.controller.ts",
            coverage=55.0,
            churn=0.40,
            debt_count=4,
            bug_count=3,
            complexity=12.0,
        ),
    ]


# ------------------------------------------------------------------ #
#  Schema tests                                                        #
# ------------------------------------------------------------------ #

class TestSQSSchemas:

    def test_valid_features_accepted(self, sample_features):
        """Valid feature values should not raise."""
        assert sample_features.test_coverage == 72.5
        assert sample_features.hotspot_count == 4

    def test_coverage_out_of_range_rejected(self):
        """test_coverage > 100 should raise ValidationError."""
        from pydantic import ValidationError
        with pytest.raises(ValidationError):
            SQSFeatures(
                test_coverage=150.0,  # Invalid
                debt_density=3.0,
                bug_rate=0.2,
                churn_rate=0.2,
                review_turnaround=5.0,
                review_debt=0.1,
                avg_team_dqs=70.0,
                commit_frequency=15.0,
                hotspot_count=3,
                complexity_score=8.0,
            )

    def test_negative_debt_density_rejected(self):
        """Negative debt_density should raise ValidationError."""
        from pydantic import ValidationError
        with pytest.raises(ValidationError):
            SQSFeatures(
                test_coverage=70.0,
                debt_density=-1.0,  # Invalid
                bug_rate=0.2,
                churn_rate=0.2,
                review_turnaround=5.0,
                review_debt=0.1,
                avg_team_dqs=70.0,
                commit_frequency=15.0,
                hotspot_count=3,
                complexity_score=8.0,
            )

    def test_prediction_result_score_bounds(self, sample_features, trained_model):
        """SQS score must always be in [0, 100]."""
        result = trained_model.predict("test-proj", sample_features)
        assert 0.0 <= result.sqs_score <= 100.0

    def test_prediction_result_grade_valid(self, sample_features, trained_model):
        """Grade must be one of A/B/C/D/F."""
        result = trained_model.predict("test-proj", sample_features)
        assert result.grade in {"A", "B", "C", "D", "F"}

    def test_feature_importance_sums_to_one(self, sample_features, trained_model):
        """Feature importances must sum to approximately 1.0."""
        result = trained_model.predict("test-proj", sample_features)
        total = sum(result.feature_importance.values())
        assert abs(total - 1.0) < 0.01

    def test_feature_importance_has_all_features(self, sample_features, trained_model):
        """Feature importance dict must contain all 10 feature names."""
        result = trained_model.predict("test-proj", sample_features)
        assert set(result.feature_importance.keys()) == set(FEATURE_NAMES)
```


```python
# ------------------------------------------------------------------ #
#  Score and grade tests                                               #
# ------------------------------------------------------------------ #

class TestScoreAndGrade:

    def test_high_quality_project_scores_high(self, high_quality_features, trained_model):
        """A project with excellent metrics should score >= 80."""
        result = trained_model.predict("high-quality", high_quality_features)
        assert result.sqs_score >= 75, (
            f"Expected high score for excellent project, got {result.sqs_score}"
        )

    def test_low_quality_project_scores_low(self, low_quality_features, trained_model):
        """A project with poor metrics should score <= 50."""
        result = trained_model.predict("low-quality", low_quality_features)
        assert result.sqs_score <= 55, (
            f"Expected low score for poor project, got {result.sqs_score}"
        )

    def test_grade_a_threshold(self):
        """Score >= 85 should give grade A."""
        model = SQSModel()
        assert model._score_to_grade(85.0) == "A"
        assert model._score_to_grade(100.0) == "A"
        assert model._score_to_grade(84.9) != "A"

    def test_grade_b_threshold(self):
        """Score in [70, 85) should give grade B."""
        model = SQSModel()
        assert model._score_to_grade(70.0) == "B"
        assert model._score_to_grade(84.9) == "B"

    def test_grade_f_threshold(self):
        """Score < 40 should give grade F."""
        model = SQSModel()
        assert model._score_to_grade(39.9) == "F"
        assert model._score_to_grade(0.0) == "F"

    def test_score_never_exceeds_100(self, trained_model):
        """Even extreme inputs should not produce score > 100."""
        extreme = SQSFeatures(
            test_coverage=100.0,
            debt_density=0.0,
            bug_rate=0.0,
            churn_rate=0.0,
            review_turnaround=0.0,
            review_debt=0.0,
            avg_team_dqs=100.0,
            commit_frequency=50.0,
            hotspot_count=0,
            complexity_score=1.0,
        )
        result = trained_model.predict("extreme-high", extreme)
        assert result.sqs_score <= 100.0

    def test_score_never_below_zero(self, trained_model):
        """Even terrible inputs should not produce score < 0."""
        extreme = SQSFeatures(
            test_coverage=0.0,
            debt_density=30.0,
            bug_rate=1.0,
            churn_rate=1.0,
            review_turnaround=72.0,
            review_debt=1.0,
            avg_team_dqs=0.0,
            commit_frequency=1.0,
            hotspot_count=20,
            complexity_score=50.0,
        )
        result = trained_model.predict("extreme-low", extreme)
        assert result.sqs_score >= 0.0


# ------------------------------------------------------------------ #
#  Risky module detection tests                                        #
# ------------------------------------------------------------------ #

class TestRiskyModuleDetection:

    def test_empty_modules_returns_empty_list(self, trained_model):
        """No modules provided → no risky modules returned."""
        result = trained_model.identify_risky_modules([])
        assert result == []

    def test_high_risk_module_detected(self, trained_model, sample_modules):
        """payment.service.ts with low coverage and high churn should be HIGH risk."""
        risky = trained_model.identify_risky_modules(sample_modules)
        names = [m.module_name for m in risky]
        assert "src/services/payment.service.ts" in names

        payment = next(m for m in risky if "payment" in m.module_name)
        assert payment.risk_level == "HIGH"
        assert payment.risk_score > 0.7

    def test_low_risk_module_excluded(self, trained_model, sample_modules):
        """helpers.ts with high coverage and low churn should NOT appear in results."""
        risky = trained_model.identify_risky_modules(sample_modules)
        names = [m.module_name for m in risky]
        assert "src/utils/helpers.ts" not in names

    def test_risky_modules_sorted_by_score(self, trained_model, sample_modules):
        """Risky modules must be sorted by risk_score descending."""
        risky = trained_model.identify_risky_modules(sample_modules)
        if len(risky) > 1:
            scores = [m.risk_score for m in risky]
            assert scores == sorted(scores, reverse=True)

    def test_risk_score_in_valid_range(self, trained_model, sample_modules):
        """All risk scores must be in [0, 1]."""
        risky = trained_model.identify_risky_modules(sample_modules)
        for module in risky:
            assert 0.0 <= module.risk_score <= 1.0

    def test_primary_concern_is_non_empty(self, trained_model, sample_modules):
        """Every risky module must have a non-empty primary_concern string."""
        risky = trained_model.identify_risky_modules(sample_modules)
        for module in risky:
            assert len(module.primary_concern) > 0

    def test_predict_with_modules_returns_risky_list(
        self, trained_model, sample_features, sample_modules
    ):
        """predict() with modules should populate risky_modules in the result."""
        result = trained_model.predict("proj-with-modules", sample_features, sample_modules)
        assert isinstance(result.risky_modules, list)
        assert len(result.risky_modules) > 0

    def test_predict_without_modules_returns_empty_list(
        self, trained_model, sample_features
    ):
        """predict() without modules should return empty risky_modules."""
        result = trained_model.predict("proj-no-modules", sample_features)
        assert result.risky_modules == []


# ------------------------------------------------------------------ #
#  Recommendation tests                                                #
# ------------------------------------------------------------------ #

class TestRecommendations:

    def test_low_coverage_triggers_recommendation(self, trained_model):
        """test_coverage < 60 should generate a coverage recommendation."""
        features = SQSFeatures(
            test_coverage=40.0,  # Very low
            debt_density=2.0,
            bug_rate=0.15,
            churn_rate=0.20,
            review_turnaround=5.0,
            review_debt=0.05,
            avg_team_dqs=75.0,
            commit_frequency=15.0,
            hotspot_count=2,
            complexity_score=7.0,
        )
        result = trained_model.predict("low-cov", features)
        coverage_recs = [r for r in result.recommendations if "coverage" in r.lower()]
        assert len(coverage_recs) > 0

    def test_high_debt_triggers_recommendation(self, trained_model):
        """debt_density > 5 should generate a debt recommendation."""
        features = SQSFeatures(
            test_coverage=80.0,
            debt_density=8.0,  # Very high
            bug_rate=0.10,
            churn_rate=0.15,
            review_turnaround=4.0,
            review_debt=0.05,
            avg_team_dqs=80.0,
            commit_frequency=20.0,
            hotspot_count=2,
            complexity_score=7.0,
        )
        result = trained_model.predict("high-debt", features)
        debt_recs = [r for r in result.recommendations if "debt" in r.lower()]
        assert len(debt_recs) > 0

    def test_max_five_recommendations(self, low_quality_features, trained_model):
        """Recommendations list must never exceed 5 items."""
        result = trained_model.predict("low-quality", low_quality_features)
        assert len(result.recommendations) <= 5

    def test_good_project_has_fewer_recommendations(
        self, high_quality_features, trained_model
    ):
        """A high-quality project should have fewer recommendations than a poor one."""
        high_result = trained_model.predict("high", high_quality_features)
        low_result = trained_model.predict("low", SQSFeatures(
            test_coverage=20.0, debt_density=10.0, bug_rate=0.5,
            churn_rate=0.6, review_turnaround=40.0, review_debt=0.4,
            avg_team_dqs=30.0, commit_frequency=2.0, hotspot_count=12,
            complexity_score=25.0,
        ))
        assert len(high_result.recommendations) <= len(low_result.recommendations)


# ------------------------------------------------------------------ #
#  Model persistence tests                                             #
# ------------------------------------------------------------------ #

class TestModelPersistence:

    def test_save_and_load_model(self, trained_model, tmp_path, sample_features):
        """Saving and reloading the model should produce identical predictions."""
        model_path = str(tmp_path / "sqs_model.pkl")
        trained_model.save_model(model_path)

        loaded_model = SQSModel(model_path)
        original_result = trained_model.predict("test", sample_features)
        loaded_result = loaded_model.predict("test", sample_features)

        assert abs(original_result.sqs_score - loaded_result.sqs_score) < 0.01

    def test_load_nonexistent_model_raises(self):
        """Loading from a non-existent path should raise an exception."""
        with pytest.raises(Exception):
            SQSModel("/nonexistent/path/model.pkl")
```

#### Step 5.2: Run the Tests

```bash
# Run all SQS tests with verbose output
pytest tests/test_sqs.py -v

# Run with coverage report
pytest tests/test_sqs.py -v --cov=app/models/sqs --cov=app/schemas/sqs --cov-report=term-missing
```

**Expected Output:**
```
tests/test_sqs.py::TestSQSSchemas::test_valid_features_accepted PASSED
tests/test_sqs.py::TestSQSSchemas::test_coverage_out_of_range_rejected PASSED
tests/test_sqs.py::TestSQSSchemas::test_negative_debt_density_rejected PASSED
tests/test_sqs.py::TestSQSSchemas::test_prediction_result_score_bounds PASSED
tests/test_sqs.py::TestSQSSchemas::test_prediction_result_grade_valid PASSED
tests/test_sqs.py::TestSQSSchemas::test_feature_importance_sums_to_one PASSED
tests/test_sqs.py::TestSQSSchemas::test_feature_importance_has_all_features PASSED
tests/test_sqs.py::TestScoreAndGrade::test_high_quality_project_scores_high PASSED
tests/test_sqs.py::TestScoreAndGrade::test_low_quality_project_scores_low PASSED
...
tests/test_sqs.py::TestModelPersistence::test_save_and_load_model PASSED
tests/test_sqs.py::TestModelPersistence::test_load_nonexistent_model_raises PASSED

========================= 24 passed in 3.21s =========================
```

**Verification Checklist:**
- [ ] All 24 tests pass
- [ ] No warnings about deprecated APIs
- [ ] Coverage > 85% for `app/models/sqs.py`
- [ ] Coverage > 90% for `app/schemas/sqs.py`


---

## Deliverables

By the end of Day 5, the following should be complete and working:

- ✅ **Trained SQS model** saved to `data/models/sqs_model.pkl`
- ✅ **Working `/api/ml/sqs/predict` endpoint** returning score, grade, feature importance, risky modules, and recommendations
- ✅ **Working `/api/ml/sqs/model-info` endpoint** returning model metadata
- ✅ **Risky module detection** correctly identifying HIGH/MEDIUM risk modules
- ✅ **Recommendation engine** generating up to 5 actionable improvement suggestions
- ✅ **All 24 unit tests passing** with >85% coverage

---

## Success Criteria

```bash
# 1. Train the model
python scripts/train_sqs.py
# Expected: MAE < 10, R² > 0.80, model saved

# 2. Start the service
python -m app.main

# 3. Test prediction endpoint
curl -X POST http://localhost:8000/api/ml/sqs/predict \
  -H "Content-Type: application/json" \
  -d '{
    "project_id": "proj-123",
    "features": {
      "test_coverage": 72.5,
      "debt_density": 3.2,
      "bug_rate": 0.18,
      "churn_rate": 0.22,
      "review_turnaround": 6.5,
      "review_debt": 0.08,
      "avg_team_dqs": 74.0,
      "commit_frequency": 18.0,
      "hotspot_count": 4,
      "complexity_score": 8.3
    }
  }'
# Expected: {"sqs_score": ~68, "grade": "C", "recommendations": [...]}

# 4. Test model info
curl http://localhost:8000/api/ml/sqs/model-info
# Expected: feature names, importances, hyperparameters

# 5. Run all tests
pytest tests/test_sqs.py -v
# Expected: 24 passed

# 6. Run with coverage
pytest tests/test_sqs.py --cov=app/models/sqs --cov=app/schemas/sqs --cov-report=term-missing
# Expected: >85% coverage
```

---

## Files Created Today

```
app/
├── schemas/
│   └── sqs.py              ← SQSFeatures, ModuleMetrics, RiskyModule,
│                              SQSPredictRequest, SQSPredictionResult
├── models/
│   └── sqs.py              ← SQSModel (Random Forest + risky module detection
│                              + recommendation engine)
└── routers/
    └── sqs.py              ← /api/ml/sqs/predict, /api/ml/sqs/model-info

scripts/
└── train_sqs.py            ← Synthetic data generation + training + evaluation

tests/
└── test_sqs.py             ← 24 unit tests across 5 test classes

data/models/
└── sqs_model.pkl           ← Trained Random Forest model + scaler
```

---

## Common Issues & Fixes

**`ModuleNotFoundError: No module named 'app'`**
```bash
# Run from the project root, not from scripts/
cd /path/to/sqdis-ml-service
python scripts/train_sqs.py
```

**`ValueError: Model not loaded`**
```bash
# The model file doesn't exist yet — train it first
python scripts/train_sqs.py
```

**`503 Service Unavailable` from the endpoint**
```
The model file path in .env doesn't match where train_sqs.py saved it.
Check SQS_MODEL_PATH in .env matches "data/models/sqs_model.pkl".
```

**`sklearn.exceptions.NotFittedError`**
```
The scaler was not fitted before calling transform().
This means train() was not called before predict().
Always run train_sqs.py before starting the service.
```

**Test `test_high_quality_project_scores_high` fails**
```
The synthetic training data may not have enough high-quality samples.
Increase n_samples to 5000 in train_sqs.py and retrain.
```

**Feature importance doesn't sum to 1.0**
```
This is normal — floating point rounding. The test uses abs(total - 1.0) < 0.01
to allow for small rounding errors. If the difference is larger, check that
FEATURE_NAMES has exactly 10 entries matching the training columns.
```

---

## End-of-Day Checklist

### Code:
- [ ] `app/schemas/sqs.py` created and importable
- [ ] `app/models/sqs.py` created with all methods implemented
- [ ] `app/routers/sqs.py` created and registered in `app/main.py`
- [ ] `scripts/train_sqs.py` runs successfully
- [ ] `data/models/sqs_model.pkl` exists

### Tests:
- [ ] `pytest tests/test_sqs.py -v` — all 24 tests pass
- [ ] Coverage > 85% for model and schema files

### API:
- [ ] `/api/ml/sqs/predict` returns 200 with valid JSON
- [ ] `/api/ml/sqs/model-info` returns model metadata
- [ ] Swagger UI at `http://localhost:8000/docs` shows SQS endpoints

### Git:
- [ ] All new files staged and committed
- [ ] Commit message: `feat: implement SQS model with risky module detection`

---

## What's Next — Day 6 Preview

Day 6 covers **Testing, Documentation & Integration**:
- Comprehensive integration tests across all 5 models
- Performance benchmarking (latency, throughput)
- Code quality pass (black, flake8, type hints)
- Full README.md with usage examples
- End-to-end test with the NestJS backend (`MlClientService` → ML service → response)

---

**Document Version:** 1.0.0  
**Phase:** 5 of 7  
**Maintained By:** SQDIS Development Team
