# Node Mapping Analysis: Original Netlist to Output AIG

## Overview
This document traces how ABC maintains node mappings and identifies the challenges in preserving mappings through optimization.

---

## Step 1: Initial Mapping (Netlist → AIG via Strash)

### API Flow:
```
1. Abc_Start()
   └── Initializes framework

2. Cmd_CommandExecute("read file.blif")
   └── IoCommandRead()
       └── Io_Read()
           └── Returns: Abc_Ntk_t (ABC_NTK_LOGIC, ABC_FUNC_SOP)
           └── Original nodes stored in pNtk->vObjs

3. Cmd_CommandExecute("strash")
   └── AbcCommandStrash()
       └── Abc_NtkStrash(pNtk)
           ├── Abc_NtkToAig(pNtk)  [Converts SOP→AIG representation]
           ├── Abc_NtkStartFrom()   [Creates new STRASH network]
           └── Abc_NtkStrashPerform(pNtk, pNtkAig)
               └── For each node in topological order:
                   └── pNodeOld->pCopy = Abc_NodeStrash(...)
                       └── Returns new AIG node
```

### Mapping Storage:
**Location:** `src/base/abci/abcStrash.c:424-429`

```c
Vec_PtrForEachEntry( Abc_Obj_t *, vNodes, pNodeOld, i )
{
    pNodeOld->pCopy = Abc_NodeStrash( pNtkNew, pNodeOld, fRecord );
}
```

**Key Point:** Each original node's `pCopy` field temporarily stores its AIG equivalent.

**Mapping Direction:**
- Original node (`pNodeOld`) → AIG node (`pNodeOld->pCopy`)
- One-to-one mapping at this stage

---

## Step 2: Optimization APIs That Break Mappings

### Problem: Mappings Are Temporary

**`pCopy` field is cleared frequently:**
- `Abc_NtkCleanCopy()` - Clears ALL pCopy fields in network
- Called before/after many operations
- Location: `src/base/abc/abcUtil.c:540`

### Optimization Commands That Modify Network Structure:

#### 1. **Balance** (`balance`)
- **API:** `Abc_NtkBalance()`
- **What it does:** Rebalances AIG to reduce depth
- **Effect:** Creates new nodes, may merge/duplicate nodes
- **Mapping impact:** Original mapping lost

#### 2. **Rewrite** (`rewrite`)
- **API:** `Abc_NtkRewrite()`
- **Location:** `src/base/abci/abcRewrite.c:110`
- **What it does:** Rewrites nodes using cuts, replaces with optimized subgraphs
- **Key function:** `Dec_GraphUpdateNetwork()` - **REPLACES nodes**
- **Location:** `src/bool/dec/decAbc.c:240`
- **Mapping impact:** **SEVERELY BREAKS** - nodes are replaced with new ones

```c
// From abcRewrite.c:141
Dec_GraphUpdateNetwork( pNode, pGraph, fUpdateLevel, nGain );
// This REPLACES pNode with a new subgraph!
```

#### 3. **Refactor** (`refactor`)
- **API:** `Abc_NtkRefactor()`
- **What it does:** Refactors logic using cuts
- **Mapping impact:** Creates new nodes, breaks mapping

#### 4. **Resubstitution** (`resub`)
- **API:** `Abc_NtkResubstitute()`
- **Location:** `src/base/abci/abcResub.c`
- **What it does:** Resubstitutes nodes with other logic
- **Mapping impact:** Replaces nodes, breaks mapping

#### 5. **Resynthesis** (`resyn`)
- **API:** `Abc_NtkResynthesize()`
- **Location:** `src/opt/res/resCore.c:213`
- **What it does:** Complete resynthesis of nodes
- **Mapping impact:** **COMPLETELY BREAKS** - entire network restructured

---

## Step 3: Why Mappings Are Lost

### 1. **Structural Changes:**
Optimizations create NEW nodes that don't exist in original:
- `Dec_GraphUpdateNetwork()` replaces a node with a new subgraph
- New nodes have no `pCopy` pointing back to original
- Old nodes may be deleted

