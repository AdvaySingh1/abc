# Gia_ManPerformDch - Annotated Code Walkthrough

## Function: `Gia_ManPerformDch`

**Location:** `src/aig/gia/giaAig.c` (lines 639-663)

**Purpose:** Performs choice computation on a GIA (Generalized Inverter Array) manager. This function identifies equivalent nodes in the circuit and creates choice nodes, allowing the optimizer to select among equivalent implementations.

---

## Annotated Code

```c
Gia_Man_t * Gia_ManPerformDch( Gia_Man_t * p, void * pPars )
{
    // Flag to enable mapping-based derivation (currently disabled)
    int fUseMapping = 0;
    // Result GIA manager and intermediate working copy
    Gia_Man_t * pGia, * pGia1;
    // Intermediate AIG representation for choice computation
    Aig_Man_t * pNew;
    
    // Step 1: Ensure timing levels are computed if timing information exists
    // This is needed for choice computation which may use level information
    // p->pManTime indicates timing manager exists, p->vLevels stores level info
    if ( p->pManTime && p->vLevels == NULL )
        Gia_ManLevelWithBoxes( p );
    
    // Step 2: Prepare working copy of the GIA manager
    // Option 1: If mapping is enabled and the GIA has mapping info, derive from mapping
    // Option 2: Otherwise, simply duplicate the GIA manager (current path)
    if ( fUseMapping && Gia_ManHasMapping(p) )
        pGia1 = (Gia_Man_t *)Dsm_ManDeriveGia( p, 0 );
    else
        pGia1 = Gia_ManDup( p );
    
    // Step 3: Convert GIA format to AIG format for choice computation
    // Choice computation (Dar_ManChoiceNew) operates on AIG representation
    pNew = Gia_ManToAig( pGia1, 0 );
    // Free the intermediate GIA copy since we now have the AIG version
    Gia_ManStop( pGia1 );
    
    // Step 4: Perform choice computation on the AIG
    // This identifies equivalent nodes and creates choice nodes to represent them
    // Choice nodes allow the optimizer to select among equivalent implementations
    pNew = Dar_ManChoiceNew( pNew, (Dch_Pars_t *)pPars );
    
    // Step 5: Convert back from AIG to GIA format
    // Use Gia_ManFromAigChoices (not Gia_ManFromAig) to preserve choice information
    // The commented line would lose choice information
//    pGia = Gia_ManFromAig( pNew );
    pGia = Gia_ManFromAigChoices( pNew );
    // Free the AIG representation since we now have the GIA version
    Aig_ManStop( pNew );
    
    // Step 6: Validate that choices were actually created
    // If no timing info exists and no choices were found, the optimization failed
    // In this case, return a duplicate of the original (unchanged) GIA manager
    if ( !p->pManTime && !Gia_ManTestChoices(pGia) )
    {
        Gia_ManStop( pGia );
        pGia = Gia_ManDup( p );
    }
    
    // Step 7: Transfer timing information from original to result
    // This preserves timing constraints and level information in the optimized result
    Gia_ManTransferTiming( pGia, p );
    return pGia;
}
```

---

## Step-by-Step Explanation

### Step 1: Timing Level Computation
- **Check:** If timing information exists (`p->pManTime`) but levels haven't been computed yet (`p->vLevels == NULL`)
- **Action:** Compute levels using `Gia_ManLevelWithBoxes()`
- **Why:** Choice computation may need level information for optimization

### Step 2: Working Copy Preparation
- **Current path:** Always duplicates the GIA manager (`Gia_ManDup`)
- **Alternative path (disabled):** If `fUseMapping` were enabled, could derive from mapping using `Dsm_ManDeriveGia`
- **Why:** We need a working copy to avoid modifying the original

### Step 3: Format Conversion (GIA → AIG)
- **Action:** Convert from GIA format to AIG format using `Gia_ManToAig()`
- **Why:** The choice computation algorithm (`Dar_ManChoiceNew`) operates on AIG representation
- **Cleanup:** Free the intermediate GIA copy since we now have the AIG version

### Step 4: Choice Computation
- **Action:** Call `Dar_ManChoiceNew()` to identify equivalent nodes and create choice nodes
- **What are choices?** Choice nodes represent multiple equivalent implementations of the same logic, allowing the optimizer to select the best one

### Step 5: Format Conversion (AIG → GIA)
- **Action:** Convert back from AIG to GIA using `Gia_ManFromAigChoices()`
- **Important:** Must use `Gia_ManFromAigChoices()` (not `Gia_ManFromAig()`) to preserve choice information
- **Cleanup:** Free the AIG representation

### Step 6: Validation
- **Check:** If no timing info exists and no choices were actually created
- **Action:** If validation fails, discard the result and return a duplicate of the original (unchanged)
- **Why:** If no choices were found, the optimization didn't help, so return the original

### Step 7: Timing Transfer
- **Action:** Transfer timing information from the original GIA manager to the result
- **Why:** Preserve timing constraints and level information in the optimized result

---

## Key Concepts

- **GIA (Generalized Inverter Array):** A data structure for representing logic circuits
- **AIG (And-Inverter Graph):** An alternative representation using only AND gates and inverters
- **Choice nodes:** Special nodes that represent multiple equivalent implementations, enabling optimization
- **Timing information:** Constraints and level information used for timing-aware optimization
