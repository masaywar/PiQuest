# PiQuest

> An AI agent designed for full-stack game development workflows — going beyond simple code editing.

**PiQuest** is an experimental game development AI agent based on [Pi](https://github.com/badlogic/pi-mono). It focuses on closing the gap between code generation and actual engine runtime verification.

---

## 🎯 Core Principles

1. **Game Repository as a Project, Not Just Code**: Treats Unity assets, scenes, prefabs, ScriptableObjects, and settings as interconnected artifacts.
2. **Evidence-Driven Verification**: "Compiles" does not mean "works." Success is measured through runtime state, logs, visual inspection, and profiler data.
3. **Workflow-First, Multi-Agent Later**: Optimizes specialized workflows before splitting into complex multi-agent architecture.
4. **Pragmatic Unity-First Focus**: Validates workflows inside Unity first before generalizing across engines.

---

## 🔄 Verification Ladder

PiQuest validates changes using the most efficient tier needed:

1. Static & Syntax Analysis
2. Unity Compile / Import Validation
3. Scene & Project Load Check
4. Play Mode Execution
5. Runtime State & Gameplay Verification
6. Visual Inspection (Screenshots)
7. Profiling & Performance Metrics

---

## 🚀 Target Workflows

- **Gameplay & Logic**: Implementing features and debugging runtime exceptions.
- **Engine Artifacts**: Managing scenes, prefabs, and ScriptableObjects.
- **Performance & Profiling**: Metric-driven optimizations (CPU, GPU, GC, Memory).
- **Art & Production**: Integrating UI and art assets into scene hierarchy.

---
