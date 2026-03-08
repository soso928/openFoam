# Adding Buoyancy to k-Epsilon in OpenFOAM via fvOptions
### A Step-by-Step Guide for Buoyant Jet Applications (v2406)

---

## Background

### Why not just use `buoyantKEpsilon`?

`buoyantKEpsilon` is a dedicated model that adds buoyancy terms to k and epsilon. However, it is **compressible-only** because it computes buoyancy from density gradients:

```
Gb = Cg * Cmu * alpha * k * (g · grad(rho)) / epsilon
```

In incompressible flow, `rho` is constant so `grad(rho) = 0` — the entire buoyancy term vanishes. This is why it does not appear in the runtime selection table when running an incompressible or Boussinesq solver.

### The alternative: `buoyancyTurbSource` via fvOptions

OpenFOAM provides a source term called `buoyancyTurbSource` that can be injected into the k and epsilon equations at runtime via `fvOptions`. It uses the Boussinesq approximation instead:

```
Gb = beta * (nu_t / Prt) * (g · grad(T))
```

This works with **incompressible solvers** that solve a temperature equation, such as:
- `buoyantSimpleFoam`
- `buoyantBoussinesqSimpleFoam`
- `buoyantPimpleFoam`
- `buoyantBoussinesqPimpleFoam`

---

## Equations

Without buoyancy, the standard `kEpsilon` equations are:

**k equation:**
```
d(rho*k)/dt + div(rho*U*k) - div(rho*nu_eff*grad(k)) = rho*G - rho*epsilon
```

**epsilon equation:**
```
d(rho*eps)/dt + div(rho*U*eps) - div(rho*nu_eff*grad(eps))
    = C1*rho*(eps/k)*G - C2*rho*(eps^2/k)
```

With `buoyancyTurbSource` active, the following terms are **added**:

**k equation gets:**
```
+ Gb          where Gb = beta * (nu_t/Prt) * (g · grad(T))
```

**epsilon equation gets:**
```
+ C3 * (epsilon/k) * Gb
```

`Gb > 0` → buoyancy produces turbulence (e.g. hot jet rising — destabilising)
`Gb < 0` → buoyancy suppresses turbulence (e.g. stable stratification)

---

## Step-by-Step Setup

### Step 1 — Confirm your solver is compatible

Check your `system/controlDict`:

```c++
application     buoyantSimpleFoam;   // ✅ compatible
// application  simpleFoam;          // ❌ not compatible — no T equation
```

### Step 2 — Check `constant/g` exists

```c++
// constant/g
FoamFile
{
    version     2.0;
    format      ascii;
    class       uniformDimensionedVectorField;
    object      g;
}

dimensions      [0 1 -2 0 0 0 0];
value           (0 0 -9.81);        // gravity pointing in -z direction
```

If it doesn't exist, create it. Without `g`, the buoyancy source term returns zero.

### Step 3 — Set up `constant/transportProperties`

The `buoyancyTurbSource` reads these automatically:

```c++
// constant/transportProperties
FoamFile
{
    version     2.0;
    format      ascii;
    class       dictionary;
    object      transportProperties;
}

transportModel  Newtonian;
nu              1e-06;          // kinematic viscosity [m^2/s]

beta            3e-04;          // thermal expansion coefficient [1/K]
                                // = 1/T for ideal gas, e.g. 1/293 ≈ 3.4e-3
TRef            293.15;         // reference temperature [K]
Pr              0.7;            // laminar Prandtl number
Prt             0.85;           // turbulent Prandtl number (used in Gb)
```

> **Tip:** `beta` and `TRef` define the Boussinesq buoyancy. For water at 20°C, use `beta = 2.07e-4`. For air at 20°C, use `beta = 3.43e-3`.

### Step 4 — Create `system/fvOptions`

This is the key file. Create it if it doesn't exist:

```c++
// system/fvOptions
FoamFile
{
    version     2.0;
    format      ascii;
    class       dictionary;
    object      fvOptions;
}
// ---------------------------------------------------------------

buoyancyTurbulenceSource        // <-- name can be anything
{
    type            buoyancyTurbSource;     // exact type name
    active          yes;

    selectionMode   all;                    // apply to entire domain
}
```

**Applying to a specific region only (optional):**

