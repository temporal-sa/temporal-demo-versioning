# Runbook: Worker Versioning demo (Pizza Tracker)

This runbook is for presenting the demo. The [README](README.md) explains what it is and how it's built.

## What it shows

A pizza-ordering service ships new Workflow code while orders are in flight. Each order is a `PizzaOrder` Workflow, and three Worker versions run side by side: v1 (4 steps), v2 (adds a quality check) and v3 (a drone delivery step that always fails). Worker Versioning routes new orders between them, and every order finishes on the version it started on.

The demo makes three problems visible:

- **Changing a Workflow's steps can break the orders already running.** v2 adds a step. Ramping and promoting it never changes the shape of an order already in flight: v1 orders keep 4 steps, and new orders get 5. Beats 2 and 3 show it, and there is no patching in the Workflow code.
- **A bad release should hit a small slice of traffic, and stop when you say so.** v3 goes out to 25% of new orders and those orders go red. One rollback sends every new order back to v2. Beats 4 and 5.
- **The orders a bad release already took still need a way out.** **Fix** rewinds one stuck order and pins it to the healthy version, where it runs from the start and completes. Beat 6.

## Before you start

**On tmprl-demo.cloud,** open **Pizza Worker Versioning** from the catalog. The dashboard is public; there is no sign-in. All three Worker versions are already running, so you never ship a version during the talk; you only route traffic.

The hosted demo is one shared deployment. Another presenter's ramp or promote shows up on your screen and yours on theirs. Before you present, check the **Workflows** panel and put it back to v1 if needed (see [Reset and tear down](#reset-and-tear-down)).

