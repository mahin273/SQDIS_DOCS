# SQDIS Technical Pre-Defense Q&A

This file focuses on the kind of technical questions teachers usually ask:

- What is this model?
- How does it work in your project?
- Why did you choose this approach?
- How do you calculate DQS/SQS?
- How do you determine code quality?
- What is AST?
- What is SHAP?
- How will you train and validate the model?
- Why should people trust your score?
- Why Redis, BullMQ, WebSocket, PostgreSQL, FastAPI, NestJS?

Use these answers as speaking guides. Do not memorize word-for-word; understand the logic.

---

## 1. Developer Quality Score (DQS)

### Q1. What is DQS in your project?

**Answer:**  
DQS means **Developer Quality Score**. In our project, it is a 0-100 score that represents the quality and reliability of an individual developer based on development signals. We do not calculate it from only commit count. We combine multiple features: commit activity, test coverage, review participation, bug-fix ratio, code churn, and review turnaround time. The idea is to measure both productivity and quality.

### Q2. How do you calculate DQS?

**Answer:**  
In the current implemented system, DQS is calculated using a heuristic fallback formula. We start from a base score, then add positive contributions such as commit activity, coverage, and review participation. We subtract negative factors such as high bug-fix ratio, high churn, and slow review turnaround. Finally, the score is clipped between 0 and 100.

Conceptually:

```text
DQS = base_score
      + commit_score
      + coverage_score
      + review_score
      - bug_deduction
      - churn_deduction
      - turnaround_deduction
```

### Q3. Why did you choose these DQS features?

**Answer:**  
We selected features that reflect both developer activity and engineering quality. Commit count alone only measures volume. Coverage shows testing discipline. Review count shows collaboration. Bug-fix ratio indicates defect-related work. Churn shows instability or repeated rewriting. Review turnaround shows responsiveness in team workflow. Together, these features create a more balanced view.

### Q4. Why not use only commit count?

**Answer:**  
Commit count is easy to game and can be misleading. A developer can create many small commits without improving quality. Another developer may make fewer commits but solve harder problems. That is why we combine commit count with coverage, review, bug, churn, and responsiveness signals.

### Q5. What does `bug_fix_ratio` mean?

**Answer:**  
`bug_fix_ratio` represents the proportion of commits related to bug fixing or defect correction. In our system, it can help identify whether a developer’s work is frequently connected to bug-related changes. But it must be interpreted carefully, because fixing bugs is not bad by itself. In future versions, we should distinguish between bugs introduced by the developer and bugs fixed by the developer.

### Q6. What does `code_churn` mean?

**Answer:**  
Code churn means how much code is repeatedly changed, rewritten, or removed after being added. High churn may indicate unstable implementation, unclear requirements, or risky changes. In DQS, high churn can reduce the score because it may represent lower stability.

### Q7. How do you handle a developer who works on complex tasks but has fewer commits?

**Answer:**  
This is a limitation of purely activity-based metrics. Our score reduces this problem by not using only commit count. But future improvement should include task complexity, issue priority, story points, seniority level, and peer review feedback. We should treat DQS as a decision-support signal, not a final judgment.

### Q8. How do you prevent DQS from being unfair?

**Answer:**  
We use multiple features, explainability, and human feedback. A manager should see why a score changed, not just the number. Also, the future version should support role-based weighting. For example, a backend developer, frontend developer, QA engineer, and DevOps engineer should not be judged by exactly the same weights.

### Q9. What ML model did you plan for DQS?

**Answer:**  
We planned **XGBoost Regressor** for DQS. DQS is a regression problem because the output is a continuous score from 0 to 100. XGBoost works well with tabular structured data, handles non-linear relationships, and supports SHAP explainability.

### Q10. Why XGBoost for DQS?

**Answer:**  
Because our DQS features are tabular numerical features, such as commit count, coverage, churn, and review time. XGBoost is strong for tabular data, works well with medium-sized datasets, handles feature interactions, and is easier to explain using SHAP compared to deep learning.

### Q11. Why not use a neural network for DQS?

**Answer:**  
Neural networks usually need larger datasets and are harder to explain. Our features are structured metrics, not images or raw sequential text. For tabular data, tree-based models like XGBoost are often more practical, more explainable, and easier to validate.

### Q12. Did you train the XGBoost model?

**Answer:**  
No, real-data training was not performed in FYDP-2. We implemented the model pipeline and service endpoints, but the deployed system uses heuristic fallback scoring. We clearly state that trained accuracy is not claimed.

---

## 2. Software Quality Score (SQS)

### Q13. What is SQS in your project?

**Answer:**  
SQS means **Software Quality Score**. It is a project-level score from 0 to 100 that represents the overall quality of a software project. It uses signals such as average DQS, test coverage, churn rate, technical debt count, and bug density.

### Q14. How is SQS different from DQS?

**Answer:**  
DQS is developer-level. It evaluates an individual developer’s contribution and quality signals. SQS is project-level. It evaluates the health of the software project itself. A project may have good developers but still have high technical debt, low coverage, or risky modules, so we need both scores.

### Q15. How do you calculate SQS?

**Answer:**  
Currently SQS is calculated using a heuristic formula. Positive factors include average DQS and test coverage. Negative factors include churn rate, technical debt count, and bug density. The final score is clipped between 0 and 100.

Conceptually:

```text
SQS = base_score
      + avg_dqs_score
      + coverage_score
      - churn_penalty
      - debt_penalty
      - bug_density_penalty
```

### Q16. Why does SQS use average DQS?

**Answer:**  
Average DQS gives a team-level developer quality signal. If the developers working on a project have strong quality indicators, that can positively influence the project quality. But SQS also includes project-specific metrics, so it does not depend only on people scores.

### Q17. What is bug density?

**Answer:**  
Bug density is a measure of defect-related activity compared with total development activity. In our project, it can be estimated from bug-fix commits or issue-related signals. Higher bug density can indicate lower project quality or unstable modules.

### Q18. What is technical debt count?