```c++
buoyancyTurbulenceSource
{
    type            buoyancyTurbSource;
    active          yes;

    selectionMode   cellZone;
    cellZone        fluidZone;      // name of your cellZone
}
```

### Step 5 — Set C3 in `constant/momentumTransport`

```c++
// constant/momentumTransport
FoamFile
{
    version     2.0;
    format      ascii;
    class       dictionary;
    object      momentumTransport;
}

simulationType  RAS;

RAS
{
    model       kEpsilon;
    turbulence  on;
    printCoeffs on;         // prints active coefficients at startup

    kEpsilonCoeffs
    {
        Cmu         0.09;
        C1          1.44;
        C2          1.92;
        C3          0.85;   // buoyancy effect on epsilon equation
                            // 0.0 = buoyancy only in k, not epsilon
                            // 1.0 = full buoyancy effect on epsilon
        sigmak      1.0;
        sigmaEps    1.3;
    }
}
```

**C3 guidance for buoyant jets:**

| Jet orientation          | Recommended C3 |
|--------------------------|----------------|
| Vertical (rising/sinking) | 1.0           |
| Horizontal               | 0.0            |
| Inclined                 | 0.5 – 0.85     |

> **Note:** `C3` in standard `kEpsilon` without `buoyancyTurbSource` controls the **velocity dilatation** term which is zero in incompressible flow — it has no effect. It only becomes the buoyancy C3 when `buoyancyTurbSource` is active.

---

## Verification

### At startup — check the log

When the case starts, you should see:

```
Selecting finite volume options type buoyancyTurbSource
    Source: buoyancyTurbulenceSource
    - selecting all cells
    - selected XXXXXX cell(s) with volume XXXXXX
    Applying buoyancyTurbSource to: epsilon and k
```

And the coefficient printout (requires `printCoeffs on`):

```
kEpsilonCoeffs
{
    Cmu         0.09;
    C1          1.44;
    C2          1.92;
    C3          0.85;   // <-- confirm your value is active
    sigmak      1;
    sigmaEps    1.3;
}
```

### Quick grep commands

```bash
# Confirm buoyancyTurbSource loaded
grep -i "buoyancy\|turbSource" log.*

# Confirm C3 value
grep "C3" log.*

# Check k and epsilon are being solved
grep "Solving for k\|Solving for epsilon" log.* | head -10
```

---

## Common Mistakes

| Mistake | Symptom | Fix |
|---------|---------|-----|
| Using `buoyantKEpsilon` with incompressible solver | "Unknown RAS model" error | Use `kEpsilon` + `buoyancyTurbSource` instead |
| Wrong library loaded | "cannot open shared object" warning | Check `ls $FOAM_LIBBIN \| grep compressible` |
| `beta` or `TRef` missing | Gb = 0, no effect | Add to `transportProperties` |
| `constant/g` missing | Gb = 0, no effect | Create `constant/g` with gravity vector |
| C3 set in `transportProperties` | No error, silently ignored | C3 belongs in `momentumTransport` under `kEpsilonCoeffs` |
| `fvOptions` file not in `system/` | Source never loaded | Confirm path is `system/fvOptions` |

---

## Physical Significance for Buoyant Jets

Without buoyancy in turbulence, standard `kEpsilon` misses extra turbulence production from
temperature gradients. This leads to:

- Underestimated jet spreading
- Underestimated dilution by **20–40%** in the near field
- Incorrect transition from momentum-dominated to buoyancy-dominated regime

With `buoyancyTurbSource` active, the model captures:

- Extra k production in regions of strong temperature gradient
- Modified dissipation rate controlled by C3
- More realistic dilution and mixing predictions

---

## Summary Checklist

- [ ] Solver is buoyant-compatible (`buoyantSimpleFoam`, `buoyantPimpleFoam`, etc.)
- [ ] `constant/g` exists with correct gravity vector
- [ ] `beta`, `TRef`, `Prt` defined in `constant/transportProperties`
- [ ] `system/fvOptions` created with `buoyancyTurbSource`
- [ ] `kEpsilonCoeffs { C3 <value>; }` set in `constant/momentumTransport`
- [ ] Log confirms: `Applying buoyancyTurbSource to: epsilon and k`
- [ ] Log confirms correct `C3` value in coefficient printout
