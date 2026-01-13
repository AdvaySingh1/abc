# Function Trace: Reading Network to AIG Conversion (Strash)

## Overview
This document traces the function call path from reading a network file (BLIF/AIGER) to converting it to a structurally hashed AIG (strash).

---

## Call Path for BLIF Files

### 1. Command Entry Point
```
Cmd_CommandExecute()
└── src/base/cmd/cmdApi.c
    └── Executes user command "read file.blif"
```

### 2. Read Command Handler
```
IoCommandRead()
└── src/base/io/io.c:210
    └── Parses command arguments
    └── Calls Io_Read() at line 311
```

### 3. Generic Read Function
```
Io_Read()
└── src/base/io/ioUtil.c:84
    └── Determines file type (BLIF, AIGER, etc.)
    └── For BLIF files:
        └── Calls Io_ReadBlifMv() at line 138
```

### 4. BLIF Parser
```
Io_ReadBlifMv()
└── src/base/io/ioReadBlifMv.c:142
    └── Creates Io_MvMan_t parser structure
    └── Calls Io_MvParse() at line 183
    └── Returns Abc_Ntk_t* with type ABC_NTK_LOGIC
    └── Network nodes have SOP (Sum-of-Products) representation
```

**Network State After Reading:**
- Type: `ABC_NTK_LOGIC`
- Function: `ABC_FUNC_SOP` (or `ABC_FUNC_BLIFMV`)
- Nodes contain SOP expressions

---

## Call Path for AIGER Files

### 1. Command Entry Point
```
Cmd_CommandExecute()
└── src/base/cmd/cmdApi.c
    └── Executes user command "read file.aig"
```

### 2. AIGER Read Command Handler
```
IoCommandReadAiger()
└── src/base/io/io.c:350
    └── Calls Io_Read() at line 377
```

### 3. Generic Read Function
```
Io_Read()
└── src/base/io/ioUtil.c:84
    └── Detects IO_FILE_AIGER type
    └── Calls Io_ReadAiger() at line 125
```

### 4. AIGER Parser
```
Io_ReadAiger()
└── src/base/io/ioReadAiger.c:234
    └── Reads binary AIGER format
    └── Creates Abc_Ntk_t* directly as ABC_NTK_STRASH
    └── Returns network already in AIG format
```

**Network State After Reading:**
- Type: `ABC_NTK_STRASH` (already structurally hashed!)
- Function: `ABC_FUNC_AIG`
- Nodes are already AIG nodes

---

## Strash Conversion (for BLIF and other logic networks)

### 1. Strash Command Entry
```
Cmd_CommandExecute("strash")
└── Calls AbcCommandStrash()
    └── src/base/abci/abc.c:3966
        └── Calls Abc_NtkStrash() at line 3966
```

### 2. Main Strash Function
```
Abc_NtkStrash()
└── src/base/abci/abcStrash.c:265
    ├── Line 269: Checks if already strashed (returns early if ABC_NTK_STRASH)
    ├── Line 274: Converts nodes to AIG form
    │   └── Abc_NtkToAig(pNtk)
    │       └── src/base/abc/abcFunc.c:1333
    │           ├── Line 1338: If already has AIG, return early
    │           ├── Line 1340-1343: If mapped, convert mapping → SOP → AIG
    │           ├── Line 1345-1349: If BDD, convert BDD → SOP → AIG
    │           └── Line 1351-1352: If SOP, convert SOP → AIG
    │               └── Abc_NtkSopToAig(pNtk)
    │                   └── Converts each node's SOP to AIG representation
    │                   └── Uses Hop_Man_t (AIG manager) to build AIGs
    ├── Line 281: Creates new STRASH network
    │   └── Abc_NtkStartFrom(pNtk, ABC_NTK_STRASH, ABC_FUNC_AIG)
    ├── Line 282: Performs structural hashing
    │   └── Abc_NtkStrashPerform(pNtk, pNtkAig, fAllNodes, fRecord)
    ├── Line 283: Finalizes network
    │   └── Abc_NtkFinalize(pNtk, pNtkAig)
    └── Line 304: Returns structurally hashed AIG network
```

### 3. Strash Perform (Core Conversion)
```
Abc_NtkStrashPerform()
└── src/base/abci/abcStrash.c:413
    ├── Line 421: Gets nodes in topological order
    │   └── Abc_NtkDfsIter(pNtkOld, fAllNodes)
    └── Line 424-430: For each node:
        └── Abc_NodeStrash(pNtkNew, pNodeOld, fRecord)
            └── src/base/abci/abcStrash.c:468
                ├── Line 477: Gets Hop_Man_t (AIG manager)
                ├── Line 478: Gets root AIG node (pNodeOld->pData)
                ├── Line 500: Sets fanin variables
                ├── Line 502: Recursively strashes AIG
                │   └── Abc_NodeStrash_rec()
                │       └── src/base/abci/abcStrash.c:445
                │           └── Recursively processes fanins
                │           └── Line 452: Creates AND node with structural hashing
                │               └── Abc_AigAnd(pMan, fanin0, fanin1)
                └── Line 505: Returns strashed node
```

