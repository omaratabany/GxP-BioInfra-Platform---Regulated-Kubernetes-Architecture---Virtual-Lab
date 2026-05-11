# Platform Primer -- Plain English Guide

> Part of [[README]] | Read this first before any other file in this vault

This file exists so that when someone asks me what I built, or when I come back to this after time away, I can read this first and not feel lost. Everything in this vault is explained here in plain language before I point at the technical files. No assumed knowledge. No jargon without explanation.

---

## What Is This Thing, Actually?

I built a platform that runs biology data analysis software in a regulated, auditable, and reproducible way.

To break that down:

**Biology data analysis software** -- specifically genomics pipelines. A genomics pipeline is a sequence of steps that processes raw DNA sequencing data and produces useful results -- things like "which genes are active in this tissue sample" or "what mutations does this patient's tumour carry." These pipelines are used by pharmaceutical companies to decide which drugs to develop and which patients to target.

**Regulated** -- pharmaceutical companies operate under strict government rules. In Europe the main one is EU GMP Annex 11. In the US it is FDA 21 CFR Part 11. These rules say that any computer system involved in drug development must have a full record of what ran, when, who ran it, and that the result can be trusted. If the system cannot prove those things, the data it produced cannot be used in a drug approval application.

**Auditable** -- every action that happens on this platform gets recorded automatically. Who logged in, which software ran, what files were written, whether anyone tried to do something they were not supposed to. That record is tamper-evident -- the software recording it runs at the kernel level, below where application code lives, so a pipeline cannot hide its own activity.

**Reproducible** -- running the same pipeline on the same data always produces the same result. This sounds obvious but it is not, because software updates silently. I solve this by locking every piece of software to an exact version fingerprint so nothing can change without a deliberate decision being recorded.

So in one sentence: I built a small-scale pharmaceutical-grade computing environment that runs bioinformatics pipelines and can prove to a regulator that everything it did was correct, controlled, and documented.

---

## Why Does This Matter to Me Specifically?

Pharmaceutical and biotech companies in Basel and Zurich need platform engineers who understand both Kubernetes and GxP regulation. Those two skills almost never exist in the same person. Platform teams know Kubernetes. QA teams know the regulations. The person who bridges both is rare and commands a significant premium.

A standard DevOps portfolio competes with hundreds of applicants. This portfolio makes a specific, provable claim: I designed and built a GxP-compliant bioinformatics platform and wrote the validation documentation to the standard those companies use internally. That is the opening line that gets a CV forwarded.

---

## What Is Kubernetes?

Kubernetes is software that manages where and how applications run across multiple computers. Instead of installing software directly on a machine, you describe what you want -- "run three copies of this web server, each with 2GB of RAM, restart any that crash" -- and Kubernetes makes it happen and keeps it that way.

The computers Kubernetes manages are called **nodes**. The software running on those nodes is packaged into **containers** -- sealed, self-contained bundles that include the application and everything it needs to run. Containers are the reason reproducibility is possible: the same container image always contains exactly the same software, version for version.

I have two nodes. The HP Omen is the **control plane** node -- it is the brain that makes scheduling decisions and runs the Kubernetes management software. The Beelink mini PC is the **worker** node -- it runs storage services and provides the persistent disk space.

---

## What Is Talos?

Talos is the operating system running on both my nodes. A normal Linux server has SSH access, a package manager, a shell, and many ways to log in and make changes. Talos has none of those. It is a locked-down, immutable OS designed specifically for running Kubernetes. The only way to change anything on it is through its API using the `talosctl` command-line tool.

This matters for regulation because every change to the machine must go through a defined, documented path. You cannot log in and make a quiet manual change. Every configuration change is a YAML file, applied via API, trackable in version control.

---

## What Is ArgoCD?

ArgoCD is a tool that watches a Git repository and keeps the Kubernetes cluster in sync with what is defined in that repository. When I push a change to the repo, ArgoCD detects it and applies the change to the cluster automatically. When someone tries to manually change something on the cluster directly, ArgoCD detects the drift and can revert it.

