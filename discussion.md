

Summary of work needed to be done in order to preserve node

Questions:
1) In the bounty, there's the following commands which need to preserve the nodes:
- `&get -n` — [src/base/abci/abc.c:1236](src/base/abci/abc.c#L1236) — registers `Abc_CommandAbc9Get`
- `&put` — [src/base/abci/abc.c:1237](src/base/abci/abc.c#L1237) — registers `Abc_CommandAbc9Put`
- `&st` — [src/base/abci/abc.c:1270](src/base/abci/abc.c#L1270) — registers `Abc_CommandAbc9Strash`
- `&syn2` — [src/base/abci/abc.c:1308](src/base/abci/abc.c#L1308) — registers `Abc_CommandAbc9Syn2`
- `&sweep` — [src/base/abci/abc.c:1334](src/base/abci/abc.c#L1334) — registers `Abc_CommandAbc9Sweep`
- `&if -g -K 6` — [src/base/abci/abc.c:1342](src/base/abci/abc.c#L1342) — registers `Abc_CommandAbc9If`
- `&nf` — [src/base/abci/abc.c:1351](src/base/abci/abc.c#L1351) — registers `Abc_CommandAbc9Nf`
- `&dch` — [src/base/abci/abc.c:1377](src/base/abci/abc.c#L1377) — registers `Abc_CommandAbc9Dch`

Firstly, we start with the following command:

- `&get -n` — [src/base/abci/abc.c:1236](src/base/abci/abc.c#L1236) — registers `Abc_CommandAbc9Get`



Abc_CommandAbc9Get calls either one of these:
    Gia_ManFromAig which copies over the nodes from Aig_Man_t
    Abc_NtkAigToGia which copies over the nodes from ABC_Ntk_t

    After this, it executes the follwing commands

    if ( fNames )
    {
        pGia->vNamesIn  = Abc_NtkCollectCiNames( pAbc->pNtkCur );
        pGia->vNamesOut = Abc_NtkCollectCoNames( pAbc->pNtkCur );
    }


    These are both Vec_Ptr_t of the CI and CO names.

    Another thing to note: the central data structure which stores the names is this:
        
    

    - consider fNames (which needs to be called for this to happen)

---

## API Usage: Vec_Vec_t - Vector of Vectors

**Location:** `src/misc/vec/vecVec.h`

### Type Definition

```c
typedef struct Vec_Vec_t_ Vec_Vec_t;
struct Vec_Vec_t_ 
{
    int    nCap;      // Capacity
    int    nSize;     // Current size (number of levels)
    void ** pArray;   // Array of Vec_Ptr_t* (or Vec_Int_t*)
};
```

### Basic Operations

#### 1. Allocation

```c
// Allocate empty vector of vectors
Vec_Vec_t * vVec = Vec_VecAlloc( 0 );

// Allocate with pre-initialized levels (each level is an empty Vec_Ptr_t)
Vec_Vec_t * vVec = Vec_VecStart( 10 );  // Creates 10 levels
```

#### 2. Adding Entries

```c
// Push pointer to a specific level (creates level if it doesn't exist)
Vec_VecPush( vVec, level, (void*)someObject );

// Push integer to a specific level (uses Vec_Int_t internally)
Vec_VecPushInt( vVec, level, someInteger );

// Push unique (only if not already present)
Vec_VecPushUnique( vVec, level, (void*)someObject );
Vec_VecPushUniqueInt( vVec, level, someInteger );
```

#### 3. Accessing Entries

```c
// Get a level (returns Vec_Ptr_t*)
Vec_Ptr_t * vLevel = Vec_VecEntry( vVec, level );

// Get a level (returns Vec_Int_t*)
Vec_Int_t * vLevel = Vec_VecEntryInt( vVec, level );

// Get specific entry at level i, position k
void * entry = Vec_VecEntryEntry( vVec, i, k );
int entry = Vec_VecEntryEntryInt( vVec, i, k );

// Get size of a specific level
int levelSize = Vec_VecLevelSize( vVec, level );

// Get total number of levels
int numLevels = Vec_VecSize( vVec );
```

#### 4. Iteration

```c
// Iterate through levels
Vec_Ptr_t * vLevel;
int i;
Vec_VecForEachLevel( vVec, vLevel, i )
{
    // Process each level
    void * pEntry;
    int k;
    Vec_PtrForEachEntry( Type, vLevel, pEntry, k )
    {
        // Process each entry in this level
    }
}

// Iterate through all entries across all levels
void * pEntry;
int i, k;
Vec_VecForEachEntry( Type, vVec, pEntry, i, k )
{
    // Process each entry
}

// For integer vectors
int Entry;
int i, k;
Vec_VecForEachEntryInt( vVec, Entry, i, k )
{
    // Process each integer entry
}
```

#### 5. Memory Management

```c
// Free the entire vector of vectors (frees all levels)
Vec_VecFree( vVec );

// Free and set pointer to NULL
Vec_VecFreeP( &vVec );

// Clear all levels (but keep structure)
Vec_VecClear( vVec );

// Duplicate
Vec_Vec_t * vCopy = Vec_VecDup( vVec );        // For Vec_Ptr_t levels
Vec_Vec_t * vCopy = Vec_VecDupInt( vVec );     // For Vec_Int_t levels
```

#### 6. Expansion

```c
// Expand to ensure level exists (auto-creates intermediate levels)
Vec_VecExpand( vVec, level );      // For Vec_Ptr_t levels
Vec_VecExpandInt( vVec, level );   // For Vec_Int_t levels
```

### Real-World Example

From `src/proof/pdr/pdrCore.c`:

```c
// Allocate vector of vectors to store clauses at different levels
p->vClauses = Vec_VecAlloc( 0 );

// Push a clause to a specific level
Vec_VecPush( p->vClauses, level, pCubeMin );

// Access clauses at a level
Vec_Ptr_t * vLevel = Vec_VecEntry( p->vClauses, level );
int numClauses = Vec_PtrSize( vLevel );

// Iterate through all levels
Vec_Ptr_t * vLevel;
int i;
Vec_VecForEachLevel( p->vClauses, vLevel, i )
{
    // Process clauses at level i
}

// Free when done
Vec_VecFree( p->vClauses );
```

### Key Points

- **Each level** is a `Vec_Ptr_t*` (for `Vec_VecPush`) or `Vec_Int_t*` (for `Vec_VecPushInt`)
- **Levels are created automatically** when you push to them
- **Use `Vec_VecFree()`** to properly free all levels and the container
- **Useful for**: Storing objects grouped by levels, frames, or any hierarchical structure











Abc_CommandAbc9Dch and choice nodes


Using the printlevel function in order to print the level etc.


Abc_FrameUpdateGia -> this function already hanels the swapping of names
This is the main function which calls for an updated gia frame upon any o fthose commands


This is one of the fields within these nodes
    Vec_Ptr_t *    vNamesNode;    // the node names



There's two possible approaches
1) Working forwards
2) Working backwords
 - ABC_Frame_t has 4 copies of Gia_man_t which means that backtracking or peeking up to 5 steps behing is possible:
    Gia_Man_t *     pGia;          // alternative current network as a light-weight AIG
    Gia_Man_t *     pGia2;         // copy of the above
    Gia_Man_t *     pGiaBest;      // copy of the above
    Gia_Man_t *     pGiaBest2;     // copy of the above
    Gia_Man_t *     pGiaSaved;     // copy of the above


Two types of alterations and optimizations:
1) Structural -> native and happen thanks to the AND gates'
2) Functional -> &dch and fraiging

## ABC9 Commands and pGia

**ABC9 commands only work with `Gia_Man_t`** - they access `pAbc->pGia` directly from `Abc_Frame_t`.

**Setting `pGia`** - two ways:
- **`&get`**: Converts existing `Abc_Ntk_t` → `Gia_Man_t` and sets `pAbc->pGia` via `Abc_FrameUpdateGia()`
- **`&read` / `&r`**: Reads files directly into GIA format and sets `pAbc->pGia` (no `&get` needed)

**Typical workflow:**
- If you have an `Abc_Ntk_t` (from `read_blif`, etc.): use `&get` to convert it
- If reading a file: use `&read <file>` to load directly into GIA space

All ABC9 commands are wrappers in `abc.c` that access `pAbc->pGia` directly, so no `&get` is needed if `pGia` is already set.








From the detailed commands, this is the more difficult aspect:

int Abc_CommandAbc9Dch( Abc_Frame_t * pAbc, int argc, char ** argv )
{
    Gia_Man_t * pTemp;
    int fEquiv = 0;  // -e flag
    
    // ... parse arguments ...
    
    if ( fEquiv )  // Mode 1: Compute and merge equivalences
    {
        // Step 1: Convert GIA to AIG format
        Aig_Man_t * pNew = Gia_ManToAigSimple( pAbc->pGia );
        
        // Step 2: Find equivalent nodes using SAT sweeping
        Dch_ComputeEquivalences( pNew, pPars );
        
        // Step 3: Transfer equivalence classes back to GIA
        Gia_ManReprFromAigRepr( pNew, pAbc->pGia );
        
        // Step 4: Merge equivalent nodes (removes duplicates)
        pTemp = Gia_ManEquivReduce( pAbc->pGia, 1, 0, 0, 0 );
        // Result: Network with equivalent nodes merged
    }
    else  // Mode 2: Compute structural choices (default)
    {
        // Step 1: Find structural choices (different implementations)
        pTemp = Gia_ManPerformDch( pAbc->pGia, pPars );
        // Result: Network with choice nodes (pSibls array populated)
        
        // Step 2: Optionally reduce choices
        if ( fMinLevel || fRandom ) 
            pTemp = Gia_ManEquivReduce2( pAbc->pGia, fRandom );
        // Result: Network with choices optimized by level or randomly
    }
    
    Abc_FrameUpdateGia( pAbc, pTemp );
    return 0;
}


Also nessessitates this:

Gia_ManToAig




Workflow:
- Run a few different test suites annotated with print_level in order to be able to see the nodes

Reasons)
1) So that we know where it's actually needed
2) Mark down areas where the transitions are lost
 - For more fine grained debugging, I'll define macros within functions with a certain debug config
    so that it's able to determine where in the function the information is lost


Reasoninig: Gia_ManPerformDch converts to the Aig_man_t type for certain reasons. This means, that in
order to keep the mapping, there must be a new mapping type in this Aig_man_t as well

Approach for this: if it's nullptr, just ignore it. Otherwise the mapping exists (in case this is only)
needed during the dch computation.


Some functions which have to be changed:
Dar_ManChoiceNew
Gia_ManFromAigChoices
Gia_ManToAig

