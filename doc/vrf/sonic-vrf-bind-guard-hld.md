# SONiC VRF bind guard

## Revision history

| Rev | Date       | Author     | Change Description |
| --- | ---------- | ---------- | ------------------ |
| 0.1 | 2026-09-28 | Bojun-Feng | Initial draft. Describes the VRF bind guard and interface retirement approach. |

## Purpose

A Linux interface can enter a new virtual routing and forwarding instance (VRF) while orchagent still owns the old router interface (RIF). New route or neighbor work can then use the old RIF and add references that prevent its removal.

The guard separates the Linux binding change from RIF readiness. IntfMgr waits for IntfsOrch to confirm the guard is active before changing the Linux binding. IntfsOrch retires the old interface: it coordinates removal of old neighbors, old interface prefixes, and the old RIF. While the guard is active, IntfsOrch keeps new interface work queued and dependency owners keep dependency work queued. After retirement, IntfsOrch releases the guard when the current request permits it. Ordinary processing can then create the new RIF and retry the queued work.

The design preserves the existing CLI and CONFIG_DB API. UNBIND and BIND remain separate operations, and UNBIND can finish without a later BIND. The guard is keyed by interface alias, such as Ethernet0; steady-state route leaking continues to use the existing route and next-hop model.

## Ownership and completion

The Linux interface, recorded interface state, interface prefixes, and RIF are distinct:

- **Linux interface**: the device whose VRF attachment IntfMgr changes with `master` or `nomaster`.
- **Recorded interface state**: IntfMgr's interface-level STATE_DB entry, including its `vrf` field. Address entries have separate recorded address state.
- **Interface prefixes**: address entries processed through APPL_DB and tracked by IntfsOrch. An interface-level entry is distinct from its interface prefixes.
- **Old RIF / new RIF**: the router interface objects that orchagent removes during retirement and creates for the new Linux binding, respectively.

| Owner | Responsibility |
| --- | --- |
| IntfMgr | Change the Linux binding, publish interface work to APPL_DB, and update recorded interface state. |
| IntfsOrch | Maintain the guard, retire the old interface, and create the new RIF. |
| Dependency owners | RouteOrch, NeighOrch, and next-hop-group owners manage their existing object references and retry their queued work. NeighOrch removes old neighbors during retirement. |

CLI return, recorded interface state, and RIF readiness are separate completion points. The CLI returns after saving the configuration in CONFIG_DB. IntfMgr updates recorded interface state without waiting for old RIF removal. “UNBIND complete” requires the Linux interface to be detached, the old RIF removed, and the guard released. “BIND complete” requires the Linux interface and recorded interface state to use the target VRF, the guard released, and the new RIF ready.

“Old RIF removed”, “new RIF ready”, and “Optional neighbor and next hop ready” describe orchagent object state after successful Switch Abstraction Interface (SAI)/API handling. Forwarding must be validated separately.

## Guard and retirement

The guard establishes two ordering rules: IntfMgr must receive confirmation that the guard is active before changing the Linux binding, and IntfsOrch must complete old interface retirement before creating the new RIF. Confirmation means an acknowledgment for the current request in state `guarded` or `retired`; retirement need not have finished.

While the guard is active, IntfsOrch keeps interface-level and interface-prefix SETs (requests to create or update entries) queued. Dependency owners check the guard before creating or reusing objects, including cached neighbors, next hops, and next-hop groups. When a dependency owner tries to use the RIF, IntfsOrch returns “RIF is not ready”, and the dependency owner keeps the work queued for retry. The guard blocks new references to the old RIF. Cleanup and withdrawal paths can still find the old RIF and release existing references. The owner of guard-blocked work must neither submit it for hardware programming nor count it as successfully processed in a bulk operation.

In “Check and retire any old interface”, IntfsOrch first checks that all ports are ready, even when no old interface or RIF remains. For an old interface, it asks NeighOrch to “Remove old neighbors and release their RIF references”. IntfsOrch then removes any remaining old interface prefixes, performs proxy-ARP cleanup where applicable, and removes the old RIF. Existing reference counts determine when removal can finish. Dependency owners independently release next-hop references and RIF references for withdrawn work; the Linux binding operation does not generate those withdrawals.

If a retirement attempt is incomplete, IntfsOrch keeps retirement queued for retry. The guard remains active until retirement completes and the current request permits guard release. The owners responsible for cleanup use existing SAI error handling, including handling of terminal errors.

## Lifecycle examples

