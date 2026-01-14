# Project Plan: Node Provenance Tracking in ABC

## Overview

Track node associations through ABC transformations to maintain a mapping from final nodes back to original nodes. When nodes are merged or transformed, the new node should be marked as related to all original nodes that contributed to its creation.

## Target Commands

The following ABC9 commands need to preserve node provenance:

| Command | Location | Function | AIG Conversion Required |
|---------|----------|----------|------------------------|
| `&get -n` | `src/base/abci/abc.c:1236` | `Abc_CommandAbc9Get` | YES (during conversion) |
| `&st` | `src/base/abci/abc.c:1270` | `Abc_CommandAbc9Strash` | NO |
| `&sweep` | `src/base/abci/abc.c:1334` | `Abc_CommandAbc9Sweep` | NO |
| `&dch` | `src/base/abci/abc.c:1377` | `Abc_CommandAbc9Dch` | YES |
| `&nf` | `src/base/abci/abc.c:1351` | `Abc_CommandAbc9Nf` | NO |
| `&syn2` | `src/base/abci/abc.c:1308` | `Abc_CommandAbc9Syn2` | NO |
| `&if -g -K 6` | `src/base/abci/abc.c:1342` | `Abc_CommandAbc9If` | NO |
| `&put` | `src/base/abci/abc.c:1237` | `Abc_CommandAbc9Put` | YES (during conversion) |

### AIG Conversion Details

**Commands requiring AIG conversion:**
- **`&dch`**: Converts GIA → AIG → GIA (via `Gia_ManToAig` → `Dar_ManChoiceNew` → `Gia_ManFromAigChoices`)
- **`&get -n`**: Converts `Abc_Ntk_t` → AIG → GIA (uses `Abc_NtkToDar` → `Gia_ManFromAig`)
- **`&put`**: Converts GIA → AIG → `Abc_Ntk_t` (uses `Gia_ManToAig` → `Abc_NtkFromAigPhase`)

**Commands working directly on GIA:**
- **`&st`**: Uses `Gia_ManRehash()` — structural hashing on GIA
- **`&sweep`**: Uses `Gia_ManFraigSweepSimple()` — SAT sweeping on GIA
- **`&nf`**: Uses `Nf_ManPerformMapping()` — LUT mapping on GIA
- **`&syn2`**: Uses `Gia_ManAigSynch2()` — synthesis on GIA
- **`&if -g -K 6`**: Uses `If_ManPerformMapping()` — LUT mapping on GIA (uses internal If format, not AIG)

**Summary:**
- Only `&dch` requires AIG conversion for its operation
- `&get` and `&put` use AIG as an intermediate during format conversion, not for the command logic
- All other commands operate directly on GIA

## Data Structure Strategy

### Proposed Mapping Structure

Add a `Vec_Vec_t` field to `Gia_Man_t` to track the mapping between current `Gia_Obj_t` nodes (within the `Gia_Man_t`) and the original node list (with IDs and names) from the initial `Abc_Frame_t`.

**Key Points:**
- When a file is read, capture original node names and IDs
- Maintain a mapping: `Gia_Obj_t` → original node IDs/names
- Add a new command (modify `Abc_CommandPrintLevel` as starting point) to print `Gia_Man_t` node mappings back to original nodes
- This command is only valid if the current `Gia_Man_t` is not closed

**Implementation:**
- Append `Vec_Vec_t` to `Gia_Man_t` structure
- Each inner vector contains the original node IDs that contributed to the current node
- When nodes merge, union the original node sets

**Note:** If Pyyosys requires BLIF input (see BLIF Writing Constraints section), we will also need to add `Vec_Vec_t` to `Abc_Ntk_t` structure to preserve node mappings during `&put` conversion from `Gia_Man_t` to `Abc_Ntk_t`.

## Implementation Approach

### Phase 1: Format Conversion Functions (Priority: High)

Start with `&get` and `&put` functions to establish the foundation for node tracking during format conversions. This provides flexibility in execution order for subsequent commands.

### Phase 2: Individual Command Implementation

Go through each command one by one, tweaking the algorithm for each function to maintain a relevant mapping back to the original step. The challenge is adapting each transformation to preserve provenance information.

