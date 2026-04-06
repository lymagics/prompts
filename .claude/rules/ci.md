# CI/CD Rules

---

## 1. Coverage

### 1.1 Coverage Reporting
- Publish coverage reports to Codecov using the GitHub codecov-action.

---

## 2. Pipeline Design

### 2.1 Automation
- Automate all test runs with a CI pipeline.

### 2.2 Modular Jobs
- CI configurations must be modular and split into three jobs:
  - fast tests;
  - deep tests;
  - code style checks.

---

## 3. Local Execution

### 3.1 Local Reproducibility
- All jobs must be executable locally using the act CLI tool:
  https://github.com/nektos/act