The three charts illustrate Ethernet0 UNBIND and BIND to a named VRF. Consumers process published APPL_DB operations asynchronously, so independent consumers can run between the steps shown. The charts show successful Linux binding operations and object changes; waits for readiness or blocking references can repeat. Completion depends on readiness, release of blocking references, and successful cleanup or creation.

### UNBIND

For Ethernet0, `config interface vrf unbind` removes the interface configuration and configured addresses. IntfMgr removes the Linux addresses and defers interface removal while counted Linux addresses remain. The address count excludes IPv6 link-local addresses. For addresses other than IPv4 link-local, IntfMgr also requests removal of the interface prefix and removes the recorded address state. IPv4 link-local address removal is local to Linux in this path. IntfsOrch may process interface-prefix removal before the guard is active or during retirement.

After confirming that no counted Linux addresses remain and that the guard is active, IntfMgr detaches the Linux interface from the old VRF. It then requests removal of the old interface, reports that the Linux binding changed, and removes the recorded interface state. Before these final updates, IntfMgr checks that applicable link-local neighbor cleanup succeeded; a failure leaves interface removal pending. IntfsOrch completes retirement asynchronously, subject to port readiness, release of blocking references, and successful cleanup. UNBIND requires no future target VRF or BIND request.

![UNBIND intent: guard confirmation precedes Linux detachment; old RIF removal and guard release complete asynchronously.](vrf-bind-guard/lifecycles/render/01-unbind-intent.png)

[UNBIND intent — full-size PNG](vrf-bind-guard/lifecycles/render/01-unbind-intent.png) · [SVG](vrf-bind-guard/lifecycles/render/01-unbind-intent.svg) · [Editable Mermaid](vrf-bind-guard/lifecycles/source/01-unbind-intent.mmd)

<details>
<summary>Mermaid source</summary>

```mermaid
sequenceDiagram
    participant M as IntfMgr
    participant O as IntfsOrch
    participant D as Dependency owners
    Note over M,D: Start: Ethernet0 is attached to its old VRF.<br/>CLI returns after saving the configuration.
    opt Interface removal runs before address removal
    M->>M: Defer interface removal while counted Linux addresses remain
    end
    loop For each configured address
    M->>M: Remove the Linux address
    opt Address is not IPv4 link-local
    M->>O: Request removal of the interface prefix
    M->>M: Remove the recorded address state
    end
    end
    Note over M,D: The address count excludes IPv6 link-local addresses.
    M->>M: Confirm no counted Linux addresses remain
    M->>O: Ask IntfsOrch to activate the guard
    O->>O: Set the guard active for this request
    O-->>M: Confirm the guard is active
    M->>M: Detach the Linux interface from the old VRF
    M->>O: Request removal of the old interface
    M->>O: Report that the Linux binding changed
    M->>M: Remove the recorded interface state
    opt Queued dependency work runs while the guard is active
    D->>O: Try to use the RIF
    O-->>D: RIF is not ready
    D->>D: Keep work queued for retry
    end
    loop While ports are not ready
    O->>O: Check and retire any old interface
    O->>O: Keep retirement queued for retry
    end
    opt Reference-blocked attempt (may repeat, guard stays active)
    O->>O: Check and retire any old interface
    O->>O: Keep retirement queued for retry
    end
    opt Dependency owners independently withdraw the blocking work
    D->>D: Release next-hop references for withdrawn work
    D->>O: Release RIF references for withdrawn work
    end
    opt Ports ready, blocking references withdrawn, cleanup succeeds
    O->>O: Check and retire any old interface
    O->>D: Remove old neighbors<br/>and release their RIF references
    opt If old interface prefixes remain
    O->>O: Remove old interface prefixes
    end
    O->>O: Remove the old RIF
    O->>O: Release the guard
    Note over M,D: UNBIND complete: Linux interface detached.<br/>Old RIF removed#59; guard released.
    end
```

</details>

Sub-port UNBIND removes configured addresses and the old interface configuration, waits for old recorded interface state to disappear, then recreates the sub-port configuration in the default VRF by omitting the VRF field. IntfsOrch must still remove the old RIF before creating the new RIF. This sub-port recreation is not part of the Ethernet0 UNBIND shown above.

### Clean BIND