**Answer:**  
Technical debt count represents known maintainability problems such as TODO, FIXME, code smells, duplicated logic, high complexity, or weak design areas. In SQDIS, debt signals can come from static analysis and tracked debt items.

### Q19. How do you determine risky modules or folders?

**Answer:**  
We calculate folder-level risk using signals like churn, coverage, bug count, and debt count. If a folder has high churn, low coverage, many bugs, or many debt items, it receives a higher risk label. The risk level can be LOW, MEDIUM, HIGH, or CRITICAL.

### Q20. Why do you need folder-level risk if you already have SQS?

**Answer:**  
A single project score tells us the overall condition, but not where the problem is. Folder-level risk helps developers and managers locate risky parts of the codebase so they can prioritize testing, refactoring, or review.

### Q21. What ML model did you plan for SQS?

**Answer:**  
We planned **Random Forest Regressor** for SQS. Like DQS, SQS is also a regression problem because the output is a numeric score from 0 to 100.

### Q22. Why Random Forest for SQS?

**Answer:**  
Random Forest is robust, works well with tabular project-level metrics, handles non-linear relationships, and is less sensitive to noise than a single decision tree. It is also easier to explain than deep learning.

### Q23. Why not use the same XGBoost model for SQS?

**Answer:**  
We could use XGBoost for SQS too. We selected Random Forest to keep the project-level model simpler and robust as a baseline. In future evaluation, we can compare Random Forest, XGBoost, Linear Regression, and other models using real data.

### Q24. How will you validate SQS?

**Answer:**  
We would compare SQS predictions with real project quality outcomes such as post-release defects, bug issue count, code review rejection rate, coverage trend, maintainability rating, and expert assessment from team leads. For trained regression, we can use MAE, RMSE, and R².

---

## 3. AST and Code Quality

### Q25. What is AST?

**Answer:**  
AST means **Abstract Syntax Tree**. It is a tree representation of source code structure. Instead of treating code as plain text, AST breaks code into meaningful parts such as functions, classes, variables, loops, conditions, imports, and function calls.

### Q26. Why do you use AST in your project?

**Answer:**  
We use AST because code quality cannot be reliably measured only by text matching. AST allows us to understand code structure. For example, we can count branches for complexity, inspect function calls for risky patterns, detect imports for dependencies, and analyze control flow more accurately.

### Q27. How do you determine code quality?

**Answer:**  
We determine code quality using multiple static analysis signals:

- Cyclomatic complexity
- Cognitive complexity
- Maintainability index
- Coupling between objects
- Cohesion
- Circular dependencies
- Semantic clones
- Security patterns
- Taint flow
- Bus factor or knowledge silo risk

These signals are combined to identify maintainability and security risks.

### Q28. What is cyclomatic complexity?

**Answer:**  
Cyclomatic complexity measures the number of independent paths through a function. More conditions, loops, and branches increase complexity. High cyclomatic complexity means the function is harder to test and maintain.

### Q29. How do you calculate cyclomatic complexity?

**Answer:**  
The basic idea is:

```text
Cyclomatic Complexity = 1 + number_of_decision_points
```

Decision points include `if`, `else if`, `for`, `while`, `case`, logical branches, and exception handling paths.

### Q30. What is cognitive complexity?

**Answer:**  
Cognitive complexity measures how difficult code is for humans to understand. It increases with nesting, recursion, complex conditions, and control flow. Two functions may have similar cyclomatic complexity, but the one with deeper nesting can be harder to read.

### Q31. What is maintainability index?

**Answer:**  
Maintainability index is a metric that estimates how maintainable code is based on complexity, lines of code, and other factors. A lower maintainability index means the code may be harder to maintain.

### Q32. What is coupling?

**Answer:**  
Coupling measures how dependent one module or class is on others. High coupling means changes in one area can easily break other areas. In code quality, lower coupling is generally better.

### Q33. What is cohesion?

**Answer:**  
Cohesion measures how closely related the responsibilities inside a module or class are. High cohesion means the module has a clear purpose. Low cohesion means the module may be doing too many unrelated things.

### Q34. What is LCOM?

**Answer:**  
LCOM means **Lack of Cohesion of Methods**. It measures whether methods inside a class work on related data. High LCOM means methods are not strongly related, which may indicate poor class design.

### Q35. What is CBO?

**Answer:**  
CBO means **Coupling Between Objects**. It measures how many other classes or modules a class depends on. High CBO can indicate fragile design because changes in dependencies may affect the class.

### Q36. What is taint analysis?

**Answer:**  
Taint analysis tracks unsafe input from a source to a dangerous sink. For example, user input is a source, and SQL query execution is a sink. If unsanitized input reaches a query, it may indicate SQL injection risk.

### Q37. How do you detect SQL injection risk?

**Answer:**  
We check for patterns where user-controlled input is used in SQL query construction without parameterization. For example, string concatenation or formatted strings inside query execution can be risky.

### Q38. What is semantic clone detection?

**Answer:**  
Semantic clone detection identifies code blocks that are logically similar, even if they are not exactly identical text. In our project, we planned TF-IDF and cosine similarity style comparison for detecting similar code fragments.

### Q39. Why should people trust your code quality result?

**Answer:**  
They should not blindly trust a single number. The trust comes from transparent metrics. SQDIS shows which signals caused the risk: high complexity, low coverage, high churn, security pattern, or debt. Since the score is explainable and based on known software engineering metrics, users can verify the reason.

### Q40. Is your code quality scanner better than SonarQube?

**Answer:**  
No. SonarQube is a mature industrial tool. Our value is integration. SQDIS combines code quality signals with developer scoring, project scoring, real-time GitHub events, onboarding, alerts, and dashboards. Our scanner is a focused implementation for our platform.

### Q41. Why not just use SonarQube?

**Answer:**  
SonarQube focuses mainly on static code quality. SQDIS addresses a broader problem: developer intelligence, project quality scoring, real-time updates, onboarding, anomaly detection, and feedback telemetry. We can even integrate SonarQube results in future as one input source.

---

## 4. SHAP and Explainability

