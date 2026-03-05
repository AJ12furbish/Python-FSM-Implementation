# Finite State Machine (FSM) — Python Implementation

A lightweight, extensible **Finite State Machine** framework for Python. Designed for game AI, simulation logic, agent behavior, and any system that benefits from clean, structured state management.

---

## Table of Contents

- [Overview](#overview)
- [Architecture](#architecture)
- [Core Components](#core-components)
  - [FSM](#fsm-class)
  - [State](#state-class)
  - [Transition](#transition-class)
- [File Structure](#file-structure)
- [Getting Started](#getting-started)
  - [1. Define Custom States](#1-define-custom-states)
  - [2. Define Custom Transitions](#2-define-custom-transitions)
  - [3. Build the Container](#3-build-the-container)
- [Full Example](#full-example)
- [API Reference](#api-reference)
- [Design Principles](#design-principles)
- [Debug Mode](#debug-mode)

---

## Overview

This FSM framework provides a clean separation between **states**, **transitions**, and the **system being controlled**. Rather than burying state logic in long `if/elif` chains, it lets you model behavior as a graph of discrete states with clearly defined rules for moving between them.

**Key characteristics:**
- States encapsulate their own enter, execute, and exit logic
- Transitions are first-class objects with optional condition functions
- The FSM drives execution — states request transitions, the FSM validates and applies them
- Fully extensible — all three core classes are designed to be subclassed
- Optional debug mode for tracing state changes during development

---

## Architecture

```
┌─────────────────────────────────────────┐
│               Container                 │
│  (your game object, agent, system, etc.)│
│                                         │
│   ┌─────────────────────────────────┐   │
│   │              FSM                │   │
│   │                                 │   │
│   │  ┌──────────┐   ┌──────────┐    │   │
│   │  │  State A │-->│  State B │    │   │
│   │  └──────────┘   └──────────┘    │   │
│   │       │    transition()    │    │   │
│   │       └────────────────────┘    │   │
│   └─────────────────────────────────┘   │
└─────────────────────────────────────────┘
```

The **Container** owns the FSM. The **FSM** manages states and drives the execution loop. **States** hold behavior logic and register transitions. **Transitions** define conditions for moving between states.

---

## Core Components

### `FSM` Class

The central manager. Holds references to all states, tracks the current and previous state, and drives the execution loop each tick.

**Responsibilities:**
- Register states via `addState()`
- Set the initial state via `setState()`
- Accept transition requests via `toTransition()`
- Drive the update loop via `execute()` — call this every frame/tick

---

### `State` Class

The base class for all states. Each state has three lifecycle methods:

|    Method   |            Purpose            | Can call `toTransition`? |
|-------------|-------------------------------|--------------------------|
| `enter()`   | Setup when state is entered   |        ❌ No            |
| `execute()` | Main logic, called every tick |        ✅ Yes           |
| `exit()`    | Cleanup when state is exited  |        ❌ No            |

States also manage their own transitions via `addTransition()` in '__init__' and request state changes via `toTransition()`.

---

### `Transition` Class

Represents a directed edge between two states. Each transition has:

- `toState` — the name of the target state
- `condition` — an optional callable that returns `True` when the transition is valid. If omitted, the transition is always valid.
- `debug` — if `True`, prints a message when the transition fires

---

## File Structure

This framework is intended to be used across three separate files per feature/system, plus the core `FSM.py`:

```
project/
│
├── FSM.py                   # Core framework — State, Transition, FSM classes
│
├── MyFeature/
│   ├── MyFeatureStates.py   # Custom state definitions (extend State)
│   ├── MyFeatureTransitions.py  # Custom transition definitions (extend transition)
│   └── MyFeature.py         # Container class — owns and drives the FSM
```

> **Note:** The state template, transition template, and container template scripts are not included in this repository — only the core `FSM.py` framework is provided here. The templates define the conventions for structuring custom implementations.

---

## Getting Started

### 1. Define Custom States

Create a file for your states and subclass `State`. Override `enter()`, `execute()`, and/or `exit()` as needed.

```python
# MyFeatureStates.py
from FSM import State

class IdleState(State):
    def enter(self):
        super().enter()
        # custom setup logic

    def execute(self):
        super().execute()
        # check conditions, then request a transition if needed
        if self.FSM.container.shouldPatrol:
            self.toTransition('toPatrol')

    def exit(self):
        super().exit()
        # custom cleanup logic


class PatrolState(State):
    def enter(self):
        super().enter()
        self.FSM.container.startPatrolRoute()

    def execute(self):
        super().execute()
        if self.FSM.container.enemySpotted:
            self.toTransition('toChase')

    def exit(self):
        super().exit()
        self.FSM.container.stopPatrolRoute()
```

> **Rule:** Never call `toTransition()` inside `enter()` or `exit()`. Only call it from `execute()`.

---

### 2. Define Custom Transitions

Create a file for your transitions and subclass `transition` if you need custom logic during the transition itself (e.g., playing an animation, logging an event). For simple cases, use `transition` directly.

```python
# MyFeatureTransitions.py
from FSM import transition

class ToPatrol(transition):
    def execute(self):
        super().execute()
        # optional: play walk animation, log event, etc.

class ToChase(transition):
    def execute(self):
        super().execute()
        # optional: trigger alert sound, notify nearby agents, etc.
```

---

### 3. Build the Container

The container owns and wires up the FSM. It creates states, registers them, attaches transitions, and calls `FSM.execute()` each update tick.

```python
# MyFeature.py
from FSM import FSM
from MyFeatureStates import IdleState, PatrolState
from MyFeatureTransitions import ToPatrol, ToChase

class Guard:
    def __init__(self):
        self.shouldPatrol = False
        self.enemySpotted = False

        # Initialize FSM
        self.FSM = FSM(self, debug=False)

        # Create and register states
        idleState   = IdleState(self.FSM)
        patrolState = PatrolState(self.FSM)

        self.FSM.addState('idle',   idleState)
        self.FSM.addState('patrol', patrolState)

        # Register transitions on states
        idleState.addTransition('toPatrol', ToPatrol('patrol'))
        patrolState.addTransition('toChase', ToChase('chase'))

        # Set initial state and enter it
        self.FSM.setState('idle')
        self.FSM.curState.enter()

    def update(self):
        # Call every frame/tick
        self.FSM.execute()
```

---

## Full Example

A minimal working example with two states and one transition:

```python
from FSM import FSM, State, transition

# --- States ---
class Sleeping(State):
    def execute(self):
        print("Zzz...")
        if self.FSM.container.alarm_triggered:
            self.toTransition('toAwake')

class Awake(State):
    def enter(self):
        print("Waking up!")

    def execute(self):
        print("I am awake.")

# --- Container ---
class Person:
    def __init__(self):
        # Add any attributes to access from the container (self.FSM.container)
        self.alarm_triggered = False

        # Create the FSM, passing a reference to self as the container 
        self.FSM = FSM(self, debug=True)

        # Create custom state objects
        sleeping = Sleeping(self.FSM)
        awake    = Awake(self.FSM)

        # Register states with the FSM
        self.FSM.addState('sleeping', sleeping)
        self.FSM.addState('awake',    awake)

        # Register transitions between states
        sleeping.addTransition('toAwake', transition('awake'))

        # Set the initial state
        self.FSM.setState('sleeping')

    def execute(self):
        self.FSM.execute()

# --- Run ---
if '__name__' == '__main__':
  person = Person()
  person.execute()               # → Zzz...
  person.alarm_triggered = True
  person.execute()               # → Waking up! / I am awake.
  person.execute()               # → I am awake.
```

---

## API Reference

### `FSM(container, debug=False)`

|     Method     |             Signature               |                                 Description                                                        |
|----------------|-------------------------------------|----------------------------------------------------------------------------------------------------|
| `addState`     | `(stateName: str, stateObj: State)` | Registers a state under the given name                                                             |
| `setState`     | `(stateName: str)`                  | Sets the active state (does not call `enter()`)                                                    |
| `toTransition` | `(transName: str)`                  | Queues a transition for evaluation on the next `execute()` call                                    |
| `execute`      | `()`                                | Runs one tick: evaluates the queued transition (if any), then runs the current state's `execute()` |

**Properties:**

|  Property   |   Type    |          Description               |
|-------------|-----------|------------------------------------|
| `curState`  | `State`   | The currently active state object  |
| `prevState` | `State`   | The previously active state object |
| `container` | `object`  | Reference to the owning container  |
| `debug`     | `bool`    | Enables debug output               |

---

### `State(FSM)`

|     Method      |                   Signature                   |                             Description                                       |
|-----------------|-----------------------------------------------|-------------------------------------------------------------------------------|
| `enter`         | `()`                                          | Called once when the state is entered. Override for setup logic.              |
| `execute`       | `()`                                          | Called every tick while this is the active state. Override for main behavior. |
| `exit`          | `()`                                          | Called once when the state is exited. Override for cleanup logic.             |
| `addTransition` | `(transName: str, transitionObj: transition)` | Registers a transition on this state                                          |
| `toTransition`  | `(transName: str)`                            | Requests a transition by name (only valid inside `execute()`)                 |

**Properties:**

|   Property    |  Type  |             Description                    |
|---------------|--------|--------------------------------------------|
| `FSM`         | `FSM`  | Reference to the owning FSM                |
| `name`        | `str`  | Name of the state (defaults to class name) |
| `transitions` | `dict` | Dictionary of registered transitions       |

---

### `transition(toState, condition=None, debug=False)`

|   Method  | Signature |                             Description                                      |
|-----------|-----------|------------------------------------------------------------------------------|
| `isValid` | `()`      | Returns `True` if the condition function passes (or if no condition is set)  |
| `execute` | `()`      | Called when the transition fires. Override to add transition-specific logic. |

**Properties:**

|   Property  |       Type         |                     Description                         |
|-------------|--------------------|---------------------------------------------------------|
| `toState`   | `str`              | Name of the target state                                |
| `condition` | `callable \| None` | Function that returns `bool`; `None` means always valid |
| `debug`     | `bool`             | If `True`, prints a message on execution                |

---

## Design Principles

**States are responsible for requesting their own transitions.**
The state's `execute()` method evaluates conditions and calls `toTransition()` when appropriate. The FSM then validates and applies the transition on the same or next tick.

**Transitions are validated before executing.**
The FSM checks `isValid()` before committing to a state change. This allows conditions to be re-evaluated at execution time, not just at the moment the transition is requested.

**The container solely holds data.**
States and transitions access game/system data through `self.FSM.container`. This keeps state logic decoupled from the broader application — states query and react, but don't own the data.

**Extend the classes; do not modify them.**
All three classes (`FSM`, `State`, `transition`) are designed to be subclassed. The base implementations provide safe defaults, so you only override what you need to add custom functionality. 

---

## Debug Mode

Pass `debug=True` to the `FSM` constructor to enable automatic logging of state entries, exits, and transition activations:

```python
self.FSM = FSM(self, debug=True)
```

Individual transitions can also have their own debug flag:

```python
sleeping.addTransition('toAwake', transition('awake', debug=True))
```

This is useful for tracing unexpected state changes during development without adding print statements throughout your state logic.
