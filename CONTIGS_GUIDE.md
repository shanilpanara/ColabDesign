# ColabDesign Contigs Guide

## Table of Contents
1. [What are Contigs?](#what-are-contigs)
2. [Basic Syntax](#basic-syntax)
3. [Residue Numbering](#residue-numbering)
4. [Single Chain Examples](#single-chain-examples)
5. [Multi-Chain Examples](#multi-chain-examples)
6. [Common Use Cases](#common-use-cases)
7. [Advanced Options](#advanced-options)

---

## What are Contigs?

Contigs in ColabDesign provide a flexible way to define protein design problems by combining:
- **Template regions** (from existing PDB structures)
- **Design/Prediction regions** (new residues to be designed or predicted)

Think of contigs as a blueprint that tells ColabDesign which parts to copy from a template and which parts to design/predict from scratch.

---

## Basic Syntax

### Contig String Format
```
"segment1/segment2/segment3:segment4/segment5"
```

- **`/`** separates segments within the same chain
- **`:` or `,`** both separate different chains (interchangeable)
- **Segments** can be:
  - **Template segments**: `A10-50` (chain A, residues 10-50)
  - **Design segments**: `30` (design 30 new residues)
  - **Variable length**: `20-40` (design 20-40 residues, randomly chosen)

### Segment Types

#### 1. Template Segments (Supervised)
```
A10-50    # Chain A, residues 10 through 50 (PDB numbering)
B5-100    # Chain B, residues 5 through 100
A         # Entire chain A
```

#### 2. Design Segments (Unsupervised)
```
30        # Design exactly 30 residues
10-20     # Design between 10 and 20 residues (random)
```

---

## Residue Numbering

**IMPORTANT**: When you specify residues like `A10-50`, you are using **PDB residue numbers**, not sequential positions.

### Example PDB Residue Numbering
```
PDB File:
Chain A: 5, 6, 7, 8, 9, 10, 11, ... 50

When you specify A10-50:
- You get residues with PDB numbers 10-50 in chain A
- NOT the 10th through 50th residue in the chain
```

### Handling Missing Residues

If your PDB has missing residues (common in crystal structures):

```python
# Option 1: Ignore missing residues (recommended)
model.prep_inputs(
    contigs="A10-100",
    pdb_filename="structure.pdb",
    ignore_missing=True  # Skip missing residues
)

# Option 2: Pad with undefined residues
model.prep_inputs(
    contigs="A10-100",
    pdb_filename="structure.pdb",
    ignore_missing=False  # Pad missing positions
)
```

---

## Single Chain Examples

### 1. Hallucination (Design from Scratch)
```python
model = mk_af_model('contigs')
model.prep_inputs(contigs="100")  # Design a 100-residue protein
```

### 2. Fixed Backbone Design
```python
# Redesign the entire chain A
model.prep_inputs(
    contigs="A",
    pdb_filename="my_structure.pdb",
    fix_pos="A"  # Fix the backbone
)
```

### 3. Partial Template - Extend Termini
```python
# Add residues to N and C termini
model.prep_inputs(
    contigs="20/A10-100/30",  # 20 new N-term, template, 30 new C-term
    pdb_filename="my_structure.pdb"
)
```

### 4. Partial Template - Internal Gaps
```python
# Design a loop between two template regions
model.prep_inputs(
    contigs="A10-50/15-25/A80-120",  # Template/design 15-25 residues/template
    pdb_filename="my_structure.pdb"
)
```

### 5. Scaffold an Active Site
```python
# Keep specific residues, design around them
model.prep_inputs(
    contigs="30/A45-55/40/A100-110/30",
    pdb_filename="enzyme.pdb",
    fix_pos="A45-55,A100-110"  # Fix active site residues
)
```

---

## Multi-Chain Examples

### 1. Two-Chain Design with Partial Templates
```python
# Chain A: partial template with designed regions
# Chain B: partial template with designed regions
model.prep_inputs(
    contigs="A10-50/20/A100-150:B5-30/15/B80-100",
    pdb_filename="complex.pdb"
)
```

This creates:
- **Chain 1**: A(10-50) → 20 designed → A(100-150)
- **Chain 2**: B(5-30) → 15 designed → B(80-100)

### 2. Binder Design (Target + Designed Binder)
```python
# Keep chain A as target, design a 50-residue binder
model.prep_inputs(
    contigs="A:50",  # Full chain A + 50 designed residues
    pdb_filename="target.pdb",
    fix_pos="A",     # Fix target backbone
    hotspot="A25-35" # Focus binding on these residues
)
```

### 3. Multi-Chain Scaffold
```python
# Use parts of multiple chains as scaffolds
model.prep_inputs(
    contigs="A20-80:B30-90:C10-60",
    pdb_filename="complex.pdb"
)
```

### 4. Design Multi-Chain Interface
```python
# Design interface residues between two chains
model.prep_inputs(
    contigs="A1-40/20/A61-100:B1-30/15/B46-80",
    pdb_filename="complex.pdb",
    fix_pos="A1-40,A61-100,B1-30,B46-80"  # Fix non-interface regions
)
```

---

## Common Use Cases

### Use Case 1: Protein Binder Design

**Goal**: Design a binder for a target protein

```python
from colabdesign import mk_af_model

# Create model
model = mk_af_model(protocol="contigs")

# Target chain A, design 60-80 residue binder
model.prep_inputs(
    contigs="A:60-80",           # Target:Binder
    pdb_filename="target.pdb",
    fix_pos="A",                 # Fix target backbone
    hotspot="A45,A67-70,A89",   # Key binding residues
)

# Run design
model.design_3stage()
```

### Use Case 2: Loop Design

**Goal**: Design a loop connecting two structured regions

```python
model = mk_af_model(protocol="contigs")

# Design 10-20 residue loop between A30 and A50
model.prep_inputs(
    contigs="A1-30/10-20/A50-100",
    pdb_filename="structure.pdb",
    fix_pos="A1-30,A50-100"  # Fix flanking regions
)

model.design_3stage()
```

### Use Case 3: Domain Insertion

**Goal**: Insert a functional domain into a scaffold

```python
model = mk_af_model(protocol="contigs")

# Insert domain (chain B) into scaffold (chain A)
model.prep_inputs(
    contigs="A1-50/5-10/B/5-10/A51-150",
    pdb_filename="scaffold_and_domain.pdb",
    fix_pos="B"  # Fix the inserted domain
)

model.design_3stage()
```

### Use Case 4: N/C-Terminal Extensions

**Goal**: Add functional elements to termini

```python
model = mk_af_model(protocol="contigs")

# Add 20 residues to N-terminus, 30 to C-terminus
model.prep_inputs(
    contigs="20/A/30",
    pdb_filename="core_protein.pdb",
    fix_pos="A"  # Fix core
)

model.design_3stage()
```

### Use Case 5: Multi-Chain Complex Assembly

**Goal**: Design multiple chains simultaneously

```python
model = mk_af_model(protocol="contigs")

# Design parts of three chains
model.prep_inputs(
    contigs="A20-80/10/A95-120:B:30-50",  # Chain1:Chain2:Chain3
    pdb_filename="complex.pdb",
    copies=1  # Not a homo-oligomer
)

model.design_3stage()
```

### Use Case 6: Symmetric Homo-oligomer

**Goal**: Design a symmetric complex

```python
model = mk_af_model(protocol="contigs")

# Design a trimer
model.prep_inputs(
    contigs="80",  # Single chain
    copies=3,      # Make trimer
    homooligomer=True  # Symmetric copies
)

model.design_3stage()
```

---

## Advanced Options

### Position-Specific Constraints

When calling `prep_inputs()`, you can specify various position constraints:

```python
model.prep_inputs(
    contigs="A10-50/30/A80-120:50",
    pdb_filename="structure.pdb",

    # Fix specific positions (don't design, keep template)
    fix_pos="A10-20,A100-110",

    # Hotspot residues (for binder design - focus here)
    hotspot="A30-35",

    # Remove template information at specific positions
    rm_template="A40-45",      # Remove all template info
    rm_template_seq="A50-55",  # Remove sequence, keep geometry
    rm_template_sc="A60-65",   # Remove sidechains, keep backbone
)
```

### Template Control Options

```python
model.prep_inputs(
    contigs="A:50",
    pdb_filename="target.pdb",

    # Target template controls
    rm_target=False,      # Keep target template
    rm_target_seq=False,  # Keep target sequence
    rm_target_sc=False,   # Keep target sidechains

    # Binder template controls
    rm_binder=True,       # Remove binder template
    rm_binder_seq=True,   # Remove binder sequence
    rm_binder_sc=True,    # Remove binder sidechains

    # Inter-chain template
    rm_template_ic=True,  # Remove inter-chain template contacts
)
```

### Loss Function Control

```python
model.prep_inputs(
    contigs="A10-100/50",
    pdb_filename="structure.pdb",
    partial_loss=True  # Only compute loss on template regions (not designed)
)
```

### Handling Multiple Copies

```python
# Symmetric homo-oligomer
model.prep_inputs(
    contigs="A",
    pdb_filename="monomer.pdb",
    copies=3,
    homooligomer=True  # All copies identical
)

# Asymmetric hetero-oligomer
model.prep_inputs(
    contigs="A:B",
    pdb_filename="heterodimer.pdb",
    copies=2,
    homooligomer=False  # Different chains
)
```

---

## Quick Reference

### Syntax Cheatsheet
```
A             # Entire chain A from PDB
A10-50        # Chain A, residues 10-50 (PDB numbering)
30            # Design 30 residues
20-40         # Design 20-40 residues (random)
A/B           # Segment A, then segment B (same chain)
A:B or A,B    # Chain A, then chain B (: and , are interchangeable)
```

### Common Patterns
```python
# Hallucination
contigs="100"

# Fixed backbone
contigs="A"

# Binder design
contigs="A:50-80"  # or "A,50-80"

# Loop design
contigs="A10-50/15/A70-120"

# N/C extension
contigs="20/A/30"

# Multi-chain
contigs="A:B:50"  # or "A,B,50"

# Homo-oligomer
contigs="80", copies=3, homooligomer=True
```

---

## Tips and Best Practices

1. **Start Simple**: Test with smaller designs first
2. **Check PDB Numbering**: Use a PDB viewer to verify residue numbers
3. **Use `ignore_missing=True`**: Recommended for most crystal structures
4. **Fix What You Want to Keep**: Use `fix_pos` for regions that shouldn't move
5. **Hotspots for Binders**: Define `hotspot` to focus binding interface
6. **Validation**: Always validate your contig string matches your intention

---

## Troubleshooting

### Error: "positions X and chain Y not found"
- Check PDB residue numbering (not sequential positions)
- Verify chain IDs in your PDB file
- Try `ignore_missing=True`

### Design looks wrong
- Verify contig string matches your intention
- Check that `fix_pos` includes regions you want fixed
- Ensure template regions are correctly specified

### Multi-chain issues
- Use `:` or `,` to separate chains, not `/`
- Both `:` and `,` work identically for separating chains
- Each chain segment should start with chain ID (e.g., `A10-50`, `B5-30`)
- Verify chain IDs exist in PDB

---

## Examples Repository

For more examples, check:
- [ColabDesign GitHub Examples](https://github.com/sokrypton/ColabDesign/tree/main/examples)
- [AlphaFold Tutorials](https://github.com/sokrypton/ColabDesign/tree/main/af)

---

**Last Updated**: 2025-10-29
**ColabDesign Version**: Compatible with latest version