### Q42. What is SHAP?

**Answer:**  
SHAP stands for **SHapley Additive exPlanations**. It explains model predictions by assigning each feature an impact value. For example, in DQS, SHAP can show that high coverage increased the score while high churn decreased it.

### Q43. Why do you need SHAP?

**Answer:**  
Because developer scoring can affect people. If we show only a score, users may not trust it. SHAP helps explain why the score changed. It makes the system more transparent and easier to validate.

### Q44. How does SHAP work conceptually?

**Answer:**  
Conceptually, SHAP compares a prediction with and without each feature and estimates how much that feature contributed. It is based on Shapley values from game theory, where each feature is treated like a player contributing to the final prediction.

### Q45. How would SHAP appear in SQDIS?

**Answer:**  
In the dashboard, SHAP-style output can show feature impact:

```text
coverage_avg: +4.2
review_turnaround_avg: -2.1
code_churn: -1.8
commit_count_30d: +3.2
```

This helps a developer or manager understand what improved or reduced the score.

### Q46. Can SHAP work with heuristic scoring?

**Answer:**  
Strict SHAP is for model explainability, but we can show SHAP-style feature impact for heuristic scoring by exposing each formula component. For example, coverage contributed +4.2 and churn deducted -1.8. In future trained XGBoost mode, real SHAP values can be calculated.

### Q47. Why is explainability important for trust?

**Answer:**  
Because users need to know why the system made a decision. A developer may ask, “Why is my score low?” Explainability shows the contributing factors, so the score becomes actionable instead of mysterious.

---

## 5. ML Training and Evaluation

### Q48. How will you train the DQS model?

**Answer:**  
First, we collect historical developer data from GitHub or organization repositories. Then we extract features such as commit count, coverage, review count, bug ratio, churn, and turnaround. We also need labels or target scores. These labels can come from expert evaluation, historical performance outcomes, or calibrated heuristic scores. Then we split data into train, validation, and test sets and train XGBoost Regressor.

### Q49. How will you train the SQS model?

**Answer:**  
For SQS, we collect project-level data such as average DQS, coverage, churn rate, debt count, bug density, release defects, and module risk. The target label can be project quality rating or future defect outcome. Then we train a regression model such as Random Forest and evaluate it on unseen test data.

### Q50. Where will labels come from?

**Answer:**  
Labels are the hardest part. Possible label sources are expert ratings from team leads, historical defect data, post-release bug count, review rejection rate, production incident count, and calibrated quality scores. For a real product, expert validation is important.

### Q51. What if you do not have labels?

**Answer:**  
If labels are unavailable, we can start with heuristic scoring, unsupervised anomaly detection, and weak supervision. Later, human feedback and team-lead overrides can create labeled data for supervised training.

### Q52. What is weak supervision?

**Answer:**  
Weak supervision means generating approximate labels using rules, heuristics, or noisy signals instead of manually labeled data. For example, high bug density and low coverage may create a weak label for low project quality.

### Q53. How will you split the dataset?

**Answer:**  
We can use 70% training, 15% validation, and 15% testing. For time-based software data, a chronological split may be better: train on older data and test on newer data, because that simulates real prediction.

### Q54. What metrics will you use for DQS/SQS regression?

**Answer:**  
We can use:

- MAE: mean absolute error
- RMSE: root mean squared error
- R² score
- Correlation with expert rating
- Stability over time

MAE is especially understandable because it tells average score error in points.

### Q55. What metrics will you use for classification?

**Answer:**  
For commit classification, we can use accuracy, precision, recall, F1-score, and confusion matrix. F1-score is useful if classes are imbalanced.

### Q56. What metrics will you use for anomaly detection?

**Answer:**  
If labeled anomalies exist, we can use precision, recall, F1-score, and ROC-AUC. If labels are not available, we can use expert review and inspect top-ranked anomalies.

### Q57. How will you avoid overfitting?

**Answer:**  
We can use train-validation-test split, cross-validation, regularization, early stopping for XGBoost, limiting tree depth, and testing on unseen repositories or future time periods.

### Q58. What is cross-validation?

**Answer:**  
Cross-validation splits the dataset into several folds. The model trains on some folds and validates on another fold repeatedly. It gives a more reliable estimate than a single split.

### Q59. What is hyperparameter tuning?

**Answer:**  
Hyperparameter tuning means searching for the best model settings, such as number of trees, max depth, learning rate, and regularization. We can use GridSearchCV or RandomizedSearchCV.

### Q60. Why do you not claim accuracy now?

**Answer:**  
Because real-data training and validation were not completed. Claiming accuracy without a trained model and proper test data would be misleading. Our honest claim is platform completion with tested heuristic-mode scoring.

### Q61. If no model is trained, what exactly is implemented in the ML service?

**Answer:**  
The ML service endpoints, request/response schemas, model loading structure, fallback scoring, AST scanner, telemetry logging, health check, and model pipeline code are implemented. It is ready to load trained models later.

### Q62. How can you replace heuristic scoring with trained models later?

**Answer:**  
After training, we serialize trained models as `.pkl` files, place them in the model directory, and update the ML service to load them. The backend API contract does not need to change because endpoints already exist.

---

## 6. Commit Classification and Anomaly Detection

### Q63. What is commit classification?

**Answer:**  
Commit classification means identifying the type of a commit, such as BUGFIX, FEATURE, REFACTOR, TEST, or DOCS. This helps calculate project metrics and understand development patterns.

### Q64. What model did you plan for commit classification?

**Answer:**  
We planned Logistic Regression with TF-IDF features. Commit messages are text, so TF-IDF converts text into numerical vectors, and Logistic Regression classifies the commit type.

### Q65. What is TF-IDF?

**Answer:**  
TF-IDF means Term Frequency-Inverse Document Frequency. It gives higher weight to words that are important in one document but not common in all documents. For commit messages, words like “fix”, “bug”, “test”, or “docs” can help classification.

### Q66. Why Logistic Regression for commit classification?

