# On-Prem Platform Design

This repository contains the architecture and implementation design for the on-prem infrastructure platform.

## Navigation

Start with [`00-Architecture.md`](00-Architecture.md).

It provides the high-level architecture, architectural layers, selected technologies, and key design decisions.

The following directories expand each architecture layer with implementation details, diagrams, and Architecture Decision Records (ADRs) where required:

- [`01-Hypervisor-Physical-Infrastructure/`](01-Hypervisor-Physical-Infrastructure/) — Physical infrastructure, Proxmox, networking, and storage.
- [`02-Virtual-machines-Infrastructure Services/`](02-Virtual-machines-Infrastructure%20Services/) — Virtual machines and infrastructure services.
- [`03-kubernetes-platform-operations/`](03-kubernetes-platform-operations/) — Kubernetes platform services and operations.
- [`04-application-delivery-runtime/`](04-application-delivery-runtime/) — CI/CD, GitOps, release processes, and application runtime.
- [`05-ai-integration/`](05-ai-integration/) — AI and MCP integrations with internal platform systems.

## Documentation Structure

The intended navigation flow is:

**High-Level Architecture → Architecture Layer → Implementation Details / Diagrams / ADRs**