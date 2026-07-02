# 🌐 Linux High Availability (HA) PCS Cluster Suite

This project serves as a comprehensive reference guide for engineering, deployment, and operational management of high-availability Linux environments powered by Pacemaker and Corosync (`pcs`).

## 📋 Architectural Overview

The environment relies on a tightly decoupled, multi-layered stack designed to maintain uninterrupted service delivery and guarantee data integrity:

```text
+---------------------------------------------+

|            Application / Service            |
+---------------------------------------------+

| Pacemaker (CRM - Cluster Resource Manager)  |
+---------------------------------------------+

|   Corosync (Messaging & Membership Layer)   |
+---------------------------------------------+

|     Linux OS (RHEL, Rocky, SLES, etc.)      |
+---------------------------------------------+
```

## 🚀 Key Framework Pillars

* **Communication Layer (Corosync):** Manages real-time node membership discovery, heartbeat messaging tracking, and dynamic math-majority quorum logic.
* **Brain Engine (Pacemaker):** Actively monitors health parameters, orchestrates starting/stopping cycles, and initiates failovers for logical cluster resources.
* **Fencing Mechanism (STONITH):** Safeguards physical and shared logical volumes from split-brain write conditions by forced isolate-reboots of erratic nodes.

---

## 🛠️ Essential Command Cheatsheet Quick-Reference

| Objective Area | Production Terminal Control Syntax |
| :--- | :--- |
| **Cluster Verification** | `pcs status` |
| **System Maintenance** | `pcs node standby <node_name>` |
| **Fault Remediation** | `pcs resource cleanup <resource_id>` |
| **Node Resynchronization** | `pcs resource clear <resource_id>` |
| **Operational Rules** | `pcs constraint list` |

---

## 📖 What's Inside the Core Document

Refer to the main guide file in this repository for full deployment execution plans:
1. **Deep Engineering Breakdowns:** Sub-daemon dynamics (`crmd`, `pengine`), CIB XML behaviors, and quorum configuration rules.
2. **Technical Interview Mastery:** 10 production-tested interview responses dealing with resource stickiness, unmanaged states, and custom fencing loops.
3. **Advanced CRUD Profiles:** Syntax rules for building ordered application resource groups, globally cloned assets, and multi-state active/passive sets.