**Answer:**  
It is simple, fast, interpretable, and works well for text classification with TF-IDF features. It is a good baseline before using more complex NLP models.

### Q67. What fallback do you use for classification?

**Answer:**  
If a trained classifier is not available, we use regex or keyword-based rules. For example, commits containing “fix”, “bug”, or “patch” can be classified as BUGFIX, while messages containing “test” may be classified as TEST.

### Q68. What is anomaly detection in SQDIS?

**Answer:**  
Anomaly detection identifies unusual or risky commits. For example, a commit changing thousands of lines, touching too many files, happening at unusual hours, or having very high churn may be flagged.

### Q69. Why Isolation Forest?

**Answer:**  
Isolation Forest is useful for unsupervised anomaly detection. It identifies points that are easier to isolate from normal data. Since anomalies are rare and labeled anomaly data is hard to collect, Isolation Forest is a practical choice.

### Q70. What anomaly features do you use?

**Answer:**  
Possible features include lines changed, files changed, time of day, churn ratio, deleted-to-added ratio, and number of risky files touched.

---

## 7. Trust, Reliability, and Fairness

### Q71. Why should a teacher or company trust your score?

**Answer:**  
They should trust it as a transparent decision-support signal, not as an absolute truth. SQDIS shows the input metrics and the reasons behind the score. It also separates heuristic mode from trained ML mode, so we are honest about the current limitation.

### Q72. How do you validate that code quality determined by your system is correct?

**Answer:**  
We validate it in three ways. First, compare scanner results with known static analysis rules. Second, test with sample code containing known problems. Third, in future, compare with expert review or established tools like SonarQube.

### Q73. What if your system gives a wrong developer score?

**Answer:**  
That is why the score should not be used alone. The dashboard should show feature explanations and allow human review. Future feedback telemetry can record corrections and improve the model.

### Q74. How do you handle bias?

**Answer:**  
Bias can come from data, features, or team context. To reduce it, we use multiple signals, explainability, human feedback, and future role-aware scoring. We should avoid comparing developers with different roles using the exact same weights.

### Q75. Can developers game the score?

**Answer:**  
Any metric can be gamed if it is too simple. SQDIS reduces gaming by using multiple features. Increasing commit count alone is not enough because churn, bugs, coverage, and reviews also affect the score.

### Q76. Why is human feedback needed?

**Answer:**  
Because human context matters. A team lead may know that a developer worked on a difficult task or inherited bad code. Feedback allows the system to record corrections and use them for future retraining.

---

## 8. Redis, Queue, and Real-Time System

### Q77. Why do you need Redis?

**Answer:**  
Redis is needed for four main reasons: caching, rate limiting, BullMQ queue support, and pub/sub for real-time cache invalidation. It improves performance and helps multiple backend instances coordinate.

### Q78. Why not just use PostgreSQL instead of Redis?

**Answer:**  
PostgreSQL is the durable source of truth, but it is not ideal for high-frequency cache reads, pub/sub, rate limiting counters, or queue operations. Redis is in-memory and much faster for these tasks.

### Q79. What do you cache in Redis?

**Answer:**  
We can cache dashboard metrics, leaderboard data, recent DQS/SQS scores, webhook secrets, repository lookup results, and expensive query results.

### Q80. How do you invalidate cache?

**Answer:**  
When a score or project metric changes, the backend publishes a `cache:invalidate` event through Redis pub/sub. Other backend instances receive it and remove stale cache keys.

### Q81. Why use BullMQ?

**Answer:**  
BullMQ uses Redis and provides reliable background job processing. It supports retries, delays, concurrency control, and job monitoring. This is useful for GitHub webhook processing and heavy analysis tasks.

### Q82. Why not process GitHub webhook synchronously?

**Answer:**  
Synchronous processing can be slow and unreliable because analysis may involve API calls, ML service calls, database writes, and file scanning. A queue lets us return quickly to GitHub and process work safely in the background.

### Q83. What happens if many commits arrive at once?

**Answer:**  
The events are queued. Workers process them based on configured concurrency. This prevents the API from being overloaded and allows retry if some jobs fail.

### Q84. What happens if Redis goes down?

**Answer:**  
Queue processing, caching, pub/sub, and rate limiting will be affected. PostgreSQL still stores durable data, but Redis is important for performance and real-time behavior. In production, Redis should be deployed with persistence and high availability.

### Q85. Why WebSocket?

**Answer:**  
WebSocket allows the server to push updates immediately to dashboards. This is better than polling for real-time score updates, alerts, and commit events.

### Q86. Why Socket.io instead of raw WebSocket?

**Answer:**  
Socket.io provides rooms, reconnection support, event-based messaging, and easier integration with authentication and room-based subscriptions. This fits organization, team, and developer dashboard updates.

### Q87. How do you secure WebSocket?

**Answer:**  
We authenticate the connection using JWT during the handshake. Then the user joins only authorized rooms based on organization, team, or developer permission.

---

## 9. Backend and Architecture

### Q88. Why NestJS?

**Answer:**  
NestJS is modular and works well for large TypeScript backends. It has dependency injection, guards, decorators, DTOs, interceptors, and clear module boundaries. SQDIS has many modules, so NestJS helps maintain structure.

### Q89. Why FastAPI for ML?

**Answer:**  
FastAPI is Python-based and integrates well with scikit-learn, XGBoost, SHAP, MLflow, and AST tools. It is fast, simple, and suitable for exposing ML endpoints.

### Q90. Why separate ML service from backend?

**Answer:**  
Backend product logic and ML logic have different technology needs. NestJS is better for SaaS backend architecture, while Python is better for ML. Separating them allows independent scaling, testing, and deployment.

### Q91. Why PostgreSQL?

**Answer:**  
SQDIS data is relational: users, organizations, projects, teams, commits, scores, alerts, reports, and audit logs. PostgreSQL supports relational integrity, transactions, and complex queries.

### Q92. Why Prisma?

**Answer:**  
Prisma gives type-safe database access, schema migrations, and better developer productivity in a TypeScript backend.

### Q93. Why Docker?