### 2. **Node Merging:**
Structural hashing merges equivalent nodes:
- Multiple original nodes → one AIG node
- One-to-many mapping (can't reverse)

### 3. **pCopy Field Reuse:**
`pCopy` is reused for different purposes:
- During strash: stores AIG node
- During mapping: stores mapped node
- During optimization: used temporarily
- **No persistent storage** of original mapping

### 4. **Network Replacement:**
Many optimizations create entirely new networks:
```c
pNtkNew = Abc_NtkStartFrom(pNtk, ...);  // New network!
// Old network may be deleted
```

---

## Step 4: Is Maintaining Mappings Possible?

### **Short Answer: Very Difficult, But Possible with Modifications**

### Challenges:
1. **Nodes are replaced** - Original nodes don't exist after optimization
2. **One-to-many mappings** - Multiple originals map to one optimized node
3. **Many-to-one mappings** - One original may map to multiple optimized nodes
4. **No built-in tracking** - ABC doesn't maintain this by default

### Possible Solutions:

#### **Option 1: Custom Mapping Structure**
Create your own mapping before optimization:
```c
// Before optimization
Vec_Ptr_t * vOriginalNodes = Vec_PtrAlloc(1000);
Vec_Int_t * vMapping = Vec_IntAlloc(1000);

Abc_NtkForEachNode(pNtkOriginal, pNode, i) {
    Vec_PtrPush(vOriginalNodes, pNode);
    // Store node ID
    Vec_IntPush(vMapping, pNode->Id);
}

// After strash (before optimization)
Abc_NtkForEachNode(pNtkOriginal, pNode, i) {
    if (pNode->pCopy) {
        // Store: original[i] -> pNode->pCopy
        // But this breaks after optimization!
    }
}
```

**Problem:** Still breaks after optimization replaces nodes.

#### **Option 2: Track at PO Level Only**
Map only Primary Outputs (POs) - they're preserved:
```c
// POs are usually preserved through optimization
Abc_NtkForEachPo(pNtkOriginal, pPo, i) {
    Abc_Obj_t * pDriverOriginal = Abc_ObjFanin0(pPo);
    // Track: pDriverOriginal -> final optimized PO driver
}
```

**Limitation:** Only works for outputs, not internal nodes.

#### **Option 3: Modify ABC Source Code**
Add persistent mapping fields:
```c
// In Abc_Obj_t structure (abc.h:128)
struct Abc_Obj_t_ {
    // ... existing fields ...
    Abc_Obj_t * pOrigNode;  // NEW: pointer to original node
    Vec_Int_t * vOrigNodes;  // NEW: list of original nodes (for merged nodes)
};
```

**Requires:** Modifying ABC core, recompiling, maintaining fork.

#### **Option 4: Use Name-Based Mapping**
If nodes have names, track by name:
```c
// Store mapping: name -> node
st__table * tNameToNode = st__init_table(strcmp, st__strhash);
Abc_NtkForEachNode(pNtk, pNode, i) {
    if (Abc_ObjName(pNode))
        st__insert(tNameToNode, Abc_ObjName(pNode), pNode);
}
```

**Limitation:** Many nodes don't have names, names may change.

#### **Option 5: Functional Equivalence**
Track by function rather than structure:
- Use truth tables or BDDs to identify equivalent functions
- Map original nodes to optimized nodes with same function
- **Complex and approximate**

---

## Step 5: Recommended Approach

### **For Your Project:**

1. **Capture mapping immediately after strash:**
   ```c
   // Right after strash, before any optimization
   Abc_Ntk_t * pNtkAig = Abc_NtkStrash(pNtkOriginal, 0, 1, 0);
   
   // Store mapping NOW (before optimization)
   Vec_Ptr_t * vOrigToAig = Vec_PtrAlloc(Abc_NtkObjNumMax(pNtkOriginal));
   Abc_NtkForEachNode(pNtkOriginal, pNode, i) {
       if (pNode->pCopy) {
           Vec_PtrWriteEntry(vOrigToAig, i, pNode->pCopy);
       }
   }
   ```

2. **Track only what survives optimization:**
   - Primary Inputs (PIs) - usually preserved
   - Primary Outputs (POs) - usually preserved
   - Latches - usually preserved
   - **Internal nodes: Very difficult to track**

3. **Use functional equivalence for internal nodes:**
   - Compare truth tables/BDDs
   - Map original nodes to optimized nodes with same function
   - Accept that some mappings may be ambiguous

4. **Consider using ABC's equivalence checking:**
   - Use `cec` (combinational equivalence checking) to verify
   - May help identify equivalent nodes

---

## Summary

**Original → AIG Mapping:** ✅ Possible (via `pCopy` after strash)

**Original → Optimized Mapping:** ❌ **Very Difficult**
- Nodes are replaced during optimization
- No built-in tracking mechanism
- `pCopy` field is cleared/reused

**Best Approach:**
1. Capture mapping immediately after strash
2. Track PIs/POs/Latches (usually preserved)
3. Use functional equivalence for internal nodes
4. Accept approximate mappings for heavily optimized nodes

**If you need precise mappings:** Consider modifying ABC source code to add persistent mapping fields.


