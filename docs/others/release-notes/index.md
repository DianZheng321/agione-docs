---
prev: false
next: true
---

# Update Notes

## Table of Contents

- [2026-05-15](#2026-05-15)

---

## 2026-05-15

### Update Details

#### General

- 【UI】Brand new UI with dark/light mode support (including new homepage, login/register pages, Console page, etc.)
- 【UI】Platform-wide naming standardization - unified terminology for model providers and model sources across all pages for better consistency and understandability
- 【UI】Optimized model publishing overview UI - improved display of key information before and after model publishing

#### Model Services

- 【Feature】Failover support - automatic model switching capability to reduce invocation failures
- 【Performance】Circuit breaker support - improved call monitoring and success rate evaluation for more stable model selection
- 【Performance】Enhanced streaming output - supports unified format while maintaining compatibility with original model API capabilities, reducing answer interruption, format anomalies, and incomplete results
- 【Feature】Extended video model billing support for both token-based billing (per million tokens) and non-token billing (per second) scenarios
- 【UI】Optimized model name display in billing records - more accurate and recognizable model information in bills and transfer records
- 【Performance】Improved billing snapshot records for easier usage and cost verification
- 【Performance】Optimized model connectivity testing interface for more accurate pre-publishing checks
- 【Feature】Added model source type identification to distinguish models from different sources
- 【Performance】Added platform data synchronization (side-channel sync) for improved accuracy of call data, billing data, and model metrics
- 【Performance】Optimized dashboard aggregation for more stable hourly and daily data summaries

#### AI Infra - On Cloud

- 【Feature】Optimized AWS-related model registration, invocation, and deployment workflows
- 【UI】Improved cloud account query experience with better recognition of shortened names like Alibaba Cloud and Huawei Cloud
- 【Feature】Support publishing cloud inference service Endpoints to Model Services platform
- 【Feature】Support publishing inference service Tokens to Model Services platform
- 【Feature】Support syncing meta-model information to Model Services platform
- 【Feature】Support inference testing of published models on the Model Services platform

#### Finance

- 【Feature】Advanced finance function integration - merging some MSP-side billing capabilities to operator side
- 【UI】Balance display optimization - added display in top-right corner of navigation bar

#### Tools

- 【Feature】Model deployment resource estimation - estimate VRAM and GPU count based on model and accelerator card parameters

### Bug Fixes

- 【Fix】Improved handling of abnormal model parameters to reduce invocation failures caused by non-standard parameters
- 【Fix】More detailed error messages for deployment failures to help users and operations staff locate issues
- 【Fix】Fixed model details, specifications query, and dependency compatibility issues for some cloud providers
- 【Fix】Improved historical data refresh capability to reduce timeout or incomplete data risks during large time range queries
- 【Fix】Fixed payment configuration default value issues for improved billing configuration stability
- 【Fix】Corrected cloud region permission query logic for more accurate available region display
- 【Fix】Fixed special character issues in English text for better multilingual display consistency
