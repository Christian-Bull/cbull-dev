---
title: "Multi-environment cluster management with Kustomize and ArgoCD"
date: 2022-10-17
---

Managing cluster add-ons and plugins across multiple environments can be rather tedious with traditional infra-automation tools (terraform/ansible). Combining a simple manifest templating tool such as Kustomize with a GitOps workflow can help simplify managing and upgrade cluster resources.

Consider a project where you have 3 distinct clusters:
- Development
  - Lab environment for testing code and infra changes
- Staging
  - Stable build mirroring production
- Production