### Main Bottlenecks

The primary bottlenecks are:
1. **`&get`**: Format conversion from `Abc_Ntk_t` → GIA
2. **`&put`**: Format conversion from GIA → `Abc_Ntk_t`
3. **`&dch`**: AIG conversion requirement for equivalence computation

## Alternative Approach: Direct GIA Reading

Currently investigating **Pyyosys** to see if we can directly run `Abc_CommandAbc9ReadBlif` (`&read_blif` instead of `read_blif`). This would allow reading directly into `Gia_Man_t` format, potentially eliminating the need for `&get` and `&put` functions at the start and end of the pipeline.

**Rationale:**
- `&get` and `&put` convert between `Gia_Man_t` and `Abc_Ntk_t`
- Since we never use the traditional `Abc_Ntk_t` in the ABC9 workflow, direct GIA reading may be more efficient
- This approach depends on whether `&dch` is needed in the pipeline

### BLIF Writing Constraints

**Current State:**
- `Abc_CommandAbc9Write` (`&write`) does **not** support BLIF output
- `&write` supports: AIGER (default), Verilog (`-p` or `-q`), MiniAIG (`-m`), MiniLUT (`-l`)
- BLIF writing in ABC uses `Abc_Ntk_t` (traditional ABC format), not `Gia_Man_t` (ABC9 format)
- To write BLIF from ABC9: use `&put` to convert `Gia_Man_t` → `Abc_Ntk_t`, then use `write_blif`

**Implications for Pyyosys Integration:**
- If Pyyosys requires BLIF format input, we will need to:
  1. Add `Vec_Vec_t` provenance tracking to `Abc_Ntk_t` structure (in addition to `Gia_Man_t`)
  2. Support `&put` functionality to preserve node mappings during conversion
  3. Ensure `write_blif` can output with preserved node information

**Investigation Items (Day 1):**
- Pyyosys API: Can it accept AIGER/Verilog instead of BLIF? Can API be modified to support direct `Gia_Man_t` operations?
- `&write_blif` feasibility: Investigate if adding `&write_blif` command is possible
  - **Challenge**: BLIF requires node names; may need to generate dummy names if names are not preserved
  - **Note**: This may not be feasible if node names are essential for BLIF format

**Decision Dependency:**
- If Pyyosys doesn't support API changes or `&write_blif` is infeasible, then `&dch` functionality will likely be needed
- In this case, full implementation path: `&get` → transformations → `&put` → `write_blif`

## Implementation Timeline (3 Weeks)

### Week 1: Analysis and Decision Making

**Day 1: Pyyosys and BLIF Investigation**
- Investigate Pyyosys API: Can it accept AIGER/Verilog instead of BLIF? Can API be modified to support direct `Gia_Man_t` operations?
- Investigate `&write_blif` feasibility: Is adding `&write_blif` command possible? (Challenge: BLIF requires node names)
- Determine if `&dch` is needed in the workflow

**Step 1: Profile Without DCH**
- Run precursor workflow without `&dch` command
- Profile and collect statistics
- **Note**: Need guidance on how to profile this with more than just a single example
- **Decision Point**: Determine if `&dch` is necessary for the target use cases

### Week 2: Core Infrastructure

**Step 2A: If DCH Not Needed**
- Investigate and implement Pyyosys API modification to allow direct reading into `Gia_Man_t`
- This eliminates `&get` and `&put` from the critical path
- Focus on `&read_blif` integration

**Step 2B: If DCH Needed**
- Implement `&get` and `&put` functions first (format conversion tracking)
- Then implement `&dch` (AIG conversion tracking)
- **This is where the bulk of the time will be spent**

### Week 3: Remaining Commands

**Step 3: Individual Command Implementation**
- Implement node tracking for each remaining command:
  - `&st` (structural hashing)
  - `&sweep` (SAT sweeping)
  - `&nf` (LUT mapping)
  - `&syn2` (synthesis)
  - `&if -g -K 6` (LUT mapping)
- Each command requires algorithm-specific tweaks to maintain provenance

## Deliverables

Node mapping from final nodes back to original nodes that tracks node associations through ABC transformations.
