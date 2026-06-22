# ConfigHash migration state

**Approach A (minimal rotation-fix) — active as of 2026-06-20.**

The ConfigHash API migration (replacing `TransportURLSecret` / notifications URL fields on
sub-CR specs with `ConfigHash` from parent `Status.Hash[InputHashName]`) was **reverted**.
Only the transport-secret rotation guard remains.

## Summary

| Operator | Status | Notes |
|----------|--------|-------|
| **manila** | Reverted | Reference; `TransportURLSecret`/`NotificationsURLSecret` restored on sub-CR specs |
| **cinder** | Reverted | 4 sub-CRs; hash dedup in sub-CR controllers kept |
| **heat** | Reverted | 3 sub-CRs; no redundant transport in verify blocks |
| **octavia** | Reverted | `NotificationsTransportURLSecret` on sub-CR specs; parent propagates both transport fields |
| **watcher** | Reverted | Sub-CRs never had transport spec fields on main; `ConfigHash` removed, no spec propagation |
| **telemetry** | Reverted | CloudKittyAPI + CloudKittyProc only |
| **barbican** | Reverted | 3 sub-CR types |
| **designate** | Reverted | 6 sub-CR types; `TransportURLSecret` propagation in createOrUpdate |
| **ironic** | Reverted | 4 sub-CR types; Inspector/NeutronAgent retain own Status transport for standalone |
| **nova** | Reverted | Sub-CRs never had transport on main; `ConfigHash`/`ObjectHash` propagation removed |

**Out of scope** (monolithic CRs): keystone, neutron, swift, ceilometer, autoscaling, glance, horizon, ovn.

## What we KEEP (Approach A)

1. Parent controller: `ManageTransportSecretFinalizer`, `FinalizeTransportSecretRotation` with `guardReady` using `subcr.RotationGuardReady(subCRStability.Stable(), conditions)`
2. Parent `subCRStability` tracking (`Record` on createOrUpdate when op != None)
3. Sub-CR `IsReady()` fixes (`subcr.ServiceIsReady`, ObservedGeneration timing)
4. Sub-CR controllers: transport/notifications **not** in input hash lists when already in parent `{parent}-config-data`
5. Parent **Status** transport/notifications fields for finalizer tracking
6. lib-common/infra rotation helpers

## What was REVERTED

1. Sub-CR API `ConfigHash` field removed; `TransportURLSecret` / notifications URL fields restored (where they existed on main)
2. Parent controllers: propagate `TransportURLSecret: instance.Status.TransportURLSecret` (and notifications) instead of `configHash := Status.Hash[InputHashName]`
3. Nova: no `ConfigHash` or `util.ObjectHash(transportSecretName)` on sub-CR specs (never had transport fields on main)
4. Watcher: no transport fields on sub-CR specs (same as main)
5. KUTTL assertions for `transportURLSecret` restored where removed

## Pattern checklist (Approach A)

1. Sub-CR API types: `TransportURLSecret` (+ notifications where applicable) — **not** `ConfigHash`
2. Parent controller: `TransportURLSecret: instance.Status.TransportURLSecret` in createOrUpdate
3. Sub-CR controllers: hash parent `{parent}-config-data` (+ scripts); omit direct transport hash if embedded in config-data
4. Octavia: sub-CR specs get `TransportURLSecret` + `NotificationsTransportURLSecret` from parent status
5. Keep parent Status transport fields for rotation finalizers
6. `make generate manifests fmt vet` per operator

## Special cases

### ironic-operator
IronicInspector and IronicNeutronAgent can run standalone and manage their own RabbitMQ transport.
Their `Status.TransportURLSecret` / `Status.NotificationsURLSecret` are retained for rotation within
each controller. Parent propagates `TransportURLSecret` to API/Conductor (and Inspector/NeutronAgent
when parent-managed).

### nova-operator
Nova sub-CRs never had `TransportURLSecret` on spec on main. Rotation is tracked via parent
`Status.TransportURLSecrets` map and finalizer guard only — no spec-level generation bump field.

### watcher-operator
Watcher sub-CRs never had transport fields on spec. Parent input hash + config-data embedding
handles config changes; rotation guard uses parent status only.

## Vet results (Approach A revert)

| Operator | `make vet` |
|----------|-----------|
| manila | pass |
| cinder | pass |
| heat | pass |
| octavia | pass |
| watcher | pass |
| telemetry | pass |
| barbican | pass |
| designate | pass |
| ironic | pass |
| nova | pass |

## Uncommitted changes

All work is local WIP across repos — nothing committed or pushed.
