# Strategy for Target as only (?) input to transpiler

Conclusions from working on:

- https://github.com/Qiskit/qiskit/pull/12850
- https://github.com/Qiskit/qiskit/issues/9256


## Initial premise

**Goal:** migrate to `Target` as only input for transpiler passes (as opposed to: loose constraints)

**Current State of Preset Pass Manager generator:** A preset pm an be created from a target, backend or loose constraints. I will use loose constraints to refer to the following input arguments that overlap with the target information: `basis_gates`, `coupling_map`, `instruction_durations`, `backend_properties`, `timing_constraints`, `inst_map` (`inst_map` follows a special logic).

```
def generate_preset_pass_manager(
    optimization_level=2,
    backend=None,
    target=None,
    basis_gates=None,
    inst_map=None,
    coupling_map=None,
    instruction_durations=None,
    backend_properties=None,
    timing_constraints=None,
    initial_layout=None,
    layout_method=None,
    routing_method=None,
    translation_method=None,
    scheduling_method=None,
    approximation_degree=1.0,
    seed_transpiler=None,
    unitary_synthesis_method="default",
    unitary_synthesis_plugin_config=None,
    hls_config=None,
    init_method=None,
    optimization_method=None,
    dt=None,
    qubits_initially_zero=True,
    *,
    _skip_target=False,
):
```

Because the arguments overlap, there are **documented** priority rules:

1. if a `target` is provided, it will override both the `backend` and loose constraints provided. Examples: 
    - `generate_preset_pass_manager(target=target, basis_gates=['x']) # basis_gates will be ignored`
    - `generate_preset_pass_manager(target=target, backend=backend) # backend will be ignored`
2. if no `target` is provided:

    - 2.1. a `backend` is provided --> the method to build the `PassManagerConfig` will be different (`PassManagerConfig.from_backend`)
        - 2.1.1. if no loose constraints are provided, a target will be extracted from the `backend` (`backend.target`). Most passes will essentially see a `target`.

        - 2.1.2. if loose constraints are provided -> there will be a "constraint resolution" process where each constraint is extracted from the backend and a parsing function will decide to keep the one with the highest priority in each category. 
        Once the individual constraints are resolved, **a new target is built using** `Target.from_configuration()`. Most passes will essentially see a `target`. For example:
            - `generate_preset_pass_manager(backend=backend, basis_gates=['x']) # in the final target, basis gates will be ['x'], but the rest of the information will come from the backend`

    - 2.2. no `backend` is provided:

        - 2.2.1. if the loose constraints are **sufficient** to build a target, **a new target is built using** `Target.from_configuration()`. All passes will see a `target`.

        - 2.2.2. if the loose constraints are **not sufficient** to build a target, `_skip_target` is set to `True`, and the individual constraints are communicated to the passes as loose constraints. Passes **don't** see a `target`.

## Issue: In a target-centric world, what do we do when loose constraints are not sufficient to build a target?

### Situations when target building is currently skipped:

1. `basis_gates is None` -> needed for target model
2. `coupling_map is None` -> instructions will not be found
3. `instruction_durations is not None` -> we don't have a good way to set priorities with other constraints
4. a custom basis gate is found in `basis_gates` -> we don't know the definition


**The situations are very common in our unit tests (~500 tests), documentation and learning material.** 

## If we want to get rid of loose constraints, I see 2 possible strategies:

### 1. We error. Certain input combinations are not allowed.

- Important breaking change in interface
- Very difficult to justify ("Why can't I just give a coupling map?" "because the target doesn't allow it" "But why?")
- If the goal is to get rid of loose constraints, we will have to:
    1. deprecate certain combinations
    2. deprecate the constraints


Comments: 

- Why make users go through 2 deprecations, one of them very difficult to justify for people not familiar with the target model? It might be better to just deprecate the constraints directly, but:

- I think this breaking change can have a very negative perception among our users. Both internal and external. People are very used to loose constraints and don't want to have to generate additional information they don't care about. I don't see any positive side from the user perspective. More work, less convenience, same output.


### 2. We assume. We fill up the missing information with defaults. We create new rules.

For this strategy, we can maintain the current interface, but need to find a way to build a target internally either:

1. Coming up with defaults --> also a change in behavior
2. Creating a `TranspilationTarget` subclass that allows these impossible combinations and recreates the current behavior. Feasibility to be discussed.

