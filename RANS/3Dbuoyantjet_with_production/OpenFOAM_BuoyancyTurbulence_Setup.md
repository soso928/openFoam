# OpenFOAM: Buoyancy Production in k-Epsilon (Incompressible/Boussinesq)

## Problem
`buoyantKEpsilon` is compressible-only — it requires `grad(rho)` which is zero in incompressible flow.
Standard `kEpsilon` has no buoyancy terms by default (`kSource()` and `epsilonSource()` return zero).

---

## Solution: Use `buoyancyTurbSource` via fvOptions

This adds the buoyancy production term `Gb` to both the k and epsilon equations, compatible with
incompressible Boussinesq solvers such as `buoyantSimpleFoam`, `buoyantBoussinesqPimpleFoam`, etc.

### Equations

**k equation — buoyancy production:**
```
Gb = beta * (nu_t / Prt) * (g · grad(T))
```

**epsilon equation — buoyancy dissipation:**
```
S_eps = C3 * (epsilon / k) * Gb
```

---

## Required Files

### 1. `system/fvOptions`
```c++
FoamFile
{
    version     2.0;
    format      ascii;
    class       dictionary;
    object      fvOptions;
}

buoyancyTurbulenceSource
{
    type            buoyancyTurbSource;
    active          yes;
    selectionMode   all;
}
```

### 2. `constant/momentumTransport`
```c++
simulationType  RAS;

RAS
{
    model       kEpsilon;
    turbulence  on;
    printCoeffs on;

    kEpsilonCoeffs
    {
        C3    0.85;   // buoyancy effect on epsilon (see guidance below)
    }
}
```

### 3. `constant/transportProperties` (must already have these)
```c++
beta    3e-04;      // thermal expansion coefficient [1/K]
TRef    293.15;     // reference temperature [K]
Prt     0.85;       // turbulent Prandtl number
```

### 4. `constant/g` (must exist)
```c++
dimensions  [0 1 -2 0 0 0 0];
value       (0 0 -9.81);
```

---

## Verification — Check Log at Startup

Run should print:
```
Selecting finite volume options type buoyancyTurbSource
    Source: buoyancyTurbulenceSource
    - selecting all cells
    - Applying buoyancyTurbSource to: epsilon and k
```

---

## C3 Guidance for Buoyant Jets

| Jet orientation relative to gravity | Recommended C3 |
|--------------------------------------|----------------|
| Vertical (rising/sinking)            | 1.0            |
| Horizontal                           | 0.0            |
| Inclined                             | 0.5 – 0.85     |

`C3 = 0` means buoyancy only affects k, not epsilon.
`C3 = 1` means full buoyancy effect on both k and epsilon.

---

## Notes

- `buoyantKEpsilon` **cannot** be used with incompressible solvers — it requires density gradients.
- The `C3` coefficient in standard `kEpsilon` without `buoyancyTurbSource` controls the **dilatation**
  term (`divU`), which vanishes in incompressible flow — it is NOT the buoyancy C3.
- `buoyancyTurbSource` reads `beta`, `TRef`, `Prt` automatically from `transportProperties`.
- Without buoyancy in turbulence, standard `kEpsilon` can underestimate mixing in buoyant jets
  by 20–40% in the near field.
