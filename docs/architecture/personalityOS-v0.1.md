# Personality OS Architecture Summary v0.1

This document summarizes the `v0.1` architecture for Personality OS. The detailed UML source lives in [personality-os-architecture-v0.1.puml](./personality-os-architecture-v0.1.puml).

## Overview

Personality OS is a bootable `x86_64` Linux-based system that starts directly into a personality-driven terminal experience. The first milestone is a research demo that works offline, boots on QEMU and basic PC hardware, and feels like a personality-native OS rather than an app running on top of a conventional desktop.

## Phase 1 Scope

- Build a minimal bootable Linux image with custom boot flow.
- Boot directly into a personality shell/TUI instead of a general-purpose login session.
- Run the personality runtime in early userspace with Linux kept mostly standard.
- Support offline operation with a fixed built-in persona.
- Persist memory and settings locally when writable storage is available.
- Provide recovery and stateless boot modes.

## Core Architecture

The boot chain consists of firmware, bootloader, Linux kernel, and initramfs. Early userspace hands off to a custom init supervisor, which decides whether to enter normal, recovery, or stateless mode.

In normal mode, the init supervisor launches the persona runtime. That runtime includes:

- a personality shell/TUI for user interaction
- a session manager for dialogue lifecycle
- a command router for dispatching user intents
- a safety policy engine that bounds system access
- an offline persona engine driven by built-in rules, templates, and persona identity
- a memory manager and persistence service for session history and long-term memory

## System Behavior

- Normal mode enables persistent memory and settings.
- Stateless mode keeps the system usable even if storage is unavailable or intentionally disabled.
- Recovery mode bypasses the persona runtime and provides a maintenance path.
- The system exposes only a bounded command surface such as help, status, safe file navigation, and reboot or shutdown.
- Unrestricted shell access is not part of the default personality interface.

## Data Model

The first version stores data in a dedicated writable state area, intended for:

- session history
- compact long-term memory
- runtime settings
- diagnostic logs

If the state store is unavailable or corrupted, the system should still boot and fall back to degraded stateless behavior.

## Phase 2 Direction

Phase 2 extends the research path into deeper kernel integration after the Phase 1 boot and runtime experience is stable. Candidate directions include:

- persona-aware kernel event hooks
- a kernel module for deeper system event integration
- an optional patched kernel fork for more ambitious runtime semantics

These are explicitly future-facing and not required for the Phase 1 milestone.

## Acceptance Criteria

- The image boots on `x86_64` QEMU and at least one basic Linux-capable PC.
- The system starts directly into the personality terminal.
- The offline persona can hold a coherent session.
- Memory persists across reboot in normal mode.
- Recovery and stateless modes remain available.