This is called **GitOps**. The Git repository becomes the single source of truth for the entire cluster state. Every change has a timestamp, an author, and a diff. This is my change management mechanism -- directly satisfying the Annex 11 requirement for a controlled change process.

---

## What Is Forgejo?

Forgejo is a self-hosted Git server -- essentially a private GitHub that runs on my own cluster. Git is version control software: it tracks every change to every file, who made it, and when. Forgejo gives me a web interface, pull requests (which require review before merging), and webhooks (which tell ArgoCD to sync when something is pushed).

The reason I run my own instead of using GitHub is control and auditability. In a GxP environment, the source code repository is part of the validated system. I need to know exactly where it is, who has access, and that the access controls are mine to manage.

---

## What Is Authentik?

Authentik is an identity provider -- it is the single login system for the entire platform. Instead of having separate usernames and passwords for Forgejo, ArgoCD, and Grafana, every user has one account in Authentik and logs in everywhere through it. This is called **SSO (Single Sign-On)** and the technical standard it uses is called **OIDC (OpenID Connect)**.

GxP regulations require no shared accounts, role-based access, and a record of who logged in when. Authentik satisfies all three: every user is individual, access is controlled by group membership (platform-admin / developer / readonly), and every login is logged.

---

## What Is MinIO?

MinIO is an object storage server -- think of it as a private Amazon S3 running on my own hardware. S3 is the storage format that the bioinformatics pipeline software (Nextflow) natively understands for reading input data and writing output results. MinIO speaks the same language as S3, so the pipeline software does not know or care that it is talking to a server in my homelab instead of AWS.

MinIO also stores the log data collected by Loki. Moving Loki's storage to MinIO means all persistent data goes through one controlled storage layer rather than being scattered across local disk paths.

---

## What Is OPA Gatekeeper?

OPA Gatekeeper is a policy enforcement tool. It sits between the Kubernetes API and everything that runs on the cluster, and it checks every new workload against a set of rules before allowing it to start. If a workload violates a rule, it is blocked before it ever runs.

The rules I enforce are: every container must declare resource limits, every container image must be referenced by an exact fingerprint, images must come from approved sources, no container may run as privileged, and all workloads must have identifying labels. These rules map directly to EU GMP Annex 11 requirements.

---

## What Is Falco?

Falco is a runtime security tool. It runs as a background process on every node and watches every system call made by every container. When something happens that matches a rule -- someone shells into a running pipeline container, a file is written to an unexpected location, a privilege escalation is attempted -- Falco generates an audit event with a timestamp, the pod name, the user, and what happened.

These events flow to Loki via a router called Falcosidekick, and are queryable in Grafana. This creates the tamper-evident audit trail that Annex 11 Clause 9 requires. Because Falco operates at the kernel level using eBPF, application code cannot hide its own activity from it.

---

## What Is Nextflow and nf-core?

Nextflow is a workflow engine for bioinformatics. A workflow is a sequence of processing steps where the output of each step feeds into the next -- trim reads, align to reference genome, count gene expression, generate QC report. Nextflow lets you define these steps in a way that runs on a laptop, cloud, or Kubernetes cluster without changing the pipeline definition.

nf-core is a community that maintains validated, standardised bioinformatics pipelines built with Nextflow. The pipeline I am demonstrating is `nf-core/rnaseq` -- an RNA sequencing analysis pipeline widely used in drug discovery. I use the **test profile** which runs on a tiny synthetic dataset -- enough to prove the platform works without needing 32GB RAM or real patient data.

---

## What Is the IQ/OQ/PQ Documentation?

These are the three validation documents required by GxP regulations to prove a computer system is fit for its intended use.

**IQ -- Installation Qualification:** Proves the system was installed correctly. What software, at what version, on what hardware, by whom, verified how.

**OQ -- Operational Qualification:** Proves the system does what it is supposed to do. Scripted tests with defined pass/fail criteria, run against the actual system, results recorded. For example: "deploy a container with no resource limits -- Gatekeeper must reject it." Result: PASS or FAIL with actual output captured.

