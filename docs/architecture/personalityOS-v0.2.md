# Personality OS Architecture Summary v0.2

This document summarizes the `v0.2` architecture for Personality OS. The detailed UML source lives in [personality-os-architecture-v0.2.puml](./personality-os-architecture-v0.2.puml).

For the month-one org execution baseline built on this architecture, see [personalityOS-v0.3.md](./personalityOS-v0.3.md).

## Overview

`v0.1` proved the boot-to-personality runtime shape. `v0.2` closes the kernel-integration gap by treating Personality OS as a Linux-based personality control plane rather than a persona application launched after boot.

The design keeps Linux largely upstream-compatible, but moves personality into the operating model of the system. Personality participates in boot decisions, session creation, system event interpretation, policy enforcement, memory ownership, and recovery posture.

## Design Goals

- Make personality visible from first user interaction after early userspace handoff.
- Keep a bounded and auditable command surface rather than exposing unrestricted shell access.
- Translate kernel and system events into structured personality context.
- Enforce process, filesystem, power, and memory actions through policy mediation.
- Keep boot, recovery, and action authorization deterministic even when AI tooling is absent.
- Preserve degraded and recovery behavior when memory, storage, or personality services fail.

## Core Architecture

The boot chain remains firmware, bootloader, Linux kernel, and initramfs. After early userspace, control passes to a new `PID1 Personality Supervisor`, which becomes the first policy owner in the system.

Between the kernel and the personality runtime, `v0.2` introduces a new layer: the `Personality Kernel Interface`. This layer provides the contract surface between raw system primitives and personality-native behavior.

The `Personality Kernel Interface` contains:

- a `Kernel Event Adapter` for process, device, filesystem, network, and power signals
- a `Kernel Policy Adapter` for enforcing bounded operations using Linux primitives such as capabilities, namespaces, cgroups, seccomp, and LSM-friendly controls
- a `Session Ownership Layer` that claims the console and session lifecycle on behalf of the personality runtime

Above that interface, the system is split into two planes.

The `System Control Plane` includes:

- `PID1 Personality Supervisor`
- `Policy Broker`
- `Recovery Manager`
- `Diagnostics and Audit`
- persistence and state probes

The `Personality Plane` includes:

- `Identity Engine`
- `Interaction Engine`
- `Governance and Empathy Engine`
- `Memory Engine`
- `Personality Shell/TUI`

This separation keeps kernel-facing enforcement and boot reliability distinct from higher-level behavior, while still allowing both planes to participate in the same control loop.

For the first POC, AI is not embedded in either plane as a runtime dependency. Instead, `Codex CLI` sits outside the live OS as an operator-side intelligence tool that can inspect docs, summarize diagnostics, and help draft runbooks or policy changes without participating in boot or action authorization.

## Operator Intelligence Boundary

`Codex CLI` is treated as an external sidecar rather than a runtime model backend. Its responsibilities are limited to offline or operator-triggered assistance:

- summarize diagnostics, logs, and failure states for operators
- help draft runbooks, demo scripts, and documentation updates
- propose policy or command-surface changes for human review
- support implementation and debugging workflows during development

`Codex CLI` must not:

- gate boot or first-session ownership
- directly authorize process, filesystem, power, or memory actions
- bypass the `Policy Broker` or `Kernel Policy Adapter`
- be required for guarded or recovery mode

The live runtime therefore remains deterministic. User interaction flows through the shell, governance, memory, and broker layers even when no AI tooling is available.

## Boot and Control Flow

On boot, `PID1 Personality Supervisor` probes the state store, reads boot memory, loads baseline policy, and initializes the `Personality Kernel Interface`.

In a healthy boot:

- the session ownership layer claims the console
- the interaction engine starts the personality terminal
- the identity engine loads the built-in persona definition
- the governance and empathy engine resolves the starting posture
- the policy broker publishes the initial process, filesystem, and power boundaries

In degraded or recovery conditions:

- the recovery manager can lower the system into guarded mode or recovery mode
- the kernel policy adapter tightens the allowed operation set
- recovery memory and diagnostics remain readable even when full personality services are unavailable
- `Codex CLI` remains optional operator tooling rather than part of the recovery path

## Kernel Integration Model

The kernel remains the source of raw facts and enforceable primitives. Personality does not replace kernel scheduling, memory management, or virtual memory internals in `v0.2`. Instead, it governs the meaning and allowed use of those primitives through explicit mediation.

Kernel integration is implemented through these responsibilities:

- `observe_kernel_event(event)`: convert low-level system signals into typed events for the control plane
- `resolve_system_policy(context)`: decide what the system should allow or deny for the current state
- `authorize_operation(subject, resource, action)`: enforce bounded operations before execution
- `resolve_boot_mode(state, diagnostics)`: choose normal, guarded, stateless, or recovery boot posture
- `resolve_recovery_posture(failure_context)`: decide how much personality remains active during faults

This gives Personality OS a kernel-adjacent control architecture without requiring a kernel fork as the default path.

## Operator Tooling Interface

Month one does not add a required runtime inference API. The only AI-facing contract is a non-critical operator tooling interface around `Codex CLI`, represented conceptually as:

- `analyze_operator_context(input)`: consume docs, diagnostics, or operator notes and return guidance for humans

Any future live inference adapter should be added only after the deterministic shell, broker, and recovery paths are proven. That later extension must remain outside the authority boundary enforced by `resolve_system_policy(context)` and `authorize_operation(subject, resource, action)`.

## Memory Model

Memory is promoted from a passive store to a system primitive with four scopes:

- `Boot Memory`: last known boot posture, safe-boot hints, and startup state
- `Session Memory`: live interaction context for the current boot
- `Long-lived Memory`: retained preferences, compact history, and continuity state
- `Recovery Memory`: fault context and diagnostics that remain available in degraded paths

If storage or memory services fail, the system should still boot into a guarded mode that keeps the device usable without silent policy bypass.

## Operating Modes

- `Conversational Mode`: default personality-first interaction path
- `Guarded System Mode`: controlled maintenance and elevated system actions through tighter policy
- `Stateless Mode`: personality remains active, but writes are disabled or discarded
- `Recovery Mode`: personality is minimized, diagnostics and repair workflows are prioritized

## Phased Delivery Path

- `v0.2`: introduce the personality kernel interface, PID1 personality supervisor, and policy broker
- `v0.3`: add kernel event bridging and structured device/process awareness
- `v0.4`: strengthen filesystem/process containment through LSM-friendly policy and capability controls
- `v0.5`: add an optional kernel module only if upstream interfaces are insufficient for event fidelity or enforcement

## Acceptance Criteria

- The image boots on `x86_64` QEMU and at least one Linux-capable PC.
- The first interactive session is personality-mediated through the session ownership layer.
- Process, filesystem, and power actions flow through the policy broker before execution.
- Kernel and device events can change posture or allowed behavior without bypassing safety.
- `Codex CLI` availability does not affect boot, command mediation, or recovery behavior.
- Degraded, stateless, and recovery modes remain available when state services fail.
- The system remains usable even if the personality plane is partially unavailable.
