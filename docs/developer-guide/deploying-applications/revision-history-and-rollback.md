---
title: Revision History and Rollback
description: How OpenChoreo records what each environment ran, and how to roll an environment back to an earlier deployment
---

# Revision History and Rollback

Every environment a component or resource is deployed to keeps a history of what it ran. Each entry is a **revision**: a frozen copy of the environment's binding spec, numbered, labelled with why it was recorded, and tracked for health while it is the one running. When a deployment goes wrong you can roll the environment back to any earlier revision from Backstage, the `occ` CLI, the API, or an AI agent through MCP.

## Releases and Binding Revisions

Two different things are versioned when you deploy:

- A **release** (ComponentRelease or ResourceRelease) is an immutable snapshot of the component or resource definition. A new release is cut when the definition changes, and the same release is promoted through every environment.
- A **binding** (ReleaseBinding or ResourceReleaseBinding) is one per environment. It says which release runs there, plus the environment-specific settings: overrides, and for components the `state` (`Active` or `Undeploy`).

A **binding revision** records one binding spec, so it captures both: the release _and_ the environment's settings. Changing only an override, without a new release, records a new revision. Promoting the same release to another environment starts that environment's own history.

| Binding                  | Revision kind                    | Short name in `occ` | Release field in the revision |
| ------------------------ | -------------------------------- | ------------------- | ----------------------------- |
| `ReleaseBinding`         | `ReleaseBindingRevision`         | `rbr`               | `spec.componentReleaseName`   |
| `ResourceReleaseBinding` | `ResourceReleaseBindingRevision` | `rrbr`              | `spec.resourceReleaseName`    |

```text
ComponentRelease ─┐
                  ├─► ReleaseBinding (one per environment) ─► ReleaseBindingRevision r1, r2, r3 …
overrides, state ─┘
```

Revisions are created and updated only by the controller. Their spec is immutable, they live in the binding's namespace, and they are owned by the binding, so deleting a binding deletes its history.

### Names and Numbers

Each binding numbers its revisions on its own: the first deploy to an environment is `r1`, and numbers keep increasing from there. A revision has two names:

| Name             | Format           | Example                   | Where you see it                                 |
| ---------------- | ---------------- | ------------------------- | ------------------------------------------------ |
| **Display name** | `<release>-r<N>` | `frontend-v7-r1`          | Backstage, `occ` tables, MCP results             |
| **Object name**  | `<binding>-r<N>` | `frontend-development-r1` | `kubectl`, YAML, the API's `revisionName` fields |

The object name is unique per environment; the display name tells you which release the revision ran.

