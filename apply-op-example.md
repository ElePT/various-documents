# Turning Elena's bad practices into hopefully productive feedback

## Example 1

**Task:** iterate over dag1 and append instructions from dag1 to dag2 (initially empty)

**Naive approach:** expose `qargs_interner` and `push_back`. Downside: these shouldn't be exposed.

**Approach respecting internals:** use `apply_operation_back`. Downsides:
- +14 lines of code
- conceptually: stripping down an instruction to give its components to a dag circuit method that builds the instruction for you
- performance-wise: probably neutral, cloning is anyways required

**A probably better approach:** should we add a method to DAGCircuit to append instructions from another dag? Does this exist in the Python DAGCircuit?

<table>
<tr>
<th>Exposing Internals</th>
<th>Respecting Internals</th>
</tr>
<tr>
<td>
  
```rust
for out_node in synth_dag.topological_op_nodes()? {
        let  NodeType::Operation(mut out_packed_instr) = synth_dag.dag()[out_node].clone() else {
            panic!("DAG node must be an instruction")
        };
        let synth_qargs = synth_dag.get_qargs(out_packed_instr.qubits);
        let mapped_qargs: Vec<Qubit> = synth_qargs
            .iter()
            .map(|qarg| out_qargs[qarg.0 as usize])
            .collect();
        out_packed_instr.qubits = out_dag.qargs_interner.insert(&mapped_qargs);
        out_dag.push_back(py, out_packed_instr)?;
    }
    out_dag.add_global_phase(py, &synth_dag.get_global_phase())?;
    Ok(())
```
  
</td>
<td>

```rust
   for out_node in synth_dag.topological_op_nodes()? {
        let NodeType::Operation(out_packed_instr) = &synth_dag.dag()[out_node] else {
            panic!("DAG node must be an instruction")
        };
        let synth_qargs = synth_dag.get_qargs(out_packed_instr.qubits);
        let mapped_qargs: Vec<Qubit> = synth_qargs
            .iter()
            .map(|qarg| out_qargs[qarg.0 as usize])
            .collect();
        let cargs = synth_dag.get_cargs(out_packed_instr.clbits);
        let new_params = out_packed_instr
            .params_view()
            .iter()
            .map(|param| param.clone_ref(py))
            .collect();
        out_dag.apply_operation_back(
            py,
            out_packed_instr.op.clone(),
            &mapped_qargs,
            cargs,
            Some(new_params),
            out_packed_instr.extra_attrs.clone(),
            #[cfg(feature = "cache_pygates")]
            None,
        )?;
    }
    out_dag.add_global_phase(py, &synth_dag.get_global_phase())?;
    Ok(())

```
</td>
</tr>
</table>

## Example 2

**Task:** create an iterable of `PackedInstruction`s and append them to an empty dag using `extend`

**Naive approach:** expose `qargs_interner` to create the `PackedInstruction`s. Downside: these shouldn't be exposed.

**Approach respecting internals:** adding a new `DAGCircuit` method to extend a dag from an iterable of `PackedInstruction` attributes. Downsides:
- longer and uglier
- conceptually: stripping down an instruction to give its components to a dag circuit method that builds the instruction for you. In my opinion, this dilutes the purpose of a container such as the PackedInstruction.
- performance-wise: probably neutral?, cloning is anyways required. Ray's initial tests may indicate a small regression.

**A probably better approach:** ?

<table>
<tr>
<th>Exposing Internals</th>
<th>Respecting Internals</th>
</tr>
<tr>
<td>

```rust
let mut instructions = Vec::new();
    for (gate, params, qubit_ids) in &sequence.gate_sequence.gates {
        let gate_node = match gate {
            None => sequence.decomp_gate.operation.standard_gate(),
            Some(gate) => *gate,
        };
        let mapped_qargs: Vec<Qubit> = qubit_ids.iter().map(|id| out_qargs[*id as usize]).collect();
        let new_params: Option<Box<SmallVec<[Param; 3]>>> = match gate {
            Some(_) => Some(Box::new(params.iter().map(|p| Param::Float(*p)).collect())),
            None => Some(Box::new(sequence.decomp_gate.params.clone())),
        };
        let instruction = PackedInstruction {
            op: PackedOperation::from_standard(gate_node),
            qubits: out_dag.qargs_interner.insert(&mapped_qargs),
            clbits: out_dag.cargs_interner.get_default(),
            params: new_params,
            extra_attrs: ExtraInstructionAttributes::new(None, None, None, None),
            #[cfg(feature = "cache_pygates")]
            py_op: OnceCell::new(),
        };
        instructions.push(instruction);
    }
    out_dag.extend(py, instructions.into_iter())?;
```
  
</td>
<td>

```rust
TODO
```
</td>
</tr>
</table>