### 4. Structural Hashing (Key Step)
```
Abc_AigAnd()
└── src/base/abc/abcAig.c
    └── Creates AND node OR reuses existing structurally equivalent node
    └── Uses hash table to detect structural equivalence
    └── This is the "structural hashing" that eliminates duplicates
```

---

## Key Data Structures

### Before Strash (Logic Network)
```c
Abc_Ntk_t {
    nType = ABC_NTK_LOGIC
    nFunc = ABC_FUNC_SOP
    pManFunc = Hop_Man_t*  // AIG manager for node functions
    // Nodes contain SOP expressions or BDDs
}
```

### After Strash (Structurally Hashed AIG)
```c
Abc_Ntk_t {
    nType = ABC_NTK_STRASH
    nFunc = ABC_FUNC_AIG
    pManFunc = Abc_Aig_t*  // Structural hashing manager
    // Nodes are AIG nodes (2-input AND gates)
    // Structurally equivalent nodes are merged
}
```

---

## Summary Flow Diagram

```
User Command: "read file.blif"
    │
    ├─> Cmd_CommandExecute()
    │       │
    │       └─> IoCommandRead()
    │               │
    │               └─> Io_Read()
    │                       │
    │                       └─> Io_ReadBlifMv()
    │                               │
    │                               └─> Returns: Abc_Ntk_t (ABC_NTK_LOGIC, ABC_FUNC_SOP)
    │
User Command: "strash"
    │
    ├─> Cmd_CommandExecute()
    │       │
    │       └─> AbcCommandStrash()
    │               │
    │               └─> Abc_NtkStrash()
    │                       │
    │                       ├─> Abc_NtkToAig()  [Convert SOP to AIG]
    │                       │
    │                       ├─> Abc_NtkStartFrom()  [Create STRASH network]
    │                       │
    │                       └─> Abc_NtkStrashPerform()
    │                               │
    │                               ├─> Abc_NtkDfsIter()  [Topological order]
    │                               │
    │                               └─> Abc_NodeStrash()  [For each node]
    │                                       │
    │                                       └─> Abc_NodeStrash_rec()
    │                                               │
    │                                               └─> Abc_AigAnd()  [Structural hashing]
    │
Result: Abc_Ntk_t (ABC_NTK_STRASH, ABC_FUNC_AIG)
```

---

## Key Functions Reference

| Function | File | Purpose |
|----------|------|---------|
| `IoCommandRead()` | `src/base/io/io.c:210` | Command handler for "read" |
| `Io_Read()` | `src/base/io/ioUtil.c:84` | Generic file reader dispatcher |
| `Io_ReadBlifMv()` | `src/base/io/ioReadBlifMv.c:142` | BLIF parser |
| `Io_ReadAiger()` | `src/base/io/ioReadAiger.c:234` | AIGER parser |
| `Abc_NtkStrash()` | `src/base/abci/abcStrash.c:265` | Main strash conversion function |
| `Abc_NtkToAig()` | `src/base/abc/abcFunc.c:1333` | Converts node functions (SOP/BDD) to AIG |
| `Abc_NtkSopToAig()` | `src/base/abc/abcFunc.c` | Converts SOP to AIG (called by Abc_NtkToAig) |
| `Abc_NtkStrashPerform()` | `src/base/abci/abcStrash.c:413` | Performs structural hashing |
| `Abc_NodeStrash()` | `src/base/abci/abcStrash.c:468` | Strash a single node |
| `Abc_AigAnd()` | `src/base/abc/abcAig.c` | Creates AND node with structural hashing |

---

## Notes

1. **AIGER files** are already in AIG format, so they return `ABC_NTK_STRASH` directly - no conversion needed.

2. **BLIF files** return `ABC_NTK_LOGIC` with SOP representation, requiring conversion via `strash`.

3. **Structural hashing** (`Abc_AigAnd`) ensures that structurally equivalent nodes (same fanins) are merged into a single node.

4. **Topological order** (`Abc_NtkDfsIter`) ensures nodes are processed after their fanins are converted.

5. The conversion happens in two stages:
   - **Stage 1**: Convert node functions (SOP/BDD) to AIG representation (`Abc_NtkToAig`)
   - **Stage 2**: Apply structural hashing to merge equivalent nodes (`Abc_NtkStrashPerform`)