Numbers are never reused. When a revision is [deleted](#deleting-a-revision) or removed by [retention](#retention), the others keep their numbers and the gap stays: delete `r2` from `r1 r2 r3` and the next revision is still `r4`. Each binding records the highest number it has used in `status.highestRevision`, and new revisions are numbered above it.

### What a Revision Records

| Field                                               | Description                                                                                                       |
| --------------------------------------------------- | ----------------------------------------------------------------------------------------------------------------- |
| `spec.revision`                                     | The revision number, counting from 1 per binding                                                                  |
| `spec.cause`                                        | Why it was recorded (see below)                                                                                   |
| `spec.componentReleaseName` / `resourceReleaseName` | The release the binding pointed to                                                                                |
| `spec.snapshotHash`                                 | SHA-256 of the binding spec, ignoring `releasePin`                                                                |
| `spec.bindingSpec`                                  | A full copy of the binding spec; this is what a rollback restores                                                 |
| `status.active`                                     | `true` for exactly one revision: the one the binding runs now                                                     |
| `status.health`                                     | `Progressing`, `Healthy`, `Degraded`, `Failed` or `Unknown`                                                       |
| `status.knownGood`                                  | Set once the revision stayed `Healthy` for the [known-good window](#health-and-known-good-revisions); never unset |
| `status.observedAt`, `status.healthySince`          | When the health last changed, and since when it has been `Healthy`                                                |

The binding itself reports its active revision's object name in `status.currentRevision`.

## When a Revision Is Recorded

On every reconcile the controller compares the binding spec with its history:

- **A spec it has not seen before** creates the next revision.
- **A spec it has already recorded** (a rollback, or any edit back to an earlier spec) makes that existing revision active again under its own name and number. No new revision is created, and the binding gets a `RevisionReactivated` event.
- **A change to `releasePin` alone** is not a new spec and records nothing.

A new revision's `cause` describes how it differs from the one it replaces:

| Cause      | Recorded when                                                                                                   |
| ---------- | --------------------------------------------------------------------------------------------------------------- |
| `Deploy`   | The binding's first revision, or the release changed                                                            |
| `Promote`  | The first revision of a binding created by promotion with `occ component deploy <component> --to <environment>` |
| `Override` | Anything else changed: overrides, trait configs, workload overrides, retain policy                              |
| `Undeploy` | The binding's `state` changed to `Undeploy`, or its first revision is undeployed (ReleaseBindings only)         |
| `Redeploy` | The binding's `state` changed from `Undeploy` back to `Active` (ReleaseBindings only)                           |

## Viewing Revision History

### In Backstage

Open a component (or resource), go to the **Deploy** tab, and select an environment. The side panel has a **Revisions** section:

- The picker shows the active revision, with a line such as `6h ago · Healthy · Deploy` underneath.
- Opening the picker lists the four newest revisions; type to search by number (`r3`), display name, release, cause or health. The active revision carries an **Active** chip and known-good revisions a shield icon.
- **Open Revision Browser** (or **…and N more in Revision Browser**) shows the whole history.

On a component, a revision whose ComponentRelease has been deleted is shown struck through, marked `release deleted`, and cannot be selected for rollback.

### With occ

```bash
occ releasebindingrevision list --namespace default --binding frontend-development
```

```text
NAME                     CAUSE     HEALTH    ACTIVE  KNOWN-GOOD  AGE
frontend-v2-r5           Deploy    Healthy   true    true        1h
frontend-20260803-1-r4   Deploy    Healthy   false   true        3h
frontend-v4-r3           Override  Degraded  false   false       1d
...
```

Leave out `--binding` to list every binding's revisions in the namespace, with a BINDING and ENVIRONMENT column; narrow it with `--project`, `--component` and `--environment`. For resources, use `occ resourcereleasebindingrevision list` (with `--binding`, or `--project`, `--resource` and `--environment`). The binding name is shown by `occ releasebinding list`. See the [CLI reference](../../reference/cli-reference.md#releasebindingrevision) for every flag.

### With the API

| Method | Path                                                                                          |
| ------ | --------------------------------------------------------------------------------------------- |
| `GET`  | `/api/v1/namespaces/{namespace}/releasebindingrevisions?component={c}&environment={e}`        |
| `GET`  | `/api/v1/namespaces/{namespace}/releasebindingrevisions/{revisionName}`                       |
| `GET`  | `/api/v1/namespaces/{namespace}/resourcereleasebindingrevisions?resource={r}&environment={e}` |
| `GET`  | `/api/v1/namespaces/{namespace}/resourcereleasebindingrevisions/{revisionName}`               |

`environment` is optional; lists are paginated with `limit` and `cursor`, ordered by environment and then newest first. Reading revisions needs the `releasebinding:view` or `resourcereleasebinding:view` permission.

## Inspecting and Comparing Revisions

The **Revision Browser** lists every revision on the left. Selecting one shows when it was created, its release, cause and health, and two tabs:

- **View** shows the full revision object as YAML (`apiVersion`, `kind`, `metadata`, `spec`, `status`), as `kubectl` would, with a copy button.
- **Compare** shows a side-by-side diff against another revision. **Compare with** defaults to the active revision, which is on the left; the selected revision is on the right, and unchanged lines are collapsed.

When reading a diff, `spec.bindingSpec` is what a rollback would change: a different `releaseName` means a different release, and differences in the environment configs are override changes. Differences in `metadata` and `status` (names, `active`, `health`) are expected between any two revisions.

From the CLI, print one revision in full and compare two with `diff`:

```bash
occ releasebindingrevision get frontend-development-r3 --namespace default --binding frontend-development -o yaml

diff \
  <(occ releasebindingrevision get -n default --binding frontend-development --revision 3 -o yaml) \
  <(occ releasebindingrevision get -n default --binding frontend-development --revision 5 -o yaml)
```

## Rolling Back

A rollback restores the binding spec an earlier revision recorded. It is always previewed first.

### Undo: Back to the Previous Revision

Without `--revision`, `occ` rolls a binding back to the revision that was active **immediately before** the current one, like `kubectl rollout undo`:

```bash
occ releasebinding rollback frontend-development --namespace default            # preview, then confirm
occ releasebinding rollback frontend-development --namespace default --dry-run  # preview only
occ resourcereleasebinding rollback orders-db-development --namespace default   # resources
```

"Previous" follows activation history, not revision numbers. A rollback makes an old revision active again, so the previous revision can be numbered higher than the current one:

| Step                    | Active | Previous |
| ----------------------- | ------ | -------- |
| Deploy `r1` … `r5`      | `r5`   | `r4`     |
| `rollback --revision 3` | `r3`   | `r5`     |
| `rollback` (undo)       | `r5`   | `r3`     |
| `rollback` (undo)       | `r3`   | `r5`     |

Each time a revision becomes active, the controller writes on it `openchoreo.dev/previous-revision: "<N>"`, the number of the revision it replaced (`"0"` for the first). The preview names the target, and after you confirm, `occ` applies that exact revision number.

The undo is refused, and nothing changes, when:

- the binding has only one revision, or the current one was its first;
- the previous revision has been deleted. It is never replaced by another revision; name one with `--revision`;
- the binding has a change the controller has not recorded yet. Retry in a moment.

Revisions recorded before this tracking existed have no `previous-revision` annotation. For them, the target is the revision activated just before the current one. If revisions are missing from such a history, a deleted one may have been active in between, so the preview shows the warning `PreviousRevisionInferred`; check the target before confirming. `--revision 0` is an error; leave the flag out instead.

The undo is available in `occ` and the API (omit `revision`). Backstage and the MCP tools always name a revision.

### In Backstage

1. In the **Revisions** section, pick the revision to restore, then click **Roll back to r*N***. You can also click **Roll back to r*N*** in the Revision Browser.
2. The confirmation dialog runs a dry run and lists the spec fields that will change (current → target) and every warning. Nothing has changed yet.
3. Click **Roll back** to apply it. A notification confirms, for example _Rolled back Development to revision 2: frontend-v5-r2 is active again._

The button is disabled, with a tooltip saying why, when the revision is already active, when its release has been deleted (components), or when you do not have permission to roll back.

### With occ

```bash
# Preview only: print the changes and warnings, change nothing
occ releasebinding rollback frontend-development --namespace default --revision 2 --dry-run

# Preview, then confirm at the prompt
occ releasebinding rollback frontend-development --namespace default --revision 2

# Apply without a prompt (scripts); the warnings are still printed
occ releasebinding rollback frontend-development --namespace default --revision 2 --yes
```

Without a terminal on stdin, `occ` refuses to roll back unless `--yes` is given. Resources use `occ resourcereleasebinding rollback <binding>` with the same flags.

### With the API

```bash
curl -X POST "$OPENCHOREO_API/api/v1/namespaces/default/releasebindings/frontend-development/rollback" \
  -H "Authorization: Bearer $TOKEN" -H "Content-Type: application/json" \
  -d '{"revision": 2, "dryRun": true}'
```

Omit `revision` to roll back to the previously active revision (see [Undo](#undo-back-to-the-previous-revision)). Resources use `POST /api/v1/namespaces/{namespace}/resourcereleasebindings/{name}/rollback`. The response has the target's object name (`revisionName`) and display name (`revisionDisplayName`), `revision`, the previously active number (`revertedFrom`), `dryRun`, `changes` and `warnings`. Applying a rollback needs the `releasebinding:rollback` or `resourcereleasebinding:rollback` permission; the default `developer`, `sre` and `platform-engineer` roles have both. When [audit logging](../../platform-engineer-guide/audit-logging.mdx) is enabled, every rollback call, from the API or MCP, is recorded as a `rollback_release_binding` or `rollback_resource_release_binding` event, and the record's resource metadata has `dryRun: true` for a preview and `dryRun: false` for an applied rollback.

### Warnings

Warnings never block a rollback; they tell you what you are restoring.

| Warning                    | Meaning                                                                                                                                                                        |
| -------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------ |
| `TargetNotKnownGood`       | The target never stayed `Healthy` for the known-good window                                                                                                                    |
| `TargetUnhealthy`          | The target was `Degraded` or `Failed` when it stopped being active                                                                                                             |
| `NoSpecChange`             | The target records the current spec; nothing changes                                                                                                                           |
| `UnrecordedChange`         | The binding has a spec change the controller has not recorded yet; the rollback discards it                                                                                    |
| `BindingWillBePinned`      | `releasePin` changes from `Auto` to `Pinned` (see [pinning](#automatic-deployment-and-pinning))                                                                                |
| `ProjectCellDrift`         | The project in this environment is now bound to a ProjectRelease created after the target revision                                                                             |
| `TargetUndeploys`          | The target's `state` is `Undeploy`, so rolling back removes the component from the environment (ReleaseBindings only)                                                          |
| `SecretReferenceDrift`     | A SecretReference the target uses no longer exists, no longer has a key it uses, or was recreated after the target revision (ReleaseBindings only)                             |
| `PreviousRevisionInferred` | No revision was named and the current revision predates activation tracking; the target was taken from activation order and a deleted revision may have been active in between |

### Errors

| Status | Code                      | What happened and what to do                                                                                                                      |
| ------ | ------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------- |
| 403    | -                         | You lack the rollback permission for this binding and environment                                                                                 |
| 404    | -                         | The binding or the revision does not exist                                                                                                        |
| 409    | `RELEASE_DELETED`         | The release the target revision ran has been deleted, so its spec cannot be restored. Pick another revision                                       |
| 409    | `CONCURRENT_MODIFICATION` | The binding changed between the preview and the apply. Nothing was written; review the new preview and apply again (Backstage re-runs it for you) |
| 409    | `NO_PREVIOUS_REVISION`    | No revision was named and there is no previous revision to return to, or it was deleted. Name a revision                                          |

Releases are immutable, so a revision can only be restored while its release still exists. Backstage marks such component revisions in advance. For resources it does not, and the dry run reports `RELEASE_DELETED` in the confirmation dialog.

### What Happens When You Roll Back

The API checks the binding and target, then writes the binding once, conditional on the version it read:

1. `spec` is replaced with the target's `bindingSpec`, and `spec.releasePin` is set to `Pinned`.
2. No revision is written by the API. On its next reconcile, the controller sees that the binding's spec matches the target's `snapshotHash` and makes the **existing** target revision active again. It keeps its name and number, its health restarts at `Progressing`, and an existing known-good mark is kept.
3. The previously active revision becomes inactive, keeping the health it had.
4. `status.currentRevision` is set to the target, and the restored spec is rendered and applied to the data plane like any other change. Health is then tracked until it settles.

### Bindings Managed by GitOps

When a binding's YAML lives in a [GitOps repository](../../platform-engineer-guide/gitops/overview.md), Git is the source of truth, and revision history works with it:

- Every change Flux applies from Git is recorded like any other change: a new release records a `Deploy` revision, changed overrides an `Override` revision, and `state: Undeploy` an `Undeploy` revision. Returning Git to a spec that was already recorded (for example with `git revert`) makes that revision active again instead of creating a new one.
- A rollback through Backstage, `occ`, the API or MCP changes only the cluster, not Git. On its next reconcile, Flux applies the fields Git declares again, so the rollback is undone and the revision matching Git becomes active again. Until then the rolled-back release runs, typically for no longer than the Kustomization's `interval`.
- `releasePin` does not stop Flux. If Git does not declare `releasePin`, the binding stays `Pinned` after Flux restores Git's release; if Git declares it, Flux restores that value too.

To roll back a GitOps-managed binding, change Git: revert the commit, or set `releaseName` and the overrides to the target revision's `spec.bindingSpec`. To hold a rollback during an incident before Git is updated, suspend the Kustomization that manages the binding (`flux suspend kustomization <name>`), roll back, then update Git and resume it (`flux resume kustomization <name>`).

## Health and Known-Good Revisions

The controller tracks health on the **active** revision only, from the binding's conditions and its RenderedRelease:

| Health        | Meaning                                                                                                                                    |
| ------------- | ------------------------------------------------------------------------------------------------------------------------------------------ |
| `Progressing` | Being rolled out: not applied yet, resources still starting, or a Job still running. An undeploy is `Progressing` until resources are gone |
| `Healthy`     | All resources are ready for this spec (or, for an undeploy, removed)                                                                       |
| `Degraded`    | Resources were applied but report degraded                                                                                                 |
| `Failed`      | The spec could not be rendered or applied, a Job failed, or resources are not ready                                                        |
| `Unknown`     | Resource readiness has not been reported                                                                                                   |

A revision becomes **known-good** once it has stayed `Healthy` continuously for the known-good window, 60 seconds by default. Leaving `Healthy` restarts the window. Once set, `knownGood` stays set, even if the revision later degrades. Revisions that undeploy a component are never known-good, since there is nothing to roll back to. When a revision stops being active, its health is frozen at its last value, which is what `TargetUnhealthy` reports.

Prefer the newest known-good revision that is not active as a rollback target.

In Backstage, the revision list refreshes every 5 seconds while any revision is `Progressing` or has no health yet. After a rollback it also refreshes until the target shows as active, for up to 60 seconds, so you see `Progressing` turn into `Healthy` without reloading.

## Automatic Deployment and Pinning

Both binding kinds have `spec.releasePin`:

| Value            | Meaning                                                           |
| ---------------- | ----------------------------------------------------------------- |
| `Auto` (default) | The release may be advanced automatically                         |
| `Pinned`         | The release is held at its current value until someone changes it |

For components with auto-deploy enabled, a new ComponentRelease normally moves the first environment's ReleaseBinding to it. A `Pinned` binding is skipped. A rollback pins the binding so that the next auto-deploy does not immediately undo it. You can still deploy or promote a release to a pinned binding yourself; the binding stays pinned.

Nothing sets `releasePin` back to `Auto` for you. When the problem is fixed and you want auto-deploy to resume, set it explicitly:

```bash
kubectl patch releasebinding frontend-development -n default --type merge \
  -p '{"spec":{"releasePin":"Auto"}}'
```

Changing `releasePin` does not record a revision.

:::note
ResourceReleaseBindings are never advanced automatically; they change only when you promote or update them. A rollback still sets `releasePin: Pinned` on a ResourceReleaseBinding, but no controller acts on it today.
:::

## Retention

Each binding keeps at most 10 revisions. When a new revision is recorded and the history is over the limit, the oldest are deleted, except:

- the active revision,
- the revision that was active before it, the target of an [undo](#undo-back-to-the-previous-revision),
- the newest known-good revision, the rollback target of last resort,
- any revision annotated `openchoreo.dev/keep: "true"`.

To keep a revision indefinitely:

```bash
kubectl annotate releasebindingrevision frontend-development-r3 -n default openchoreo.dev/keep=true
```

If the protected revisions alone exceed the limit, they are all kept and the binding gets a `RevisionHistoryLimitExceeded` warning event.

Platform engineers can change the limit and the known-good window with the control plane chart's `controllerManager.manager.revisionHistoryLimit` (default `10`; `0` keeps every revision) and `controllerManager.manager.revisionKnownGoodWindow` (default `"60s"`; `"0"` marks a revision known-good on its first `Healthy` observation). See the [control plane Helm reference](../../reference/helm/control-plane.mdx).

## Deleting a Revision

You can remove one old revision from a binding's history. It can no longer be rolled back to; the binding, its active revision and the running workload are not touched, and the other revisions keep their numbers.

```bash
occ releasebindingrevision delete r2 --binding frontend-development --namespace default --dry-run  # check only
occ releasebindingrevision delete r2 --binding frontend-development --namespace default            # shows it, asks [y/N]
occ resourcereleasebindingrevision delete r3 --binding orders-db-development --namespace default
```

In Backstage, open the Revision Browser, highlight an inactive revision and click **Delete r*N***. The dialog shows the revision, binding, environment and status, checks with the server that it may be deleted, and deletes it only when you confirm. Numbers missing from the history then appear as **Unavailable** rows.

The API is `DELETE /api/v1/namespaces/{namespace}/releasebindingrevisions/{name}` (resources: `…/resourcereleasebindingrevisions/{name}`), with optional `binding` and `dryRun` query parameters. It needs the `releasebinding:delete-revision` or `resourcereleasebinding:delete-revision` permission; the default roles that can delete bindings have it.

A revision cannot be deleted (409 `REVISION_PROTECTED`) when it is:

- active,
- the newest known-good revision,
- annotated `openchoreo.dev/keep: "true"`,
- the target of a rollback that is being applied, or
- not yet fully recorded by the controller.

The delete is conditional on the revision as checked, so a revision that becomes active in the meantime is not deleted (409 `CONCURRENT_MODIFICATION`). The previously active revision can be deleted by hand; an undo then reports that it was deleted. Deleting revisions directly with `kubectl` skips these checks.

## AI Agents (MCP)

The control plane MCP server's Deployment toolset includes six revision tools:

| Tool                                      | Arguments                                                                     |
| ----------------------------------------- | ----------------------------------------------------------------------------- |
| `list_release_binding_revisions`          | `namespace_name`, `component_name`, optional `environment`, `limit`, `cursor` |
| `get_release_binding_revision`            | `namespace_name`, `name` (object name, e.g. `frontend-development-r3`)        |
| `rollback_release_binding`                | `namespace_name`, `name` (ReleaseBinding name), `revision`, `dry_run`         |
| `list_resource_release_binding_revisions` | `namespace_name`, `resource_name`, optional `environment`, `limit`, `cursor`  |
| `get_resource_release_binding_revision`   | `namespace_name`, `name` (object name)                                        |
| `rollback_resource_release_binding`       | `namespace_name`, `name` (ResourceReleaseBinding name), `revision`, `dry_run` |

`dry_run` defaults to `true`, so an agent's first rollback call only previews. The tool descriptions tell the agent to:

1. pick the newest known-good revision that is not active, unless you ask for another;
2. call the rollback tool with `dry_run` left at `true`;
3. show you the target, the `changes` and every warning, and ask you to confirm;
4. only then call again with the same `revision` and `dry_run: false`.

The tools use the same permissions as the API. They always name a revision: undo and revision deletion are not available to agents. For example, ask: _"Roll back frontend in development to the last known-good revision."_

## Related

- [Deploy and Promote](./deploy-and-promote.md)
- [CLI reference: releasebindingrevision](../../reference/cli-reference.md#releasebindingrevision)
- [MCP servers](../../reference/mcp-servers.mdx)
- [ReleaseBinding](../../reference/api/platform/releasebinding.md) and [ResourceReleaseBinding](../../reference/api/platform/resourcereleasebinding.md) API reference