**To run it locally,** you need [Docker](https://www.docker.com/) with Compose v2. The Compose stack runs its own Temporal dev server, so you need no account and no API key.

## Start it

### On tmprl-demo.cloud

Open the dashboard. A healthy start shows:

- The **Workflows** panel: `v1` **CURRENT**, `v2` and `v3` **INACTIVE**, each with an "in flight" count.
- **Live orders** filling with `v1` cards, each with a 4-step stepper: New, Cook, Out, Done.

Order publishing stops on its own 15 minutes after it starts, so the board may say "Order publishing is paused." Press **Resume orders** in the **Live orders** header. A new order starts every 6 seconds, and each v1 order takes about a minute.

### Locally

From the repository root:

1. `make app-up` builds and starts Temporal, the dashboard backend and the v1 Worker. It stays attached to the logs; leave it running.
2. In a second terminal, start the other two versions: `make app-v2`, then `make app-v3`.
3. Open http://localhost:8080. The Temporal UI is at http://localhost:8233.

Locally the Worker Deployment is called `pizza` and the Build IDs are `v1-local`, `v2-local` and `v3-local`. On tmprl-demo.cloud the deployment is `pizza-worker` and the Build IDs are `v1`, `v2` and `v3`. The dashboard shows the friendly labels in both places.

The backend promotes v1 to Current on its own when it starts (see [What's demo-only](#whats-demo-only)), so orders flow without any click.

## The walkthrough

The whole story takes 6 to 8 minutes; most of it is waiting for orders to move. Press `Esc` to close any modal.

### 1. Steady state on v1

1. Point at **Live orders**. Each card shows the order ID, a version badge, the pizza and a stepper.
2. Point at the **Workflows** panel: `v1` is **CURRENT** and its bar is full.
3. Say: "Every card is a Workflow. They're all running the v1 code: New, Cook, Out, Done. v2 and v3 are already deployed, and right now they get no traffic."

### 2. Ramp v2 to 25%

1. Click **Deploy**. The modal opens with the Current version selected.
2. Pick `v2` under **Target version**. The slider reads **Ramping 25%**. Click **Apply**.
3. The `v2` chip changes to **RAMPING 25%**. Over the next half minute, about one new order in four arrives with a `v2` badge and a 5-step stepper: New, Cook, QC, Out, Done.
4. Point at a v1 order that is part-way through. It keeps its 4 steps.
5. Say: "v2 adds a quality check. A quarter of new orders get it. Orders that started on v1 are pinned there and finish the way they started. Nobody wrote compatibility code for the old shape."

Which orders land in the 25% depends on the Workflow ID, so the share is close to a quarter, not exact.

### 3. Promote v2

1. Click **Deploy**. `v2` is selected because it's ramping. Drag the slider to the last stop (**Ramping 100%**) and click **Apply**. The note under the slider says it: "At 100% this promotes the version to Current."
2. `v2` becomes **CURRENT** and `v1` changes to **DRAINING**. Every new order is v2.
3. Watch the `v1` "in flight" count fall to 0 as its last orders finish, still with 4 steps.
4. Say: "v2 is now the default. v1 stays up only to finish its own orders. When its count reaches zero, it can be shut down."

You can stop at 50% on the way if you want to show a second ramp step.

### 4. Ship a bad release to 25%

1. Click **Deploy**, pick `v3`, leave the slider at **Ramping 25%**, and click **Apply**.
2. About a quarter of new orders arrive as `v3`. Their stepper reads New, Cook, QC, Drone, Done.
3. About 45 seconds after each v3 order starts, it reaches **Drone**, the card turns red, and a **Fix** button appears on it. v2 orders keep completing.
4. Say: "v3 swapped the courier for a drone, and the drone is broken. Temporal keeps retrying that step, so these orders aren't lost; they're stuck. And it's only a quarter of new traffic."

### 5. Roll back

1. Click **Rollback** (it's enabled only while a version is ramping), then **Roll back** in the confirmation.
2. The ramp drops to 0. `v2` stays **CURRENT** with the full bar, and `v3` changes to **DRAINING**. No new order starts on v3.
3. The red orders stay red. A v3 order that started just before the rollback can still turn red in the next 45 seconds, because it was already pinned to v3.
4. Say: "Rollback is one routing change. It caps the damage at the orders v3 already took, but it doesn't heal them. They're pinned to v3."

### 6. Recover the stuck orders

1. Click **Fix** on a red card. A toast says "Recovered order-…".
2. The card goes back to New with a `v2` badge and a 5-step stepper, and it completes about a minute later. Its elapsed time keeps counting from the original start.
3. Fix the others the same way. When none are left, `v3` shows 0 in flight.
4. Say: "Fix rewinds this order to its first step and pins it to v2 in one call. It runs the v2 code from the start, so there's nothing for it to be incompatible with."
5. For proof, describe the recovered order with the Temporal CLI, pointed at the demo's Namespace:

   ```bash
   temporal workflow describe -w order-<id>
   ```

   Its **Versioning Info** shows `Behavior Pinned`, the v2 Build ID, and `OverrideBehavior Pinned`. Its history starts with a `WorkflowTaskFailed` (the reset point) and a `WorkflowExecutionOptionsUpdated` (the override). Locally you can also open it in the Temporal UI at http://localhost:8233.

To see the routing state from a terminal at any point:

```bash
temporal worker deployment describe --name pizza-worker   # locally: --name pizza
```

## Show it in code

All links point at commit [`8564792`](https://github.com/temporal-sa/temporal-demo-versioning/commit/8564792884216cf9b83c6dffdc8232999af50c74). Open a file locally with `code -g <path>:<line>`.

### Give each Worker a version and pin its Workflows

**When to use it:** at beat 1, or when someone asks what makes a Worker "versioned".

**Where it is:** [`cmd/worker/main.go` lines 65–74](https://github.com/temporal-sa/temporal-demo-versioning/blob/8564792884216cf9b83c6dffdc8232999af50c74/cmd/worker/main.go#L65-L74) (`code -g cmd/worker/main.go:65`)

```go
w := worker.New(c, taskQueue, worker.Options{
	DeploymentOptions: worker.DeploymentOptions{
		UseVersioning: true,
		Version: worker.WorkerDeploymentVersion{
			DeploymentName: deploymentName,
			BuildID:        buildID,
		},
		DefaultVersioningBehavior: workflow.VersioningBehaviorPinned,
	},
})
```

**What you need to understand:**

- A Worker Deployment Version is a deployment name plus a Build ID. Both come from environment variables: `TEMPORAL_DEPLOYMENT_NAME` and `TEMPORAL_WORKER_BUILD_ID`. On tmprl-demo.cloud each of the three Worker components sets them in the catalog entry; on Kubernetes the Temporal Worker Controller injects them.
- `UseVersioning: true` means this Worker only gets Tasks that routing sends to its version. Starting a Worker doesn't give it traffic; v2 and v3 sit at **INACTIVE** until you ramp or promote them.
- `DefaultVersioningBehavior` applies to any Workflow registered without its own behavior. Each version here also sets Pinned when it registers its Workflow. If neither is set, registration fails.
- All three versions poll the same Task Queue, `pizza`. Routing between them is server-side, by version, not by queue name.
- A Build ID has to change whenever the code changes. In the Kubernetes setup the controller derives it from the pod template, which is why the README pins images to digests.

**Docs:** [Worker Versioning](https://docs.temporal.io/production-deployment/worker-deployments/worker-versioning), [Versioning definitions](https://docs.temporal.io/worker-versioning#versioning-definitions)

### Change the Workflow's steps without patching

**When to use it:** at beat 2, when v2 orders show the extra QC step, or when someone asks how the old orders survive the change.

**Where it is:** [`internal/pizza/workflow_v2.go` lines 40–43](https://github.com/temporal-sa/temporal-demo-versioning/blob/8564792884216cf9b83c6dffdc8232999af50c74/internal/pizza/workflow_v2.go#L40-L43) (`code -g internal/pizza/workflow_v2.go:40`) and [lines 59–63](https://github.com/temporal-sa/temporal-demo-versioning/blob/8564792884216cf9b83c6dffdc8232999af50c74/internal/pizza/workflow_v2.go#L59-L63)

```go
state.CurrentStep = 2
if err := workflow.ExecuteActivity(ctx, QualityCheck, in).Get(ctx, nil); err != nil {
	return err
}
```

```go
func RegisterV2(w worker.Worker) {
	w.RegisterWorkflowWithOptions(PizzaOrderV2, workflow.RegisterOptions{
		Name:               WorkflowTypeName,
		VersioningBehavior: workflow.VersioningBehaviorPinned,
	})
```

**What you need to understand:**

- v2 inserts a new Activity between Cook and Out for delivery. Replaying a v1 order's history against this code would fail, because the history and the code disagree about what comes after Cook. Pinning means that never happens: a v1 order's Tasks only go to v1 Workers.
- There is no `GetVersion` call and no branch on the old shape. Each version's code is simply the code for that version.
- Every version registers under the same Workflow type name, `PizzaOrder` ([`internal/pizza/types.go` lines 6–8](https://github.com/temporal-sa/temporal-demo-versioning/blob/8564792884216cf9b83c6dffdc8232999af50c74/internal/pizza/types.go#L6-L8)). The client starts `PizzaOrder` and never knows which version it got.
- The repo keeps all three shapes side by side (`workflow_v1.go`, `workflow_v2.go`, `workflow_v3.go`) so one binary can play any version in the demo. In a real service, v2 replaces v1 in the same file, and the old Build ID keeps running until its Workflows drain.
- Pinned suits Workflows that finish in minutes, like these orders. A Workflow that runs for weeks would keep its old version alive the whole time; that's the case for Auto-Upgrade, which moves running Workflows to the new version and so needs replay-safe changes.

**Docs:** [Roll out and pin Workflows](https://docs.temporal.io/production-deployment/worker-deployments/worker-versioning/roll-out-and-pin)

### Start orders without picking a version

**When to use it:** at beat 2, when someone asks how the client decides which version an order goes to.

**Where it is:** [`internal/publishing/publisher.go` lines 91–99](https://github.com/temporal-sa/temporal-demo-versioning/blob/8564792884216cf9b83c6dffdc8232999af50c74/internal/publishing/publisher.go#L91-L99) (`code -g internal/publishing/publisher.go:91`)

```go
id++
in := pizza.OrderInput{OrderID: id, Pizza: pizza.Menu[id%len(pizza.Menu)]}
opts := client.StartWorkflowOptions{
	ID:        fmt.Sprintf("order-%d", id),
	TaskQueue: p.taskQueue,
}
if _, err := p.c.ExecuteWorkflow(ctx, opts, pizza.WorkflowTypeName, in); err != nil {
	p.logger.Warn("failed to start order", "orderId", id, "err", err)
}
```

**What you need to understand:**

- The start options name a Task Queue and nothing about versions. Temporal sends the first Workflow Task to the Current version, or to the Ramping version for a share of Workflows set by the ramp percentage.
- Whether a given order falls in the ramp is decided from its Workflow ID, so the same ID always gets the same answer at a given percentage.
- Once a v2 Worker completes the order's first Workflow Task and reports Pinned, the order stays on v2. Its Activities go to v2 Workers too.
- The client never changes during a rollout. Ramp, promote and rollback are routing settings on the server.

**Docs:** [Versioning definitions](https://docs.temporal.io/worker-versioning#versioning-definitions)

### Ramp, promote and roll back are routing calls

**When to use it:** at beats 2, 3 and 5, or when someone asks what the **Deploy** and **Rollback** buttons actually do.

**Where it is:** [`internal/dashboard/actions.go` lines 88–93](https://github.com/temporal-sa/temporal-demo-versioning/blob/8564792884216cf9b83c6dffdc8232999af50c74/internal/dashboard/actions.go#L88-L93) (ramp, `code -g internal/dashboard/actions.go:88`), [lines 56–60](https://github.com/temporal-sa/temporal-demo-versioning/blob/8564792884216cf9b83c6dffdc8232999af50c74/internal/dashboard/actions.go#L56-L60) (promote) and [lines 272–275](https://github.com/temporal-sa/temporal-demo-versioning/blob/8564792884216cf9b83c6dffdc8232999af50c74/internal/dashboard/actions.go#L272-L275) (rollback)

```go
// Ramp: the 25% and 50% stops
if _, err := a.handle().SetRampingVersion(ctx, client.WorkerDeploymentSetRampingVersionOptions{
	BuildID:                 buildID,
	Percentage:              pct,
	AllowNoPollers:          true,
	IgnoreMissingTaskQueues: true,
}); err != nil {
// ...

// Promote: the 100% stop
if _, err := a.handle().SetCurrentVersion(ctx, client.WorkerDeploymentSetCurrentVersionOptions{
	BuildID:                 buildID,
	AllowNoPollers:          true,
	IgnoreMissingTaskQueues: true,
}); err != nil {
// ...

// Rollback: clear the ramp
if _, err := a.handle().SetRampingVersion(ctx, client.WorkerDeploymentSetRampingVersionOptions{
	BuildID:    "",
	Percentage: 0,
}); err != nil {
```

**What you need to understand:**

- Each button is one call on the Worker Deployment handle. The same three operations are `temporal worker deployment set-ramping-version`, `set-current-version`, and `set-ramping-version --delete`; the README lists the exact commands.
- Rollback clears the ramp, so 100% of new Workflows go to Current. It's safe here because a Current version is always set. It doesn't touch Workflows already pinned to the ramping version.
- `AllowNoPollers` and `IgnoreMissingTaskQueues` turn off two server-side safety checks: that the target version has had pollers, and that it polls every Task Queue the Current version did. The dashboard sets them because a freshly started Worker may not have polled yet. A production pipeline would usually leave them off so the server refuses a version that isn't ready.
- On Kubernetes, the [Temporal Worker Controller](https://docs.temporal.io/production-deployment/worker-deployments/kubernetes-controller) can make these calls for you with a progressive rollout. The README's Kubernetes setup runs it in Manual mode so the presenter drives every step; on tmprl-demo.cloud there is no controller, and the dashboard makes every call.

**Docs:** [Rolling out changes with the CLI](https://docs.temporal.io/production-deployment/worker-deployments/worker-versioning/roll-out-and-pin#rolling-out-changes-with-the-cli)

### Move a stuck order to the healthy version (reset-with-move)

**When to use it:** at beat 6, or when someone asks how a Pinned Workflow ever leaves a bad version.

**Where it is:** [`internal/dashboard/actions.go` lines 324–356](https://github.com/temporal-sa/temporal-demo-versioning/blob/8564792884216cf9b83c6dffdc8232999af50c74/internal/dashboard/actions.go#L324-L356) (`code -g internal/dashboard/actions.go:324`)

```go
req := &workflowservice.ResetWorkflowExecutionRequest{
	Namespace:                 a.namespace,
	WorkflowExecution:         exec,
	Reason:                    "recover stuck orders: bad drone-delivery build",
	RequestId:                 uuid.NewString(),
	WorkflowTaskFinishEventId: resetPoint,
	// ...
	PostResetOperations: []*workflowpb.PostResetOperation{{
		Variant: &workflowpb.PostResetOperation_UpdateWorkflowOptions_{
			UpdateWorkflowOptions: &workflowpb.PostResetOperation_UpdateWorkflowOptions{
				WorkflowExecutionOptions: &workflowpb.WorkflowExecutionOptions{
					VersioningOverride: &workflowpb.VersioningOverride{
						Override: &workflowpb.VersioningOverride_Pinned{
							Pinned: &workflowpb.VersioningOverride_PinnedOverride{
								Behavior: workflowpb.VersioningOverride_PINNED_OVERRIDE_BEHAVIOR_PINNED,
								Version: &deploymentpb.WorkerDeploymentVersion{
									DeploymentName: a.deploymentName,
									BuildId:        goodBuild,
								},
							},
						},
					},
				},
				UpdateMask: &fieldmaskpb.FieldMask{Paths: []string{"versioning_override"}},
			},
		},
	}},
}
```

**What you need to understand:**

- One Reset call does two things atomically: it rewinds the order to its first completed Workflow Task, and it sets a Pinned versioning override to the Current Build ID (`goodBuild`), so every Task of the reset run goes to v2.
- Rewinding to the start is what makes the move safe. v2's code never has to replay v3's history, because the reset run keeps none of it. The trade-off is that the order repeats steps it already did; the steps here only wait, but real Activities would run again.
- A versioning override on its own (`temporal workflow update-options`) moves a Workflow without rewinding it. That works only when the target version's code can replay the existing history, which v2 can't for a v3 order that reached the drone.
- The dashboard fixes one order per click. For a real incident, `temporal workflow reset with-workflow-update-options --query ...` does the same for every Workflow on the bad Build ID in one batch.

**Docs:** [Moving a pinned Workflow](https://docs.temporal.io/production-deployment/worker-deployments/worker-versioning/roll-out-and-pin#moving-a-pinned-workflow), [Recover pinned Workflows after a bad rollout](https://docs.temporal.io/production-deployment/worker-deployments/recover-pinned-workflows)

### Why the bad orders stay red instead of failing

**When to use it:** at beat 4, when someone asks why the v3 orders don't just fail.

**Where it is:** [`internal/pizza/workflow_v3.go` lines 43–46](https://github.com/temporal-sa/temporal-demo-versioning/blob/8564792884216cf9b83c6dffdc8232999af50c74/internal/pizza/workflow_v3.go#L43-L46) and [lines 69–74](https://github.com/temporal-sa/temporal-demo-versioning/blob/8564792884216cf9b83c6dffdc8232999af50c74/internal/pizza/workflow_v3.go#L69-L74) (`code -g internal/pizza/workflow_v3.go:43`)

```go
ctx = workflow.WithActivityOptions(ctx, workflow.ActivityOptions{
	StartToCloseTimeout: stepDwell + 15*time.Second,
	RetryPolicy:         &temporal.RetryPolicy{MaximumAttempts: 0, MaximumInterval: droneRetryMaxInterval},
})
// ...
state.CurrentStep = 3
state.Failing = true
droneCtx := workflow.WithScheduleToCloseTimeout(ctx, droneRetryWindow)
if err := workflow.ExecuteActivity(droneCtx, DroneDelivery, in).Get(droneCtx, nil); err != nil {
	return err
}
```

**What you need to understand:**

- `MaximumAttempts: 0` means unlimited attempts, and the backoff is capped at 30 seconds. The drone Activity fails every time, so the order stays Running and retrying: stuck, but recoverable.
- The schedule-to-close timeout of 15 minutes (`droneRetryWindow`) bounds all attempts together. An order nobody fixes fails after 15 minutes and leaves the board, so the demo doesn't keep retrying forever.
- This is an Activity failure, not a non-determinism error. The bug is in v3's new step, which is the kind of regression Pinned versions and rollback are for.

**Docs:** [Retry Policies](https://docs.temporal.io/encyclopedia/retry-policies)

### What's demo-only

When someone asks what they'd change for their own system, these are the parts to leave out:

- **The steps only wait.** [`internal/pizza/activities.go` lines 18–28](https://github.com/temporal-sa/temporal-demo-versioning/blob/8564792884216cf9b83c6dffdc8232999af50c74/internal/pizza/activities.go#L18-L28): each step sleeps 15 seconds, the final one 7, and each drone attempt 5. Replace the bodies with real work and the versioning code doesn't change.
- **The order generator.** [`internal/publishing/workflow.go` lines 1–14](https://github.com/temporal-sa/temporal-demo-versioning/blob/8564792884216cf9b83c6dffdc8232999af50c74/internal/publishing/workflow.go#L1-L14): an unversioned Workflow on its own Task Queue starts an order every 6 seconds, stops after 15 minutes, and terminates the orders still open 10 minutes after that. It keeps live traffic on screen and caps what an unattended demo costs.
- **The friendly version labels.** [`cmd/worker/main.go` lines 92–97](https://github.com/temporal-sa/temporal-demo-versioning/blob/8564792884216cf9b83c6dffdc8232999af50c74/cmd/worker/main.go#L92-L97): each Worker writes `pizzaVersion` into its Worker Deployment Version metadata so the dashboard can show `v1`/`v2`/`v3` instead of Build IDs.
- **v1 promotes itself at startup.** [`internal/dashboard/actions.go` lines 213–222](https://github.com/temporal-sa/temporal-demo-versioning/blob/8564792884216cf9b83c6dffdc8232999af50c74/internal/dashboard/actions.go#L213-L222): when no Current version is set, the backend promotes the version labelled v1. A real rollout sets the first Current version from its deploy pipeline or the Worker Controller.
- **The dashboard queries every open order each second.** [`internal/dashboard/temporal_reader.go` lines 80–82](https://github.com/temporal-sa/temporal-demo-versioning/blob/8564792884216cf9b83c6dffdc8232999af50c74/internal/dashboard/temporal_reader.go#L80-L82): it lists up to 50 running orders and runs the `getState` Query on each. That's fine for a demo board, not for a production fleet.
- **One binary holds three Workflow shapes.** In production each Build ID holds one version of the code.

## Reset and tear down

- **Between runs (hosted or local):**
  1. If a version is ramping, click **Rollback**, then **Roll back**.
  2. Click **Deploy**, pick `v1`, drag the slider to 100%, and click **Apply**. v1 is Current again and v2 drains.
  3. Click **Fix** on any red cards, or leave them: they fail on their own after 15 minutes.
  4. Wait for the `v2` and `v3` "in flight" counts to reach 0 before the next audience, so the board shows only v1.
- **To stop starting orders:** press **Pause orders**. Orders already in flight carry on. Ten minutes later, any order still open is terminated; press **Resume orders** before then to keep them.
- **If you walk away from the hosted demo:** publishing stops on its own after 15 minutes, stuck orders are closed 10 minutes after that, and an unfixed v3 order fails after 15 minutes of retries. Nothing keeps consuming Actions indefinitely, but press **Pause orders** when you finish so it stops sooner.
- **Locally:** `make app-down` stops and removes the whole stack, including the v2 and v3 Workers. The dev server keeps nothing, so the next `make app-up` starts clean.

## When something goes wrong

| What you see | Likely cause | Fix |
|---|---|---|
| "Order publishing is paused." and no cards | Publishing timed out after 15 minutes, or someone paused it | Press **Resume orders** |
| The board already shows v2 or v3 as Current or ramping | Another presenter used the shared hosted demo | Follow [Reset and tear down](#reset-and-tear-down) and wait for the counts to drain |
| No **Deploy** or **Rollback** buttons | Fewer than two versions are registered; locally, the v2 and v3 Workers aren't running | Locally, run `make app-v2` and `make app-v3`; wait a few seconds for them to register |
| **Rollback** is greyed out | Nothing is ramping. Rollback only clears a ramp | To go back after a promote, use **Deploy**, pick the old version, slider to 100%, **Apply** |
| A toast says "Ramp failed", "Promote failed" or "Rollback failed" | The backend couldn't reach Temporal or the call was rejected; the toast carries the server's error | Read the error, retry; check the backend logs (`make app-logs` locally) |
| v3 orders keep turning red after the rollback | They started on v3 before the rollback and reached the drone step later | Expected for about 45 seconds after the rollback; fix them as they appear |
| **Fix** shows "Recover failed: dashboard: no current version to recover onto" | No Current version is set | Promote a version with **Deploy** at 100%, then click **Fix** again |
| `v1` stays **DRAINING** after its orders finish | Expected: a version that was Current shows as draining or drained, and both display as **DRAINING** | Nothing to fix |

## Keeping this runbook current

The code links point at a fixed commit. When a linked file changes, update its link, line numbers, and snippet together. Before an event, list the linked files that have changed:

```sh
grep -oE 'blob/[0-9a-f]{40}/[^#)]+' RUNBOOK.md | sort -u | while IFS=/ read -r _ sha file; do
  git diff --quiet "$sha" HEAD -- "$file" || echo "changed since ${sha:0:7}: $file"
done
```
