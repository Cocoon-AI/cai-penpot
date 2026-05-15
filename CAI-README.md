# Cocoon AI tenant fork — cai-penpot

The **deploy target** for Penpot on Cocoon AI's platform. Tenant fork
of [`Cocoon-AI/foreign-penpot`](https://github.com/Cocoon-AI/foreign-penpot) (the
base fork), per [`cai-portal` ADR 0004](https://github.com/Cocoon-AI/cai-portal/blob/main/docs/adrs/0004-foreign-app-pattern.md)
Phase 5b + D15.

> See [`README.md`](./README.md) for upstream Penpot. This file
> documents the **Cocoon AI overlay** only.

## What this repo is

- A real git fork of `Cocoon-AI/foreign-penpot` (which is itself a fork of
  `penpot/penpot`).
- Carries the same platform-integration overlay as the base fork +
  tenant-specific manifest values (`deployment.account: cai-apps`,
  `observability.service_name: cai-penpot`, etc.).
- **Deploys to `cai-apps`** on ECS Fargate per ADR D14a.

## Lineage

```
penpot/penpot (upstream)
  ↓ rebased on cadence
Cocoon-AI/foreign-penpot (base fork)
  ↓ rebased on cadence — this fork's parent
Cocoon-AI/cai-penpot (this repo — Cocoon's tenant fork; the deploy target)
```

`cai-upstream-track.yml` in this repo merges from `Cocoon-AI/foreign-penpot`
(the base), not from upstream directly. That keeps the rebase work
concentrated in the base fork.

## Daily ops

| Task | Command |
|---|---|
| Deploy | `cai foreign deploy <tag>` |
| Tail logs (Loki via Grafana primary; CloudWatch fallback) | `cai foreign logs -f` |
| Status | `cai foreign status --json` |
| Sync from base fork | `cai foreign upgrade` |

See `cai-portal/docs/runbooks/foreign-app-deploy.md`.

## SSO

Native OIDC against the `cai-portal` Keycloak realm. The whole
hostname `penpot.cocoon-ai.com` is carved out of ALB OIDC
(`auth.alb_oidc: false`) so Penpot does its own OIDC without
double-prompting.

Keycloak groups `penpot-{admin,member,viewer}` map to Penpot's
`admin/editor/viewer` roles via the `groups` claim mapper per ADR D5.

## Observability

`{job="cai-penpot"}` in Grafana Loki returns this app's logs alongside
`{job="cai-portal"}` etc. — same naming convention native apps use.
JVM auto-instrumentation agent (ADR D10 Pattern 2) emits traces via
the `cai-otel-collector` sidecar.

## Deploy state

**Not deployed yet.** Phase 5b's terraform-apply gate is held until
the base fork's Dockerfile TODOs are resolved + the foreign-app
composite Terraform module is extracted into `cai-infra` (currently
the Terraform/main.tf references a not-yet-extracted path). The
`cai foreign deploy` workflow path is wired and CI-testable on PRs.

The path to first deploy:

1. Resolve Dockerfile TODOs in this fork (Penpot backend build
   command + OTel javaagent version pin).
2. Extract the foreign-app Terraform composite module into
   `cai-infra/infra/terraform/modules/foreign-app/`.
3. `terraform apply` from this repo's `Terraform/` against
   `cai-apps`.
4. Provision Keycloak `penpot` client via `just kcadm`.
5. First sign-in E2E.

These are captured against acceptance criteria #1, #2, #3 in
`cai-portal/docs/specs/0004-foreign-app-pattern.md`.