**PQ -- Performance Qualification:** Proves the system performs acceptably under expected load. Run the pipeline, record how long it took, how much CPU and RAM it used, verify the output matches across multiple runs (reproducibility test).

---

## What Are the SOPs?

SOP stands for Standard Operating Procedure. Written instructions for recurring operational tasks. In GxP environments, if there is no SOP for a process, that process is not considered controlled and evidence produced by it cannot be trusted.

My SOPs cover: Change Control, Audit Trail, Backup and Recovery, and Access Control. All four are in [[PH-07 GxP Validation Documentation]].

---

## What Is etcd?

etcd is a distributed key-value database that Kubernetes uses to store its entire state -- what nodes exist, what pods are running, what configuration is applied. If etcd data is lost or corrupted, the cluster does not know what it is supposed to be running. I take snapshots of etcd before each major change so I can restore the cluster to a known good state if something goes wrong. On a single-control-plane setup, this is the only disaster recovery path.

---

## What Is MetalLB?

MetalLB is a load balancer for Kubernetes clusters that do not run in a cloud environment. In AWS or Google Cloud, the cloud provider automatically assigns an external IP to services that need one. On bare metal, there is no cloud provider. MetalLB fills that role -- it assigns IPs from a pool I define (`192.168.0.200`) to services that need them.

---

## What Is Ingress-NGINX?

Ingress-NGINX is a reverse proxy and HTTP router. When a request comes in for `forgejo.homelab`, it routes to the Forgejo pods. When a request comes in for `grafana.homelab`, it routes to Grafana. One IP address, many services.

---

## What Is Cert-Manager?

Cert-Manager automatically creates and renews TLS certificates. TLS is what puts the `https://` in a URL and encrypts traffic. Instead of manually generating certificates, I define a ClusterIssuer and Cert-Manager handles generation and renewal for every service that needs one.

---

## What Is Sealed Secrets?

Sealed Secrets solves the problem of storing passwords and API keys in a Git repository without exposing them. I encrypt a secret using a public key tied to my cluster controller. The encrypted blob (the SealedSecret) can be committed to a public Git repository safely. Only the controller in my specific cluster can decrypt it.

---

## What Is Loki and What Is Grafana?

**Loki** is a log aggregation system. Every service generates logs -- text records of what it is doing. Loki collects all of these, stores them, and makes them searchable. Falco's audit events also flow to Loki, which is why Loki is where the GxP audit trail lives.

**Grafana** is the dashboard and query interface. It connects to Prometheus (metric data) and Loki (log data) and provides a web UI for querying and visualising both.

**Promtail** is the agent that runs on each node and ships pod logs into Loki. It is a DaemonSet -- one instance per node, always running.

---

## Page-by-Page Guide

### [[README]] -- The Hub
Main navigation page. Status table for all eight phases and the dependency chain. Start here when I need to know what is happening and what to do next.

### [[INF-01 Infrastructure Baseline]] -- My Hardware
Everything about the physical machines and network. Omen specs, Beelink specs, confirmed disk layout, what is already deployed, where config files live.

### [[INF-02 Architecture and Components]] -- The Blueprint
The full picture of the finished platform. Architecture diagram, component table with phase assignments, data flow from code push to pipeline output, namespace plan.

### [[INF-03 Infrastructure Analysis]] -- Honest Hardware Analysis
Utilization targets per node, six real limits I am working within, five things I initially got wrong in the design, what the ideal production version looks like.

### [[INF-04 System Tier Comparison]] -- Side-by-Side Tables
Tables comparing minimal / current / ideal across nodes, CPU, RAM, storage, networking, GxP compliance posture, pipeline performance, per-component resource cost, failure scenarios, and cost.

### [[INF-05 Hardware Scaling Guide]] -- How to Grow This
How to run this platform on any hardware from a single machine to a full production cluster. Tier identification, what to skip at lower tiers, node expansion, storage class migration, CNI migration, component re-enablement.

### [[PRJ-01 Strategic Rationale]] -- Why I Built This
The hiring argument. Why pharma companies in Basel and Zurich specifically have this gap, what the regulatory standards are and why they matter, how each phase maps to a transferable technical skill, CKA alignment.