A clean BIND starts with the Linux interface detached, recorded interface state absent, no old RIF, and the guard inactive. VrfBlue is the target VRF in the example. Before saving the new interface configuration, `config interface vrf bind` removes configured addresses and the old interface configuration, then waits for old recorded interface state to disappear. The CLI does not wait for old RIF removal. IntfMgr checks interface and VrfBlue state readiness before asking IntfsOrch to activate the guard. After confirming that the guard is active, IntfMgr attaches the Linux interface to VrfBlue, requests creation of the new interface, reports that the Linux binding changed, and records the new interface state.

IntfsOrch must still perform “Check and retire any old interface”: all ports must be ready before it can confirm that no old interface or RIF remains and release the guard. During ordinary interface processing, IntfsOrch then checks that ports and VrfBlue are ready. With the port available and successful SAI/API handling, IntfsOrch creates the new RIF, completing BIND.

NeighOrch can then process independently supplied neighbor work. With a valid neighbor IP, a usable MAC, and successful processing, NeighOrch uses the new RIF to create the neighbor and next hop. Address configuration and neighbor learning occur independently as needed; BIND itself supplies neither replacement addresses nor a neighbor request.

![Clean BIND intent: separate Linux attachment, recorded interface state, guard release, and new RIF readiness; neighbor work is optional.](vrf-bind-guard/lifecycles/render/02-clean-bind-intent.png)

[Clean BIND intent — full-size PNG](vrf-bind-guard/lifecycles/render/02-clean-bind-intent.png) · [SVG](vrf-bind-guard/lifecycles/render/02-clean-bind-intent.svg) · [Editable Mermaid](vrf-bind-guard/lifecycles/source/02-clean-bind-intent.mmd)

<details>
<summary>Mermaid source</summary>

```mermaid
sequenceDiagram
    participant M as IntfMgr
    participant O as IntfsOrch
    participant D as Dependency owners
    Note over M,D: Start: Linux interface detached#59; no old RIF.<br/>VrfBlue exists. Guard inactive.
    Note over M,D: CLI returns after saving the configuration.<br/>The old recorded interface state is already absent.
    loop While interface or VrfBlue state is not ready
    M->>M: Check interface and VrfBlue state readiness
    end
    M->>O: Ask IntfsOrch to activate the guard
    O->>O: Set the guard active for this request
    O-->>M: Confirm the guard is active
    M->>M: Attach the Linux interface to VrfBlue
    M->>O: Request creation of the new interface
    M->>O: Report that the Linux binding changed
    M->>M: Record the new interface state
    opt New interface work runs while the guard is active
    O->>O: Keep work queued for retry
    end
    opt Queued dependency work runs while the guard is active
    D->>O: Try to use the RIF
    O-->>D: RIF is not ready
    D->>D: Keep work queued for retry
    end
    loop While ports are not ready
    O->>O: Check and retire any old interface
    O->>O: Keep retirement queued for retry
    end
    opt Ports ready and no old interface or RIF remains
    O->>O: Check and retire any old interface
    O->>O: Release the guard
    loop While ports or VrfBlue are not ready
    O->>O: Check that ports and VrfBlue are ready
    end
    opt Port available and new RIF creation succeeds
    O->>O: Create the new RIF
    Note over M,D: BIND complete: Linux interface and recorded interface state use VrfBlue.<br/>Guard released#59; new RIF ready.
    opt Independently supplied eligible neighbor work succeeds
    Note over O,D: Eligible neighbor: IP and usable MAC.<br/>Address setup / learning as needed<br/>is independently supplied.
    D->>O: Try to use the RIF
    O-->>D: Use the new RIF
    D->>D: Create the neighbor and next hop
    Note over O,D: Optional neighbor and next hop ready.
    end
    end
    end
```

</details>

### BIND while earlier UNBIND retirement is pending

Here the Linux interface is already detached and recorded interface state is absent, but the guard is active and old interface retirement is pending. IntfsOrch transfers the active guard to the new BIND request without a gap. Once IntfMgr has confirmation for that request, it can attach the Linux interface to VrfBlue and record the new interface state while old interface retirement remains pending. IntfsOrch keeps new interface work queued, and dependency owners keep dependency work queued, until retirement completes and the guard is released.

IntfsOrch completes the earlier UNBIND retirement once ports are ready, dependency owners have withdrawn blocking work, and cleanup succeeds. The old neighbors, remaining old interface prefixes, and old RIF belong to the earlier UNBIND; IntfsOrch releases the guard using the validated current request. New RIF creation and optional neighbor work then follow the same conditions as clean BIND.