**Answer:**  
Docker makes the system reproducible. Backend, frontend, ML service, PostgreSQL, Redis, and monitoring can run together with the same configuration on another machine.

### Q94. What is multi-tenancy in your project?

**Answer:**  
Multi-tenancy means multiple organizations can use the same system while their data remains isolated. Each user, project, score, and report is scoped to an organization.

### Q95. How do you enforce organization isolation?

**Answer:**  
Through authentication, role guards, organization scope guards, and service-level checks. Queries should include organization ID so users only access data from their own organization.

---

## 10. GitHub Integration and Security

### Q96. How does GitHub integration work?

**Answer:**  
The user connects a GitHub repository. GitHub sends webhook events for pushes, pull requests, reviews, and releases. The backend verifies and queues these events, then extracts commits and file changes for scoring and analysis.

### Q96A. How will the full GitHub pipeline work in SQDIS?

**Answer:**  
The full GitHub pipeline works like this:

```text
GitHub Push / PR / Review
 -> GitHub Webhook
 -> Backend HMAC verification
 -> Rate limit + idempotency check
 -> BullMQ queue
 -> Worker extracts commits and file changes
 -> ML service + AST scanner
 -> PostgreSQL save
 -> Redis cache invalidation
 -> Socket.io broadcast
 -> Dashboard update
```

First, a user connects a repository to SQDIS. Then GitHub sends events such as `push`, `pull_request`, `pull_request_review`, `release`, or `commit_comment` to the backend webhook endpoint. The backend verifies that the event is authentic, queues the event, processes it in the background, calculates scores, stores the result, clears stale cache, and updates the frontend in real time.

### Q96B. What happens when a developer pushes code?

**Answer:**  
When a developer pushes code, GitHub sends a webhook request to SQDIS. The backend verifies the request, extracts repository and commit information, then places the event into BullMQ. A worker later processes the commits, changed files, author identity, lines added/deleted, and timestamps.

### Q96C. Why does the backend verify the GitHub webhook?

**Answer:**  
Webhook verification is needed so fake requests cannot enter the system. SQDIS uses HMAC signature verification to confirm that the webhook was sent by GitHub and signed using the correct repository secret.

### Q96D. What is idempotency in the GitHub pipeline?

**Answer:**  
Idempotency means the same GitHub event should not be processed twice. GitHub provides a delivery ID for each webhook. SQDIS stores or checks that delivery ID so duplicate events do not create duplicate commits, scores, or alerts.

### Q96E. Why does the GitHub event go to BullMQ?

**Answer:**  
GitHub webhook processing can be heavy. It may require commit parsing, file-change extraction, AST scanning, ML service calls, database writes, cache invalidation, and notification events. BullMQ lets the backend respond quickly to GitHub while workers process the heavy tasks asynchronously.

### Q96F. What does the worker extract from GitHub events?

**Answer:**  
The worker extracts useful development signals such as:

- commit SHA
- commit message
- author identity
- changed files
- lines added and deleted
- timestamp
- pull request information
- review information
- repository and project mapping

These signals become features for DQS, SQS, classification, anomaly detection, and code-quality analysis.

### Q96G. How does the GitHub pipeline connect to ML?

**Answer:**  
After extracting commit and file-change data, the backend calls ML service endpoints. The ML service can classify commit type, detect anomaly risk, analyze code quality using AST scanning, calculate DQS, and calculate SQS. The response is returned to the backend and saved in PostgreSQL.

### Q96H. What data is saved after GitHub processing?

**Answer:**  
SQDIS saves commits, file changes, developer identity mapping, DQS/SQS scores, alerts, technical debt items, risky modules, and audit or processing metadata. PostgreSQL is the durable source of truth.

### Q96I. Why is Redis used after GitHub processing?

**Answer:**  
After new scores or commits are saved, old dashboard cache may become stale. Redis is used to invalidate affected cache keys and publish update messages. This ensures dashboards show fresh data without expensive repeated database queries.

### Q96J. How does the dashboard update after GitHub processing?

**Answer:**  
After processing is complete, the backend emits Socket.io events such as `commit:new`, `score:updated`, or `alert:new`. Connected frontend dashboards receive those events and update automatically without manual refresh.

### Q96K. If a teacher asks, “Show me the full process from fetching GitHub data to DQS and SQS generation,” how should we answer?

**Answer:**  
When a repository is connected to SQDIS, GitHub sends events to our backend through webhooks. For example, when a developer pushes code or creates a pull request, GitHub sends a payload containing commit information, author details, changed files, timestamps, and repository metadata.

First, the NestJS backend receives the webhook. It verifies the request using HMAC signature verification, checks rate limiting, and checks idempotency using the GitHub delivery ID so the same event is not processed twice.

After verification, the backend does not process everything directly. It sends the event to a BullMQ queue. This keeps the webhook response fast and allows heavy processing to happen in the background.

Then a worker takes the job from the queue and extracts useful raw data:

- commit SHA
- commit message
- developer identity
- files changed
- lines added and deleted
- timestamp
- pull request and review data
- repository/project mapping

From this raw data, SQDIS generates features.

For **DQS**, we generate developer-level features:

- `commit_count_30d`
- `coverage_avg`
- `review_count`
- `bug_fix_ratio`
- `code_churn`
- `review_turnaround_avg`

For **SQS**, we generate project-level features:

- `avg_dqs`
- `coverage`
- `churn_rate`
- `debt_count`
- `bug_density`

After feature extraction, the backend sends these feature vectors to the FastAPI ML service.

For **DQS**, the ML endpoint receives the developer feature vector and returns a developer quality score from 0 to 100. In planned ML mode, this is handled by an XGBoost Regressor with SHAP explanation. In the current FYDP-2 implementation, because real-data training was not performed, the endpoint uses tested heuristic fallback logic.

For **SQS**, the ML endpoint receives the project-level feature vector and returns a software quality score from 0 to 100, plus risky module information. In planned ML mode, this uses a Random Forest Regressor. In the current implementation, it also uses heuristic fallback logic.

