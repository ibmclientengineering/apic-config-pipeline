# apic-config-pipeline

A Tekton pipeline that automates the IBM API Connect **post-install Cloud Manager configuration checklist** as code, turning a fresh v10 install into a ready-to-use API platform.

> Part of the IBM Client Engineering **Cloud Pak for Integration production-deployment demo** — this repo handles the API Connect "day-2 wiring" step that comes right after the operator finishes installing the cluster.

## What this is

When API Connect v10 finishes installing, the platform components (DataPower gateway, analytics, developer portal) exist but are not yet registered with Cloud Manager, and there is no provider organization or catalog to publish APIs into. Historically an administrator clicked through the [Cloud Manager configuration checklist](https://www.ibm.com/docs/en/api-connect/10.0.x?topic=environment-cloud-manager-configuration-checklist) by hand. This repo replaces those clicks with a repeatable, parameterized pipeline so a rebuilt cluster comes back up as an **exact replica** of the original configuration.

## What's inside

- **`tekton/`** — the pipeline itself:
  - `pipeline.yaml` / `task.yaml` — a 3-step Tekton Task: `git-clone` the scripts, `initialize-config` (discovery), `config-apic` (the REST-driven configuration).
  - `config/` — three `Opaque` Secret templates you fill in: `apic-pipeline-git` (repo access), `apic-config-email-server` (SMTP), `apic-pipeline-provider-org` (the provider-org owner and catalog).
  - `clusterrole.yaml` / `rolebinding.yaml` — least-privilege RBAC granting the pipeline service account read-only access to OpenShift `routes` and `secrets`.
- **`scripts/`**:
  - `config.sh` — discovers all API Connect endpoints from OpenShift `routes`, downloads the APIC toolkit CLI, pulls the Cloud Manager admin password and toolkit credentials from cluster secrets, and writes a `config.json`.
  - `config_apicv10.py` — the 12 configuration steps: email server + sender, register the gateway / analytics / portal services, associate analytics with the gateway, create the provider organization and its owner, and associate the gateway with the **Sandbox** catalog.
  - `api_calls.py` / `utils.py` — a small REST client (bearer-token auth, retries) and config helpers.
- **`APIC_Config.drawio`** / **`diagram.png`** — the architecture diagram of the flow.

## Why it's built this way

- **The checklist as code.** Every manual Cloud Manager step becomes an idempotent, reviewable script instead of tribal knowledge in a runbook — the difference between "someone remembers how we set this up" and "it's in git."
- **Disaster-recovery and replication.** Because discovery reads live OpenShift routes and secrets, the same pipeline reconfigures *any* cluster into the same shape. Rebuild in a new region or after a failure and you get an exact replica, not a best-effort reconstruction.
- **Separation of duties via secrets.** SMTP details, provider-org identity, and git credentials live in Kubernetes Secrets, not in the scripts — so the pattern is safe to share and the sensitive values stay with whoever owns them.
- **Least privilege.** The bundled ClusterRole grants only `get/list/watch` on routes and secrets — the pipeline can read what it needs to discover the cluster and nothing more.
- **A foundation, not a replacement for GitOps.** v10 could not (yet) express this configuration declaratively, so this pipeline is the pragmatic bridge that gets you automation and auditability today.

> **Honest 2026 note.** On **API Connect 12.1**, a single `APIConnectCluster` custom resource **auto-registers the whole topology** (gateway, analytics, portal) at install time, so much of this pipeline is no longer needed for a standard deployment. Treat this repo as a **proven pattern to re-point** for older v10 estates, multi-AZ setups, or extra provider-org/catalog wiring — not a turnkey run. Note also that the reference `pipeline.yaml` still points its script source at a **now-retired GitHub Enterprise repo** (`github.ibm.com/techaeta/...`); re-point that URL before using it.

## How it fits the bigger picture

This is the **configure** stage of the API Connect track. Its companion **[`apic-publish-pipeline`](https://github.com/ibmclientengineering/apic-publish-pipeline)** then publishes the API and Product definitions from **[`apic-products-apis-yaml`](https://github.com/ibmclientengineering/apic-products-apis-yaml)** into the Sandbox catalog this pipeline just created — so *platform wiring* and *API delivery* stay cleanly separated.

Zoom out and API Connect is one capability in the wider Cloud Pak for Integration demo, governed by the GitOps backbone (**`multi-tenancy-gitops`**, **`-infra`**, **`-services`**, **`-apps`**) and sitting alongside the App Connect Enterprise (**`ace-infra`**, **`ace-config`**) and MQ (**`mq-infra`**, **`mq-qm01`**) factory-and-overlay repos that deliver the rest of the integration stack.

Maintained by IBM Client Engineering.