![BIND while earlier UNBIND retirement is pending — intent: the active guard transfers to the current request, preserving earlier UNBIND retirement before new RIF creation.](vrf-bind-guard/lifecycles/render/03-bind-waits-for-unbind-intent.png)

[BIND while earlier UNBIND retirement is pending — intent, full-size PNG](vrf-bind-guard/lifecycles/render/03-bind-waits-for-unbind-intent.png) · [SVG](vrf-bind-guard/lifecycles/render/03-bind-waits-for-unbind-intent.svg) · [Editable Mermaid](vrf-bind-guard/lifecycles/source/03-bind-waits-for-unbind-intent.mmd)

<details>
<summary>Mermaid source</summary>

```mermaid
sequenceDiagram
    participant M as IntfMgr
    participant O as IntfsOrch
    participant D as Dependency owners
    Note over M,D: Start: Linux interface detached#59; recorded interface state absent.<br/>Guard active#59; old interface retirement pending. VrfBlue exists.
    Note over M,D: CLI returns after saving the configuration.<br/>The old recorded interface state is already absent.
    loop While interface or VrfBlue state is not ready
    M->>M: Check interface and VrfBlue state readiness
    end
    M->>O: Ask IntfsOrch to activate the guard
    O->>O: Set the guard active for this request
    O-->>M: Confirm the guard is active
    M->>M: Attach the Linux interface to VrfBlue
    M->>O: Request creation of the new interface
    M->>O: Report that the Linux binding changed
    M->>M: Record the new interface state
    opt New interface work runs while the guard is active
    O->>O: Keep work queued for retry
    end
    opt Queued dependency work runs while the guard is active
    D->>O: Try to use the RIF
    O-->>D: RIF is not ready
    D->>D: Keep work queued for retry
    end
    loop While ports are not ready
    O->>O: Check and retire any old interface
    O->>O: Keep retirement queued for retry
    end
    opt Reference-blocked attempt (may repeat, guard stays active)
    O->>O: Check and retire any old interface
    O->>O: Keep retirement queued for retry
    end
    opt Dependency owners independently withdraw the blocking work
    D->>D: Release next-hop references for withdrawn work
    D->>O: Release RIF references for withdrawn work
    end
    opt Ports ready, blocking references withdrawn, cleanup succeeds
    rect rgb(232, 240, 247)
    Note over O,D: Earlier UNBIND retirement<br/>and the guard release it enables
    O->>O: Check and retire any old interface
    O->>D: Remove old neighbors<br/>and release their RIF references
    opt If old interface prefixes remain
    O->>O: Remove old interface prefixes
    end
    O->>O: Remove the old RIF
    O->>O: Release the guard
    end
    loop While ports or VrfBlue are not ready
    O->>O: Check that ports and VrfBlue are ready
    end
    opt Port available and new RIF creation succeeds
    O->>O: Create the new RIF
    Note over M,D: BIND complete: Linux interface and recorded interface state use VrfBlue.<br/>Guard released#59; new RIF ready.
    opt Independently supplied eligible neighbor work succeeds
    Note over O,D: Eligible neighbor: IP and usable MAC.<br/>Address setup / learning as needed<br/>is independently supplied.
    D->>O: Try to use the RIF
    O-->>D: Use the new RIF
    D->>D: Create the neighbor and next hop
    Note over O,D: Optional neighbor and next hop ready.
    end
    end
    end
```

</details>

## Request protocol

The guard uses internal APPL_DB and STATE_DB tables. IntfMgr retains one current request per interface alias and assigns each new request a `request_id` greater than the one in the retained request record. IntfMgr saves the request in STATE_DB before publishing an APPL_DB notification. IntfsOrch validates the notification's ID and action against the retained request before processing it.

| Record | Writer and contents |
| --- | --- |
| APPL_DB `INTF_GUARD_TABLE:<alias>` | IntfMgr publishes `id` and `action`: `prepare`, `applied`, or `cancel`. |
| STATE_DB `INTERFACE_GUARD_TABLE\|<alias>` | IntfMgr writes `request_id`, `action`, `target_vrf`, `kernel_pending`, `applied_id`, and `applied_vrf`. IntfsOrch writes acknowledgment `id` and `state`, plus `retired_id` after retirement. |
| STATE_DB notification `INTF_GUARD_ACK` | IntfsOrch wakes IntfMgr after writing the acknowledgment. IntfMgr checks the acknowledgment ID and state in the retained request record. |

The actions connect the lifecycle operations to the retained request:

- **`prepare`** implements “Ask IntfsOrch to activate the guard”. IntfsOrch validates the request, performs “Set the guard active for this request”, and writes the acknowledgment state. IntfMgr proceeds only with a matching `guarded` or `retired` acknowledgment.
- **`applied`** implements “Report that the Linux binding changed”. IntfMgr publishes ordinary interface work and records `applied_id` and `applied_vrf`. `applied_vrf` records the applied request’s target VRF, or empty for `nomaster`. IntfsOrch then completes retirement and releases the guard for the current request.
- **`cancel`** ends a `prepare` request that did not change the Linux binding. Any earlier applied request's unfinished retirement still has to complete before guard release.

The acknowledgment state describes progress: `guarded` means the guard is active; `retired` means old interface retirement has completed but the guard is still active; `released` means the guard is inactive. IntfsOrch transfers the active guard to a replacement request without allowing new work between requests. After the guard is released, the retained request record remains so IntfsOrch can reject stale APPL_DB notifications before they act on a new RIF.

`target_vrf` names the current request’s target VRF: VrfBlue in the BIND examples, or empty for `nomaster`. IntfMgr sets `kernel_pending` before a guarded Linux binding operation and clears it after updating recorded interface state. If processing is interrupted or the request is replaced, IntfMgr uses `kernel_pending` to keep unfinished Linux binding work guarded. Retries of the same `prepare` request retain the request ID.

## Retained work and recovery

IntfsOrch tracks unfinished retirement separately from the ordinary interface queue. If a later SET replaces an interface DEL (a removal request) in that queue, `applied_id` still identifies the applied request whose retirement must complete. IntfsOrch compares `applied_id` with `retired_id` to determine whether retirement is unfinished, including when a newer `prepare` request is canceled. Canceling that newer request therefore does not discard the earlier unfinished retirement.

NeighOrch retains desired neighbor input separately from its cache of hardware neighbors. After successfully removing an old neighbor, NeighOrch requeues a desired SET that it previously processed, unless a newer SET or DEL is already pending. Desired SETs that are still pending remain queued while the guard is active. After guard release and new RIF creation, NeighOrch retries the queued work using the current desired neighbor input.

With retained database state, IntfsOrch reconstructs active requests and retained request records for released guards during startup, before dependency work can use a RIF. IntfMgr checks retained `prepare` requests and unfinished Linux binding work against configuration and recorded interface state. IntfMgr resumes a retained interface removal before processing a replacement request for a different VRF. It uses the old APPL_DB interface entry to recover any required link-local neighbor cleanup. IntfMgr can cancel a retained `prepare` request with no corresponding work to resume and no unfinished interface removal. This recovery scope assumes the retained request state is available.

`INTF_GUARD_ACK` prompts IntfMgr to retry pending work. IntfMgr also retries on timeout and after other events, so pending work can progress even if a notification is lost or input arrives continuously.

## Compatibility and validation

For non-loopback interfaces, the guard applies to interface-level removal from a named VRF, creation of an interface attached to a named VRF, and retained unfinished Linux binding work. Removing an interface already in the default VRF does not use the guard when no unfinished Linux binding work is retained. Loopback processing remains separate. Physical ports, VLAN interfaces, link aggregation group (LAG) interfaces, and sub-ports retain their existing manager and orchestrator readiness requirements. The Linux binding operation remains `master` or `nomaster`.

Validation should compare guarded behavior with the existing behavior and check each completion point separately:

- Exercise default-to-named, named-to-default, and named-to-named changes, independent UNBIND, cancellation, and overlapping BIND across the supported interface types and both address families, including addressless and link-local configurations.
- Verify matching guard confirmation before the Linux binding change, old RIF removal before new RIF creation, and guard-blocked work retained for retry. Cover direct routes, gateway routes, equal-cost multipath (ECMP), fine-grained ECMP, cached object reuse, and steady-state route leaking.
- Exercise a later SET replacing an interface DEL in the queue, stale requests and acknowledgments, command and SAI failures, and supported reload/restart interleavings with retained state. Cover both cases for desired neighbor input: a previously processed SET that NeighOrch requeues after old neighbor removal, and a SET that remains queued while the guard is active.
- Inspect Linux, FRRouting (FRR), APPL_DB, STATE_DB, ASIC_DB, object identity and reference counts, and forwarding separately. Check that IntfMgr updates recorded interface state without waiting for old RIF removal and that unrelated interfaces continue to progress.