After the ML service returns the scores, the backend saves the results in PostgreSQL. It stores commits, file changes, DQS, SQS, alerts, technical debt, and risky module data.

Then Redis invalidates old cached dashboard data. Finally, Socket.io broadcasts events such as `score:updated`, `commit:new`, or `alert:new` to the frontend dashboard.

Complete flow:

```text
GitHub event
 -> NestJS webhook
 -> HMAC + rate limit + idempotency
 -> BullMQ queue
 -> Worker extracts raw data
 -> Feature engineering
 -> FastAPI ML service
 -> DQS/SQS prediction
 -> PostgreSQL save
 -> Redis cache invalidation
 -> Socket.io real-time update
 -> Dashboard shows new scores
```

Important clarification: in our current FYDP-2 system, the ML model endpoints and pipeline are implemented, but real-data model training was not performed. So DQS and SQS are generated using tested heuristic fallback logic. The trained XGBoost and Random Forest models are future scope once real labeled data is collected.

### Q97. What is HMAC verification?

**Answer:**  
HMAC verification checks whether the webhook request was signed using the correct shared secret. The backend computes the expected signature and compares it with GitHub’s signature.

### Q98. Why is HMAC needed?

**Answer:**  
Without HMAC, anyone could send fake webhook events. HMAC proves the event came from GitHub and was not modified.

### Q99. What is idempotency?

**Answer:**  
Idempotency means processing the same event multiple times should not create duplicate results. GitHub provides a delivery ID, and we can use that ID to avoid duplicate processing.

### Q100. Why rate limiting?

**Answer:**  
Rate limiting protects the backend from floods or abuse. It also prevents too many expensive jobs from being created at once.

---

## 11. Testing and Validation

### Q101. How did you test without trained ML models?

**Answer:**  
We tested endpoint behavior, request/response schemas, fallback scoring, edge cases, AST scanner outputs, telemetry logging, and integration paths. We validate system behavior, not trained accuracy.

### Q102. What does API contract testing mean?

**Answer:**  
It means verifying that each API accepts the expected request format and returns the expected response structure. This is important because the backend and frontend depend on stable ML service responses.

### Q103. How do you test DQS/SQS formulas?

**Answer:**  
We pass known input feature vectors and verify that the output score matches the expected formula result, including clipping between 0 and 100.

### Q104. How do you test AST scanner?

**Answer:**  
We use sample code with known issues, such as high complexity or insecure patterns, then check whether the scanner detects them correctly.

### Q105. How do you test real-time flow?

**Answer:**  
We trigger a scoring or commit event, verify database update, cache invalidation, Redis pub/sub event, and Socket.io broadcast to the dashboard.

### Q106. How do you test security?

**Answer:**  
We test invalid JWT, role restrictions, organization scope checks, invalid webhook signatures, duplicate delivery IDs, and rate-limit behavior.

---

## 12. Limitations and Future Defense Questions

### Q107. What is the biggest technical limitation?

**Answer:**  
The biggest limitation is that real-data model training was not performed, so trained accuracy is not available. The current system is complete in heuristic mode.

### Q108. If training is future work, what is the technical achievement now?

**Answer:**  
The technical achievement is the complete platform and data pipeline: GitHub ingestion, queue processing, backend modules, ML endpoints, AST scanner, scoring fallback, database storage, cache invalidation, WebSocket updates, dashboards, reports, and deployment.

### Q109. What would you improve first?

**Answer:**  
I would collect a real dataset, define reliable labels, train DQS and SQS models, compare model performance against heuristic baseline, and validate results with expert review.

### Q110. How would you prove your system works in a company?

**Answer:**  
Run it on historical project data, compare its risk predictions with actual bugs and incidents, ask team leads to review score explanations, and measure whether alerts help reduce late defect discovery.

### Q111. What if a teacher says your DQS is subjective?

**Answer:**  
The current weights are heuristic, so they are a transparent baseline, not final truth. The long-term plan is to train and validate weights using real outcomes and expert feedback. We clearly separate heuristic scoring from trained ML.

### Q112. What if a teacher asks why your code quality score is reliable?

**Answer:**  
It is reliable as a transparent static-analysis signal because it uses known software engineering metrics like complexity, coupling, cohesion, churn, coverage, and security patterns. It should be validated against expert review and tools like SonarQube in future work.

### Q113. What if they ask why people will accept developer scoring?

**Answer:**  
People will accept it only if it is transparent, explainable, and used responsibly. That is why we show feature impacts and recommend using DQS as supporting evidence, not as the only performance evaluation method.

### Q114. What if they ask whether this can replace managers?

**Answer:**  
No. SQDIS supports managers by giving measurable signals. It does not replace human judgment. Context, task difficulty, and team role still require human review.

### Q115. What if they ask whether this project is ML or software engineering?

**Answer:**  
It is both, but the completed FYDP-2 contribution is mainly an integrated software engineering platform with ML-ready services and heuristic-mode intelligence. The ML training and accuracy validation are future research scope.

---

## 13. Rapid Technical Answers

## 13. Core Model Theory Questions

### Q116. How does XGBoost work?

**Answer:**  
XGBoost means **Extreme Gradient Boosting**. It builds many decision trees sequentially. The first tree makes an initial prediction. The next tree tries to correct the errors of the previous tree. This process continues, and each new tree focuses more on the mistakes made earlier. Finally, the predictions from all trees are combined.

In simple terms:

```text
Final prediction = first tree + correction tree + correction tree + ...
```

It is called gradient boosting because it uses gradient-based optimization to reduce the loss function.

### Q117. How does XGBoost fit your DQS problem?

**Answer:**  
DQS uses tabular numerical features such as commit count, coverage, review count, churn, bug ratio, and review turnaround. XGBoost works very well on tabular data and can learn non-linear relationships. For example, high commit count is good only if churn and bug ratio are not too high. XGBoost can learn this kind of feature interaction.

### Q118. What is boosting?

**Answer:**  
Boosting is an ensemble learning method where models are trained one after another. Each new model focuses on correcting the mistakes of the previous models. The final result is stronger than a single model.

