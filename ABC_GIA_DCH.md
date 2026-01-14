# Why DCH Requires AIG Format Instead of GIA

## Overview

The `&dch -e` command (equivalence computation mode) requires converting GIA format to AIG format before processing. This document explains the fundamental data structure differences and API dependencies that necessitate this conversion.

## Table of Contents

1. [Data Structure Comparison](#data-structure-comparison)
2. [DCH Data Structures](#dch-data-structures)
3. [Critical AIG-Specific APIs](#critical-aig-specific-apis)
4. [Why GIA Cannot Be Used Directly](#why-gia-cannot-be-used-directly)
5. [Conversion Process](#conversion-process)
6. [Code Flow Analysis](#code-flow-analysis)

---

## Data Structure Comparison

### AIG Format: `Aig_Man_t` and `Aig_Obj_t`

**Location:** `src/aig/aig/aig.h`

#### AIG Manager Structure

```c
// src/aig/aig/aig.h:94-170
struct Aig_Man_t_
{
    char *           pName;          // the design name
    char *           pSpec;          // the input file name
    // AIG nodes
    Vec_Ptr_t *      vCis;           // the array of PIs
    Vec_Ptr_t *      vCos;           // the array of POs
    Vec_Ptr_t *      vObjs;          // the array of all nodes (optional)
    Vec_Ptr_t *      vBufs;          // the array of buffers
    Aig_Obj_t *      pConst1;        // the constant 1 node
    
    // ... counters ...
    
    // structural hash table
    Aig_Obj_t **     pTable;         // structural hash table
    int              nTableSize;     // structural hash table size
    
    // representation of fanouts
    int *            pFanData;       // the database to store fanout information
    int              nFansAlloc;     // the size of fanout representation
    
    // representatives - CRITICAL for DCH
    Aig_Obj_t **     pEquivs;        // linked list of equivalent nodes (when choices are used)
    Aig_Obj_t **     pReprs;         // representatives of each node
    int              nReprsAlloc;    // the number of allocated representatives
    
    // ... other fields ...
};
```

**Key Features:**
- **`pReprs`**: Array of `Aig_Obj_t*` pointers storing equivalence representatives
- **`pFanData`**: Specialized fanout database structure
- **`pEquivs`**: Linked list structure for equivalent nodes
- Objects stored as **pointers** (`Aig_Obj_t*`)

#### AIG Object Structure

```c
// src/aig/aig/aig.h: (Aig_Obj_t definition)
// AIG objects are allocated individually and accessed via pointers
// Each object has:
//   - Direct pointer access to fanins/fanouts
//   - pReprs array index for equivalence classes
//   - Fanout information stored in pFanData
```

---

### GIA Format: `Gia_Man_t` and `Gia_Obj_t`

**Location:** `src/aig/gia/gia.h`

#### GIA Manager Structure

```c
// src/aig/gia/gia.h:96-133
struct Gia_Man_t_
{
    char *         pName;         // name of the AIG
    char *         pSpec;         // name of the input file
    int            nRegs;         // number of registers
    int            nRegsAlloc;    // number of allocated registers
    int            nObjs;         // number of objects
    int            nObjsAlloc;    // number of allocated objects
    Gia_Obj_t *    pObjs;         // the array of objects (CONTIGUOUS ARRAY)
    
    Vec_Int_t *    vCis;          // the vector of CIs (PIs + LOs)
    Vec_Int_t *    vCos;          // the vector of COs (POs + LIs)
    Vec_Int_t      vHash;         // hash links
    Vec_Int_t      vHTable;       // hash table
    
    // representatives - DIFFERENT STRUCTURE
    int *          pReprsOld;     // representatives (for CIs and ANDs)
    Gia_Rpr_t *    pReprs;        // representatives (for CIs and ANDs) - BITFIELD STRUCTURE
    int *          pNexts;        // next nodes in the equivalence classes
    int *          pSibls;        // next nodes in the choice nodes
    
    int *          pFanData;      // the database to store fanout information
    int            nFansAlloc;    // the size of fanout representation
    // ... other fields ...
};
```

#### GIA Representative Structure

```c
// src/aig/gia/gia.h:57-65
struct Gia_Rpr_t_
{
    unsigned       iRepr   : 28;  // representative node (INDEX, not pointer)
    unsigned       fProved :  1;  // marks the proved equivalence
    unsigned       fFailed :  1;  // marks the failed equivalence
    unsigned       fColorA :  1;  // marks cone of A
    unsigned       fColorB :  1;  // marks cone of B
};
```

#### GIA Object Structure

```c
// src/aig/gia/gia.h:76-90
struct Gia_Obj_t_
{
    unsigned       iDiff0 :  29;  // the diff of the first fanin (OFFSET, not pointer)
    unsigned       fCompl0:   1;  // the complemented attribute
    unsigned       fMark0 :   1;  // first user-controlled mark
    unsigned       fTerm  :   1;  // terminal node (CI/CO)

    unsigned       iDiff1 :  29;  // the diff of the second fanin (OFFSET, not pointer)
    unsigned       fCompl1:   1;  // the complemented attribute
    unsigned       fMark1 :   1;  // second user-controlled mark
    unsigned       fPhase :   1;  // value under 000 pattern

    unsigned       Value;         // application-specific value
};
```

**Key Differences:**
- **GIA uses indices/offsets** (`iDiff0`, `iDiff1`) instead of pointers
- **GIA uses bitfield structures** (`Gia_Rpr_t`) instead of pointer arrays
- **GIA objects are in a contiguous array** (`pObjs[]`) vs. individual allocations
- **GIA representatives are indices** (`iRepr : 28`) vs. pointers (`Aig_Obj_t*`)

---

## DCH Data Structures

**Location:** `src/proof/dch/dchInt.h`

### DCH Manager Structure

```c
// src/proof/dch/dchInt.h:51-96
struct Dch_Man_t_
{
    // parameters
    Dch_Pars_t *     pPars;          // choicing parameters
    
    // AIGs used in the package - REQUIRES Aig_Man_t
    Aig_Man_t *      pAigTotal;      // intermediate AIG
    Aig_Man_t *      pAigFraig;      // final AIG
    
    // equivalence classes - STORES Aig_Obj_t* POINTERS
    Dch_Cla_t *      ppClasses;      // equivalence classes of nodes
    Aig_Obj_t **     pReprsProved;   // equivalences proved (ARRAY OF POINTERS)
    
    // SAT solving
    sat_solver *     pSat;           // recyclable SAT solver
    int              nSatVars;       // the counter of SAT variables
    int *            pSatVars;       // mapping of each node into its SAT var
    Vec_Ptr_t *      vUsedNodes;     // nodes whose SAT vars are assigned
    Vec_Ptr_t *      vFanins;        // fanins of the CNF node (Vec_Ptr_t = pointer vector)
    Vec_Ptr_t *      vSimRoots;      // the roots of cand const 1 nodes to simulate
    Vec_Ptr_t *      vSimClasses;    // the roots of cand equiv classes to simulate
    
    // ... statistics ...
};
```

### DCH Class Structure

```c
// src/proof/dch/dchClass.c:37-57
struct Dch_Cla_t_
{
    // class information
    Aig_Man_t *      pAig;             // original AIG manager (NOT Gia_Man_t)
    Aig_Obj_t ***    pId2Class;        // non-const classes by ID of repr node (POINTER ARRAY)
    int *            pClassSizes;      // sizes of each equivalence class
    
    // statistics
    int              nClasses;         // the total number of non-const classes
    int              nCands1;          // the total number of const candidates
    int              nLits;            // the number of literals in all classes
    
    // memory
    Aig_Obj_t **     pMemClasses;      // memory allocated for equivalence classes (POINTER ARRAY)
    Aig_Obj_t **     pMemClassesFree;  // memory allocated for equivalence classes to be used
    
    // temporary data
    Vec_Ptr_t *      vClassOld;        // old equivalence class after splitting (POINTER VECTOR)
    Vec_Ptr_t *      vClassNew;        // new equivalence class(es) after splitting (POINTER VECTOR)
    
    // procedures used for class refinement
    void *           pManData;
    unsigned (*pFuncNodeHash) (void *,Aig_Obj_t *);              // Aig_Obj_t* parameter
    int (*pFuncNodeIsConst)   (void *,Aig_Obj_t *);              // Aig_Obj_t* parameter
    int (*pFuncNodesAreEqual) (void *,Aig_Obj_t *, Aig_Obj_t *); // Aig_Obj_t* parameters
};
```

**Critical Dependency:**
- **All structures use `Aig_Obj_t*` pointers**, not indices
- **`Vec_Ptr_t`** (vector of pointers) is used throughout
- **Function signatures require `Aig_Obj_t*`**, not `Gia_Obj_t*`

---

## Critical AIG-Specific APIs

### 1. Fanout Information: `Aig_ManFanoutStart()`

**Location:** `src/aig/aig/aigFanout.c:56`

```c
// src/proof/dch/dchMan.c:53
Aig_ManFanoutStart( p->pAigTotal );
```

**Why it's needed:**
- DCH needs to traverse fanouts of nodes during equivalence checking
- AIG has a specialized `pFanData` structure that must be initialized
- GIA has `pFanData` but uses a different format and initialization

**AIG Implementation:**
```c
// src/aig/aig/aigFanout.c:56
void Aig_ManFanoutStart( Aig_Man_t * p )
{
    Aig_Obj_t * pObj;
    int i;
    assert( Aig_ManBufNum(p) == 0 );
    // allocate fanout datastructure
    // ... builds pFanData using Aig_Obj_t* pointers ...
}
```

**Problem with GIA:**
- GIA's `pFanData` uses object indices, not pointers
- DCH code expects pointer-based fanout traversal
- Conversion would require rewriting all fanout traversal code

---

### 2. Equivalence Representation: `Aig_ManReprStart()`

**Location:** `src/aig/aig/aigRepr.c:45`

```c
// src/proof/dch/dchClass.c:148
Aig_ManReprStart( pAig, Aig_ManObjNumMax(pAig) );
```

**Why it's needed:**
- DCH stores equivalence classes in `Aig_Man_t->pReprs` array
- This array stores `Aig_Obj_t*` pointers to representative nodes
- Used throughout DCH for equivalence class management

**AIG Implementation:**
```c
// src/aig/aig/aigRepr.c:45
void Aig_ManReprStart( Aig_Man_t * p, int nIdMax )
{
    assert( Aig_ManBufNum(p) == 0 );
    assert( p->pReprs == NULL );
    p->nReprsAlloc = nIdMax;
    p->pReprs = ABC_ALLOC( Aig_Obj_t *, p->nReprsAlloc );  // ARRAY OF POINTERS
    // ...
}
```

**GIA Difference:**
```c
// GIA uses:
Gia_Rpr_t * pReprs;  // Array of bitfield structures, not pointers
// Where Gia_Rpr_t contains:
//   unsigned iRepr : 28;  // INDEX, not pointer
```

**Problem:**
- DCH code accesses `pReprs[iObj]` expecting an `Aig_Obj_t*` pointer
- GIA's `pReprs[iObj]` returns a `Gia_Rpr_t` structure with an index
- All DCH code would need to be rewritten to use indices instead of pointers

---

### 3. SAT Variable Mapping: `Aig_ManObjNumMax()`

**Location:** `src/proof/dch/dchMan.c:56`

```c
// src/proof/dch/dchMan.c:56
p->pSatVars = ABC_CALLOC( int, Aig_ManObjNumMax(p->pAigTotal) );
```

**Why it's needed:**
- DCH maps each AIG object to a SAT variable
- Uses object ID as array index
- AIG and GIA have different object numbering schemes

**AIG:**
- Objects have unique IDs that may have gaps (deleted objects)
- `Aig_ManObjNumMax()` returns the maximum allocated ID

**GIA:**
- Objects are in a contiguous array
- IDs are sequential indices (0, 1, 2, ...)
- Different API: `Gia_ManObjNum()` vs `Aig_ManObjNumMax()`

---

### 4. Object Pointer Arrays: `Aig_Obj_t**`

**Location:** Throughout DCH code

```c
// src/proof/dch/dchClass.c:143
p->pId2Class = ABC_CALLOC( Aig_Obj_t **, Aig_ManObjNumMax(pAig) );

// src/proof/dch/dchMan.c:62
p->pReprsProved = ABC_CALLOC( Aig_Obj_t *, Aig_ManObjNumMax(p->pAigTotal) );
```

**Why it's critical:**
- DCH stores arrays of object **pointers**, not indices
- All equivalence class operations use pointer arithmetic
- Function signatures require `Aig_Obj_t*` parameters

**Example from DCH:**
```c
// src/proof/dch/dchSat.c (hypothetical usage)
int Dch_NodesAreEquiv( Dch_Man_t * p, Aig_Obj_t * pObj1, Aig_Obj_t * pObj2 )
{
    // Direct pointer comparison and manipulation
    if ( p->pReprsProved[pObj1->Id] == pObj2 )
        return 1;
    // ...
}
```

**GIA Problem:**
- GIA objects are accessed via `Gia_ManObj(pGia, iObj)` function
- No direct pointer access - must use index-based lookup
- All DCH code would need index-to-pointer conversion everywhere

---

## Why GIA Cannot Be Used Directly

### 1. Pointer vs. Index Architecture

**AIG Architecture:**
```c
Aig_Obj_t * pObj = Aig_ManObj(pAig, i);  // Returns pointer
pObj->pNext = otherObj;                   // Direct pointer assignment
pReprs[i] = pObj;                         // Store pointer in array
```

**GIA Architecture:**
```c
Gia_Obj_t * pObj = Gia_ManObj(pGia, i);  // Returns pointer to array element
// But fanins are stored as OFFSETS:
int iFanin0 = Gia_ObjFaninId0(pObj);      // Returns index, not pointer
Gia_Obj_t * pFanin0 = Gia_ManObj(pGia, iFanin0);  // Must lookup
// Representatives are indices:
pReprs[i].iRepr = j;                      // Store index, not pointer
```

**Impact on DCH:**
- DCH algorithms assume direct pointer access
- All fanout/fanin traversal uses pointer dereferencing
- Equivalence class management relies on pointer arrays

---

### 2. Function Signature Mismatch

**DCH Function Signatures:**
```c
// src/proof/dch/dchInt.h:150
extern int Dch_NodesAreEquiv( Dch_Man_t * p, Aig_Obj_t * pObj1, Aig_Obj_t * pObj2 );

// src/proof/dch/dchInt.h:152
extern Dch_Cla_t * Dch_CreateCandEquivClasses( Aig_Man_t * pAig, int nWords, int fVerbose );

// src/proof/dch/dchInt.h:164
unsigned (*pFuncNodeHash) (void *,Aig_Obj_t *);
int (*pFuncNodeIsConst)   (void *,Aig_Obj_t *);
int (*pFuncNodesAreEqual) (void *,Aig_Obj_t *, Aig_Obj_t *);
```

**All require `Aig_Obj_t*`, not `Gia_Obj_t*`**

---

### 3. Data Structure Incompatibility

**DCH Class Structure:**
```c
// src/proof/dch/dchClass.c:40
Aig_Obj_t *** pId2Class;        // Triple pointer array
Aig_Obj_t **  pMemClasses;      // Double pointer array
```

**These structures are:**
- Allocated based on `Aig_ManObjNumMax()` (AIG-specific)
- Accessed using `Aig_Obj_t*` pointers
- Used in pointer-based algorithms

**GIA equivalent would require:**
- Rewriting all allocation code
- Converting indices to pointers everywhere
- Changing all algorithm implementations

---

## Conversion Process

### Code Flow in `&dch -e`

**Location:** `src/base/abci/abc.c:48359-48367`

```c
if ( fEquiv )  // Mode 1: Compute and merge equivalences
{
    // Step 1: Convert GIA to AIG format
    // Location: src/aig/gia/giaAig.c
    Aig_Man_t * pNew = Gia_ManToAigSimple( pAbc->pGia );
    
    // Step 2: Find equivalent nodes using SAT sweeping
    // Location: src/proof/dch/dchCore.c:136
    // This function REQUIRES Aig_Man_t and uses:
    //   - Aig_ManFanoutStart()
    //   - Aig_ManReprStart()
    //   - Aig_Obj_t* pointers throughout
    Dch_ComputeEquivalences( pNew, (Dch_Pars_t *)pPars );
    
    // Step 3: Transfer equivalence classes back to GIA
    // Location: src/aig/gia/giaAig.c (Gia_ManReprFromAigRepr)
    // Converts AIG's pReprs (Aig_Obj_t* array) to GIA's pReprs (Gia_Rpr_t array)
    Gia_ManReprFromAigRepr( pNew, pAbc->pGia );
    
    // Step 4: Merge equivalent nodes (removes duplicates)
    // Location: src/aig/gia/giaEquiv.c:677
    // Uses GIA's pReprs structure (now populated from AIG)
    pTemp = Gia_ManEquivReduce( pAbc->pGia, 1, 0, 0, 0 );
    
    Aig_ManStop( pNew );  // Clean up AIG
}
```

---

### Detailed Conversion: `Gia_ManToAigSimple()`

**Location:** `src/aig/gia/giaAig.c`

**What it does:**
1. Creates new `Aig_Man_t` structure
2. Allocates individual `Aig_Obj_t` objects (not array)
3. Converts GIA's offset-based fanins to AIG's pointer-based fanins
4. Sets up AIG's `pReprs` array (initially NULL)
5. Returns `Aig_Man_t*` ready for DCH processing

**Key conversion:**
```c
// GIA: Fanins stored as offsets
int iFanin0 = Gia_ObjFaninId0(pGiaObj);
int iFanin1 = Gia_ObjFaninId1(pGiaObj);

// AIG: Fanins stored as pointers
Aig_Obj_t * pFanin0 = Aig_ManObj(pAig, iFanin0);
Aig_Obj_t * pFanin1 = Aig_ManObj(pAig, iFanin1);
pAigObj->pFanin0 = pFanin0;  // Direct pointer assignment
pAigObj->pFanin1 = pFanin1;
```

---

### Detailed Conversion: `Gia_ManReprFromAigRepr()`

**Location:** `src/aig/gia/giaAig.c` (or similar)

**What it does:**
1. Reads AIG's `pReprs` array (contains `Aig_Obj_t*` pointers)
2. Converts pointers to GIA object indices
3. Populates GIA's `pReprs` array (contains `Gia_Rpr_t` structures with indices)

**Key conversion:**
```c
// AIG: Representative stored as pointer
Aig_Obj_t * pRepr = pAig->pReprs[iObj];

// GIA: Representative stored as index in bitfield
int iRepr = Aig_ObjId(pRepr);  // Convert pointer to ID
pGia->pReprs[iObj].iRepr = iRepr;  // Store index in bitfield
```

---

## Code Flow Analysis

### Complete DCH Equivalence Computation Flow

```
┌─────────────────────────────────────────────────────────────┐
│ 1. GIA Input (pAbc->pGia)                                   │
│    - Gia_Man_t with Gia_Obj_t array                         │
│    - Fanins as offsets (iDiff0, iDiff1)                     │
│    - Representatives as indices (Gia_Rpr_t)                  │
└───────────────────────┬───────────────────────────────────────┘
                        │
                        ▼
┌─────────────────────────────────────────────────────────────┐
│ 2. Gia_ManToAigSimple()                                     │
│    Location: src/aig/gia/giaAig.c                           │
│    - Creates Aig_Man_t                                       │
│    - Allocates Aig_Obj_t objects individually               │
│    - Converts offsets → pointers                            │
│    - Initializes pReprs = NULL                              │
└───────────────────────┬───────────────────────────────────────┘
                        │
                        ▼
┌─────────────────────────────────────────────────────────────┐
│ 3. Dch_ComputeEquivalences()                                │
│    Location: src/proof/dch/dchCore.c:136                    │
│                                                              │
│    3a. Dch_ManCreate()                                      │
│        - Stores Aig_Man_t* in p->pAigTotal                 │
│        - Calls Aig_ManFanoutStart() [AIG-SPECIFIC]          │
│        - Allocates pSatVars using Aig_ManObjNumMax()        │
│        - Allocates pReprsProved as Aig_Obj_t**              │
│                                                              │
│    3b. Dch_CreateCandEquivClasses()                         │
│        - Uses Aig_Man_t*                                     │
│        - Returns Dch_Cla_t with Aig_Obj_t** arrays          │
│                                                              │
│    3c. Dch_ManSweep()                                       │
│        - Uses Aig_Obj_t* pointers throughout                │
│        - Accesses p->pAigTotal->pReprs (Aig_Obj_t** array) │
│        - Calls Dch_NodesAreEquiv() with Aig_Obj_t* params   │
└───────────────────────┬───────────────────────────────────────┘
                        │
                        ▼
┌─────────────────────────────────────────────────────────────┐
│ 4. Gia_ManReprFromAigRepr()                                 │
│    Location: src/aig/gia/giaAig.c                           │
│    - Reads Aig_Man_t->pReprs (Aig_Obj_t* array)             │
│    - Converts pointers → indices                            │
│    - Writes to Gia_Man_t->pReprs (Gia_Rpr_t array)          │
└───────────────────────┬───────────────────────────────────────┘
                        │
                        ▼
┌─────────────────────────────────────────────────────────────┐
│ 5. Gia_ManEquivReduce()                                      │
│    Location: src/aig/gia/giaEquiv.c:677                     │
│    - Uses GIA's pReprs (now populated)                      │
│    - Merges equivalent nodes                                │
│    - Returns reduced Gia_Man_t                              │
└───────────────────────┬───────────────────────────────────────┘
                        │
                        ▼
┌─────────────────────────────────────────────────────────────┐
│ 6. Result: GIA with merged equivalences                     │
│    - pAbc->pGia updated via Abc_FrameUpdateGia()            │
└─────────────────────────────────────────────────────────────┘
```

---

## Summary

### Why Conversion is Necessary

1. **Architectural Mismatch:**
   - AIG uses **pointers** (`Aig_Obj_t*`)
   - GIA uses **indices/offsets** (stored in bitfields)

2. **API Dependencies:**
   - DCH requires `Aig_ManFanoutStart()` - AIG-specific fanout structure
   - DCH requires `Aig_ManReprStart()` - AIG's pointer-based `pReprs` array
   - DCH uses `Aig_ManObjNumMax()` - AIG's object numbering scheme

3. **Data Structure Incompatibility:**
   - DCH stores `Aig_Obj_t**` arrays (pointer arrays)
   - DCH function signatures require `Aig_Obj_t*` parameters
   - DCH algorithms use pointer arithmetic throughout

4. **Historical Reasons:**
   - DCH was written for AIG format before GIA became standard
   - Rewriting DCH for GIA would require:
     - Changing all function signatures
     - Rewriting all algorithms to use indices
     - Converting all pointer operations to index lookups
     - Extensive testing and validation

### The Conversion Bridge

The conversion process (`Gia_ManToAigSimple` → DCH → `Gia_ManReprFromAigRepr`) acts as a **bridge** between:
- **GIA's efficient, compact representation** (used by ABC9 commands)
- **AIG's pointer-based algorithms** (used by DCH and other legacy code)

This allows ABC to:
- Use modern GIA format for most operations
- Leverage existing, proven AIG-based algorithms
- Maintain compatibility with legacy code

---

## References

### Key Files

1. **AIG Structures:**
   - `src/aig/aig/aig.h` - AIG manager and object definitions
   - `src/aig/aig/aigFanout.c` - Fanout management
   - `src/aig/aig/aigRepr.c` - Equivalence representation

2. **GIA Structures:**
   - `src/aig/gia/gia.h` - GIA manager and object definitions
   - `src/aig/gia/giaAig.c` - GIA↔AIG conversion functions

3. **DCH Implementation:**
   - `src/proof/dch/dchInt.h` - DCH data structures
   - `src/proof/dch/dchMan.c` - DCH manager creation
   - `src/proof/dch/dchCore.c` - Main DCH equivalence computation
   - `src/proof/dch/dchClass.c` - Equivalence class management

4. **Command Implementation:**
   - `src/base/abci/abc.c:48359-48367` - `&dch -e` command handler

---

## Representatives vs. Choice Nodes

### Overview

ABC uses two related but distinct mechanisms for handling functionally equivalent nodes:
1. **Representatives (`pReprs`)**: Equivalence classes for merging equivalent nodes
2. **Choice Nodes (`pSibls`)**: Structural alternatives kept for optimization flexibility

### Representatives (`pReprs`)

**Location:** `src/aig/gia/gia.h:126`

**Structure:**
```c
// src/aig/gia/gia.h:57-65
struct Gia_Rpr_t_
{
    unsigned       iRepr   : 28;  // representative node (INDEX)
    unsigned       fProved :  1;  // marks the proved equivalence
    unsigned       fFailed :  1;  // marks the failed equivalence
    unsigned       fColorA :  1;  // marks cone of A
    unsigned       fColorB :  1;  // marks cone of B
};

// In Gia_Man_t:
Gia_Rpr_t *    pReprs;        // representatives (for CIs and ANDs)
int *          pNexts;        // next nodes in the equivalence classes
```

**Purpose:**
- **Equivalence classes**: Nodes proven functionally equivalent (via SAT/simulation) form a class
- **One representative per class**: All nodes in a class point to a single representative node
- **Used for merging**: `Gia_ManEquivReduce()` removes duplicates, keeping only the representative

**Example:**
```c
// Nodes 10, 15, and 20 are proven equivalent
// Node 10 is chosen as representative
pReprs[15].iRepr = 10;  // Node 15 → Node 10
pReprs[20].iRepr = 10;  // Node 20 → Node 10
pReprs[10].iRepr = GIA_VOID;  // Representative points to itself (or VOID)

// After Gia_ManEquivReduce():
// - Only Node 10 remains
// - Nodes 15 and 20 are removed
// - All references to 15/20 are redirected to 10
```

**When created:**
- `&dch -e`: Creates representatives via `Dch_ComputeEquivalences()` → `Gia_ManEquivReduce()`
- Fraiging operations: SAT-based equivalence checking creates equivalence classes
- Simulation-based refinement: Identifies candidate equivalent nodes

---

### Choice Nodes (`pSibls`)

**Location:** `src/aig/gia/gia.h:128`

**Structure:**
```c
// In Gia_Man_t:
int *          pSibls;        // next nodes in the choice nodes
```

**Purpose:**
- **Structural alternatives**: Different implementations of the same function
- **Kept as alternatives**: All implementations remain in the network
- **Used for optimization**: Mapping/rewriting passes can choose the best alternative

**Example:**
```c
// Node 10 has two alternative implementations: Node 20 and Node 25
pSibls[20] = 10;  // Node 20 is an alternative to Node 10
pSibls[25] = 10;  // Node 25 is an alternative to Node 10
// Forms linked list: 10 ← 20 ← 25

// All three nodes remain in the network
// Optimizer can choose: 10 (area-optimal) or 20 (delay-optimal) or 25 (LUT-friendly)
```

**When created:**
- `&dch` (default): Creates choice nodes via `Gia_ManPerformDch()` → `Dar_ManChoiceNew()`
- From representatives: `Aig_ManMarkValidChoices()` converts representatives to choices
- Structural choice computation: Finds different structural implementations

---

### Relationship Between Representatives and Choice Nodes

**Key Insight:** Siblings (choice nodes) **do belong to the same equivalence class** and share the same representative.

**Conversion Process:**

**Location:** `src/aig/aig/aigRepr.c:481-519`

```c
// src/aig/aig/aigRepr.c:481
void Aig_ManMarkValidChoices( Aig_Man_t * p )
{
    Aig_Obj_t * pObj, * pRepr;
    int i;
    assert( p->pReprs != NULL );
    
    // Create choice nodes from equivalence classes
    p->pEquivs = ABC_ALLOC( Aig_Obj_t *, Aig_ManObjNumMax(p) );
    
    Aig_ManForEachNode( p, pObj, i )
    {
        // Get representative of this node
        pRepr = Aig_ObjFindRepr( p, pObj );
        if ( pRepr == NULL )
            continue;
        
        // Skip nodes with fanouts (can't be choices)
        if ( pObj->nRefs > 0 )
            continue;
        
        // Add as choice node: link to representative's choice list
        p->pEquivs[pObj->Id] = p->pEquivs[pRepr->Id];
        p->pEquivs[pRepr->Id] = pObj;  // Add to head of list
    }
}
```

**Relationship Diagram:**

```
Equivalence Class (pReprs):
┌─────────────────────────────────────┐
│ Representative: Node 10              │
│   pReprs[10].iRepr = GIA_VOID       │
│   pReprs[15].iRepr = 10  ←──────────┼───┐
│   pReprs[20].iRepr = 10  ←──────────┼───┼───┐
│   pReprs[25].iRepr = 10  ←──────────┼───┼───┼───┐
└─────────────────────────────────────┘   │   │   │
                                           │   │   │
Choice Nodes (pSibls) - if kept:          │   │   │
┌─────────────────────────────────────┐   │   │   │
│ Node 10 (representative)             │◄──┘   │   │
│   pSibls[10] = 0 (no sibling)        │       │   │
│                                       │       │   │
│ Node 15 (alternative)                 │◄──────┘   │
│   pSibls[15] = 10                    │           │
│                                       │           │
│ Node 20 (alternative)                 │◄──────────┘
│   pSibls[20] = 10                    │
│                                       │
│ Node 25 (alternative)                 │
│   pSibls[25] = 10                    │
└─────────────────────────────────────┘
```

**Important Points:**

1. **Same equivalence class**: All siblings share the same representative
   - `pReprs[15].iRepr = 10`
   - `pReprs[20].iRepr = 10`
   - `pReprs[25].iRepr = 10`

2. **Not all equivalent nodes become choices**: Only nodes without fanouts (`nRefs == 0`) can become choice nodes
   - Nodes with fanouts are merged immediately (using representatives)
   - Nodes without fanouts can be kept as alternatives (choice nodes)

3. **Two-phase process**:
   - **Phase 1**: Create equivalence classes (representatives) via SAT/simulation
   - **Phase 2**: Convert to choices (siblings) for nodes that can be alternatives

4. **Usage difference**:
   - **Representatives**: Used for immediate merging (`Gia_ManEquivReduce()`)
   - **Choice nodes**: Used for delayed optimization (mapping/rewriting picks best)

---

### Code Examples

#### Creating Representatives

**Location:** `src/base/abci/abc.c:48359-48367`

```c
if ( fEquiv )  // &dch -e mode
{
    // Step 1: Find equivalent nodes
    Dch_ComputeEquivalences( pNew, pPars );
    // Result: pAig->pReprs populated with equivalence classes
    
    // Step 2: Transfer to GIA
    Gia_ManReprFromAigRepr( pNew, pAbc->pGia );
    // Result: pGia->pReprs populated
    
    // Step 3: Merge equivalent nodes
    pTemp = Gia_ManEquivReduce( pAbc->pGia, 1, 0, 0, 0 );
    // Result: Only representatives remain, duplicates removed
}
```

#### Creating Choice Nodes

**Location:** `src/base/abci/abc.c:48368-48374`

```c
else  // &dch mode (default)
{
    // Step 1: Find structural choices
    pTemp = Gia_ManPerformDch( pAbc->pGia, pPars );
    // Result: pGia->pSibls populated with choice relationships
    
    // Step 2: Optionally reduce choices
    if ( fMinLevel || fRandom ) 
        pTemp = Gia_ManEquivReduce2( pAbc->pGia, fRandom );
    // Result: Choices optimized but alternatives kept
}
```

#### Converting Representatives to Choices

**Location:** `src/aig/aig/aigRepr.c:481`

```c
// After equivalence classes are created (pReprs populated):
Aig_ManMarkValidChoices( pAig );

// This function:
// 1. Iterates through all nodes with representatives
// 2. For nodes with nRefs == 0 (no fanouts):
//    - Adds them to choice list of their representative
// 3. Result: pEquivs array contains choice relationships
```

---

### Summary

| Aspect | Representatives (`pReprs`) | Choice Nodes (`pSibls`) |
|--------|---------------------------|------------------------|
| **Purpose** | Merge equivalent nodes | Keep alternatives for optimization |
| **Structure** | `Gia_Rpr_t` array with indices | `int*` array forming linked lists |
| **Relationship** | One representative per equivalence class | Siblings share same representative |
| **When created** | `&dch -e`, fraiging, simulation | `&dch`, structural choice computation |
| **Usage** | Immediate reduction | Delayed optimization |
| **Result** | Network with duplicates removed | Network with alternatives kept |

**Key Relationship:**
- ✅ **Siblings belong to the same equivalence class** (share the same representative)
- ✅ **Not all nodes in a class become choices** (only those without fanouts)
- ✅ **Representatives are created first**, then converted to choices if needed
- ✅ **Both mechanisms track the same functional equivalence**, but serve different optimization purposes

---

*Document created: 2026-01-13*
*ABC Version: Current development version*