### [[PRJ-02 Phase Map and Schedule]] -- The Schedule
Week-by-week schedule matching CKA study domains to project phases. Phase summaries. Dependency chain.

### [[PRJ-03 Project Significance]] -- The Industry Context
What problem drug development has with computational pipelines, who the end users are, why Kubernetes specifically, why Nextflow and nf-core, what this project is and is not.

### [[ADR-00 Decision Log]] -- Every Choice I Made
Twelve decisions documented with context, reasoning, and alternatives rejected. When someone asks "why Forgejo instead of GitLab?" this is where the full technical answer lives.

### [[ADR-01 Alternative Configurations]] -- What Else Could Work
Secondary options for every component, formatted as comparison tables showing when a different choice would be correct.

### [[OPS-01 Build Instructions]] -- Step by Step
The actual implementation guide. Each phase has numbered steps with exact commands, and a failure table listing the most likely problems with causes and resolutions.

### [[OPS-02 Reference Commands]] -- All the Commands
Every kubectl, talosctl, mc, ArgoCD, Helm, and Nextflow command used anywhere in the project, grouped by tool.

### [[OPS-03 Implementation Log]] -- Live Build Log
Running log of what I actually built, commands I ran, issues I hit, and how I resolved them. Filled in during the build, not before.

### [[PH-00 Cluster Preparation]] -- Foundation
Removing the control-plane taint, labeling nodes, applying the HDD mount patch, deploying local-path-provisioner, verifying the StorageClass, test PVC, etcd snapshot.

### [[PH-01 Forgejo]] -- Self-Hosted Git
Deploying Forgejo, connecting it to ArgoCD via webhook, sealing admin credentials, verifying SSH and HTTPS clone work.

### [[PH-02 Authentik SSO]] -- Single Sign-On
Deploying Authentik, creating OIDC providers for Forgejo, ArgoCD, and Grafana, creating the three access groups, validating SSO login.

### [[PH-03 MinIO Object Storage]] -- Object Storage
Deploying MinIO on the Beelink HDD, creating pipeline and logging buckets, setting lifecycle policy, migrating Loki to MinIO backend.

### [[PH-04 OPA Gatekeeper]] -- Policy Enforcement
Deploying Gatekeeper, installing constraints in warn mode, auditing violations, fixing them, flipping to deny mode one at a time.

### [[PH-05 Falco Runtime Security]] -- Audit Trail
Deploying Falco DaemonSet with eBPF driver, applying custom GxP audit rules, routing events to Loki, verifying audit events appear in Grafana.

### [[PH-06 Nextflow and nf-core]] -- Pipeline Run
Creating pipelines namespace with scoped RBAC, configuring Nextflow K8s executor, running nf-core/rnaseq test profile, validating output and audit trail.

### [[PH-07 GxP Validation Documentation]] -- Compliance Evidence
IQ, OQ, and PQ documents. Scripted OQ test cases with pass/fail results. PQ benchmark runs. Four SOPs. The document set that makes the whole platform a validated system.

---

## The Order Things Depend On

- The **StorageClass** (PH-00) is what lets every other service store data persistently. Without it, PVCs are stuck in Pending.
- **Forgejo** (PH-01) is required before any subsequent phase because all changes must go through Forgejo PRs per the Change Control SOP.
- **Authentik** (PH-02) needs Forgejo running first because its OIDC redirect URIs point at Forgejo's hostname.
- **MinIO** (PH-03) needs sealed secrets, working ingress, and TLS.
- **Gatekeeper** (PH-04) must come before Falco and Nextflow -- those workloads need to be validated at admission when they are created.
- **Falco** (PH-05) needs MinIO-backed Loki running, because audit events must persist across reboots.
- **Nextflow** (PH-06) needs MinIO for data, Gatekeeper active for admission validation, and Falco active for the audit trail.
- **PH-07 docs** reference evidence from all prior phases -- I start drafting from PH-04 but cannot finalize until PH-06 is complete.