### Q119. What is gradient boosting?

**Answer:**  
Gradient boosting is a boosting method that minimizes a loss function step by step. It calculates the error direction using gradients and trains the next tree to reduce that error.

### Q120. What is the loss function in XGBoost regression?

**Answer:**  
For regression, a common loss function is squared error:

```text
Loss = (actual_value - predicted_value)^2
```

The model tries to minimize prediction error. For DQS, the predicted value would be the developer quality score.

### Q121. What are decision trees?

**Answer:**  
A decision tree is a model that splits data based on feature conditions. For example:

```text
if coverage_avg > 80:
    go left
else:
    go right
```

After several splits, the tree reaches a leaf node that gives a prediction.

### Q122. Why is XGBoost better than a single decision tree?

**Answer:**  
A single decision tree can overfit and may not generalize well. XGBoost combines many small trees, where each tree corrects previous errors. This usually gives better accuracy and stability.

### Q123. What are important XGBoost hyperparameters?

**Answer:**  
Important hyperparameters include:

- `n_estimators`: number of trees
- `max_depth`: maximum depth of each tree
- `learning_rate`: how much each tree contributes
- `subsample`: percentage of data used per tree
- `colsample_bytree`: percentage of features used per tree
- `reg_alpha` and `reg_lambda`: regularization

### Q124. What is learning rate in XGBoost?

**Answer:**  
Learning rate controls how strongly each new tree affects the final prediction. A smaller learning rate makes training slower but can improve generalization. A larger learning rate trains faster but may overfit.

### Q125. What is overfitting in XGBoost?

**Answer:**  
Overfitting means the model learns the training data too specifically and performs poorly on new data. In XGBoost, overfitting can happen if trees are too deep or there are too many trees without proper regularization.

### Q126. How do you reduce overfitting in XGBoost?

**Answer:**  
We can reduce overfitting by limiting tree depth, using regularization, lowering learning rate, using subsampling, using cross-validation, and applying early stopping.

### Q127. What is early stopping?

**Answer:**  
Early stopping stops training when validation performance stops improving. This prevents the model from continuing to learn noise from training data.

### Q128. How does Random Forest work?

**Answer:**  
Random Forest is an ensemble of many decision trees. Each tree is trained on a random sample of the data, and each split considers a random subset of features. For regression, the final output is usually the average prediction of all trees.

```text
Final prediction = average(tree1, tree2, tree3, ...)
```

### Q129. Why is it called Random Forest?

**Answer:**  
It is called a forest because it contains many decision trees. It is random because each tree uses random samples of data and random subsets of features.

### Q130. How does Random Forest fit your SQS problem?

**Answer:**  
SQS uses project-level tabular features such as average DQS, coverage, churn, debt count, and bug density. Random Forest can learn how these features together affect project quality. It is also robust to noisy data and works well as a baseline model.

### Q131. Difference between Random Forest and XGBoost?

**Answer:**  
Random Forest trains many trees independently and averages their results. XGBoost trains trees sequentially, where each new tree corrects previous errors.

Short version:

```text
Random Forest = parallel trees + averaging
XGBoost = sequential trees + error correction
```

### Q132. Which one is more explainable, Random Forest or XGBoost?

**Answer:**  
Both are more explainable than deep learning. Random Forest can show feature importance. XGBoost can also show feature importance and works well with SHAP for detailed feature-level explanations.

### Q133. What is feature importance?

**Answer:**  
Feature importance tells which input features were most useful for prediction. For example, in SQS, coverage or bug density may have high importance if they strongly affect project quality prediction.

### Q134. What is the difference between regression and classification?

**Answer:**  
Regression predicts a continuous numeric value, such as DQS score from 0 to 100. Classification predicts a category, such as BUGFIX, FEATURE, TEST, or DOCS.

### Q135. In your project, which tasks are regression?

**Answer:**  
DQS and SQS are regression tasks because they output numeric scores from 0 to 100.

### Q136. In your project, which tasks are classification?

**Answer:**  
Commit type prediction is classification because it outputs a class such as BUGFIX, FEATURE, REFACTOR, TEST, or DOCS.

### Q137. How does Logistic Regression work?

**Answer:**  
Logistic Regression is a classification model. It calculates a weighted sum of input features and converts it into class probabilities using a sigmoid or softmax function. The class with the highest probability is selected.

### Q138. Why use Logistic Regression for commit classification?

**Answer:**  
Commit messages are short text. After converting them to TF-IDF vectors, Logistic Regression is fast, simple, and effective as a baseline text classifier.

### Q139. What is TF-IDF?

**Answer:**  
TF-IDF means **Term Frequency-Inverse Document Frequency**. It converts text into numbers. Words that appear often in one document but not in every document get higher importance.

For example, words like `fix`, `bug`, `test`, `docs`, or `refactor` can help classify commit messages.

### Q140. How does TF-IDF work?

**Answer:**  
TF-IDF has two parts:

- **TF:** how often a word appears in a document
- **IDF:** how rare the word is across all documents

```text
TF-IDF = Term Frequency × Inverse Document Frequency
```

A word gets high value if it is frequent in one document but rare across the dataset.

### Q141. How does Isolation Forest work?

**Answer:**  
Isolation Forest detects anomalies by randomly splitting data. Normal points usually need many splits to isolate, but unusual points are easier to isolate with fewer splits. If a commit is very different from normal commits, it receives a higher anomaly score.

### Q142. How does Isolation Forest fit your project?

**Answer:**  
In SQDIS, unusual commits may be risky. For example, a commit with 5000 changed lines, 100 files, very high churn, or unusual commit time may be anomalous. Isolation Forest can learn normal commit behavior and flag unusual ones.

### Q143. Why use unsupervised anomaly detection?

**Answer:**  
Because labeled anomaly data is difficult to collect. We may not have many examples labeled as “risky commit” or “normal commit”. Isolation Forest can work without labeled examples.

### Q144. What is model training?

**Answer:**  
Model training is the process where a model learns patterns from historical data. It adjusts internal parameters to reduce prediction error on training examples.

### Q145. What is model inference?

**Answer:**  
Inference means using a trained model to make predictions on new data. In SQDIS, inference would happen when new GitHub activity is processed and the model predicts DQS, SQS, classification, or anomaly score.

### Q146. What is a feature vector?

**Answer:**  
A feature vector is a numerical representation of one sample. For example, one developer’s DQS feature vector may be:

```text
[commit_count_30d, coverage_avg, review_count, bug_fix_ratio, code_churn, review_turnaround_avg]
```

### Q147. What is normalization?

**Answer:**  
Normalization means scaling features so they are comparable. For example, commit count may range from 0 to 100, while bug ratio ranges from 0 to 1. Scaling can help some models learn better.

### Q148. Do tree models like XGBoost and Random Forest need normalization?

**Answer:**  
Usually no. Tree-based models split based on feature thresholds, so they are less sensitive to feature scale. But normalization may still help for other models or for consistent preprocessing.

### Q149. What is model serialization?

**Answer:**  
Model serialization means saving a trained model to a file, such as `.pkl`. Later, the ML service can load that file and use it for prediction.

### Q150. What happens if the trained model file is missing?

**Answer:**  
The system uses fallback logic. For DQS and SQS, it uses heuristic formulas. For classification, it uses regex rules. For anomaly detection, it uses rule-based thresholds.

### Q151. Why is fallback important?

**Answer:**  
Fallback is important because it keeps the platform functional even if trained models are missing, corrupted, or the ML service is not ready. It improves reliability.

### Q152. What is the difference between heuristic and trained model?

**Answer:**  
A heuristic uses manually defined rules and weights. A trained model learns patterns from data. Heuristics are transparent and easy to control, while trained models can adapt better if trained with good data.

### Q153. Why begin with heuristic scoring?

**Answer:**  
Because it provides a transparent baseline and allows the full platform to work before collecting enough real training data. Later, trained models can be compared against this baseline.

### Q154. How would you compare heuristic vs trained model?

**Answer:**  
We can compare both against real labels or expert ratings. Metrics could include MAE, RMSE, correlation with expert judgment, and ability to predict future bugs or risk.

### Q155. What is a confusion matrix?

**Answer:**  
A confusion matrix shows how many classification predictions were correct or incorrect for each class. For commit classification, it would show how many BUGFIX commits were correctly predicted as BUGFIX and how many were confused with FEATURE or REFACTOR.

### Q156. What is precision?

**Answer:**  
Precision measures how many predicted positives were actually correct.

```text
Precision = True Positives / (True Positives + False Positives)
```

### Q157. What is recall?

**Answer:**  
Recall measures how many actual positives were found by the model.

```text
Recall = True Positives / (True Positives + False Negatives)
```

### Q158. What is F1-score?

**Answer:**  
F1-score is the harmonic mean of precision and recall. It is useful when the dataset is imbalanced.

### Q159. What is MAE?

**Answer:**  
MAE means Mean Absolute Error. It measures the average absolute difference between actual and predicted values.

For example, if the actual DQS is 80 and predicted DQS is 75, the error is 5.

### Q160. What is RMSE?

**Answer:**  
RMSE means Root Mean Squared Error. It penalizes large errors more than MAE because errors are squared before averaging.

### Q161. What is R² score?

**Answer:**  
R² measures how much variance in the target value is explained by the model. A higher R² means better fit, but it should be interpreted with other metrics.

### Q162. How would you answer if asked “Explain your ML pipeline”?

**Answer:**  
Our ML pipeline has these steps:

1. Collect GitHub and project data.
2. Extract features for developers and projects.
3. Clean and preprocess the dataset.
4. Split into train, validation, and test sets.
5. Train models such as XGBoost and Random Forest.
6. Evaluate using MAE/RMSE for regression and F1-score for classification.
7. Save trained models as `.pkl`.
8. Load them in the FastAPI ML service for prediction.
9. Use SHAP or feature impact for explainability.

### Q163. How would you answer “What is your model input and output?”

**Answer:**  
For DQS, input is developer feature vector and output is a 0-100 developer quality score. For SQS, input is project feature vector and output is a 0-100 software quality score plus risk indicators. For classification, input is commit message and files, output is commit type. For anomaly detection, input is commit behavior features, output is anomaly risk.

### Q164. How would you answer “Why these models are suitable?”

**Answer:**  
Because our data is mostly structured tabular engineering metrics. XGBoost and Random Forest are strong for tabular data. Logistic Regression with TF-IDF is a good baseline for short commit text. Isolation Forest is suitable because anomaly labels are hard to collect.

---

## 14. Rapid Technical Answers

### What is AST?

AST is a structured tree representation of source code, used to inspect functions, classes, branches, calls, and dependencies.

### What is SHAP?

SHAP explains how much each feature contributes to a model prediction.

### What is DQS?

DQS is a 0-100 developer quality score based on activity, coverage, review, bug, churn, and responsiveness signals.

### What is SQS?

SQS is a 0-100 project quality score based on average DQS, coverage, churn, debt, and bug density.

### Why Redis?

For caching, BullMQ queue support, rate limiting, and pub/sub cache invalidation.

### Why BullMQ?

To process GitHub events asynchronously with retry, concurrency, and fault tolerance.

### Why WebSocket?

To push score and alert updates to dashboards in real time.

### Why XGBoost?

It is strong for tabular data, handles non-linear feature interactions, and supports SHAP explainability.

### Why Random Forest?

It is robust for project-level tabular metrics and works well as a baseline regressor.

### Why FastAPI?

Python ML ecosystem integration with a simple, fast API layer.

### Why NestJS?

Modular TypeScript backend structure with guards, DTOs, dependency injection, and scalable architecture.

### Why no ML accuracy?

Because real-data model training was not completed, so accuracy would be misleading.

### Current strongest claim?

SQDIS is a complete working SaaS prototype in heuristic ML mode with an ML-ready architecture.
