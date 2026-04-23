# mathcore-constants — Specification

**Version:** draft-1 (2026-04-22)
**Status:** draft, pre-freeze
**Target release:** v0.1.0
**Crate kind:** data-only (types + catalog; no algebra, no computation)

This document specifies the authoritative contract for `mathcore-constants`.
It depends on, and must be read alongside, `mathcore-units/SPECIFICATION.md`.
Every `ConstantId` variant defined in mathcore-units § 4 has exactly one
`ConstantSpec` entry here. Types, field names, and wire format described in
this document are frozen for the v0.x series once accepted. Additions
(new ConstantId variants and their matching ConstantSpec entries) are
permitted in minor versions; removals or renames require a major version
bump coordinated with mathcore-units and thales.

---

## 1. Scope and non-goals

### 1.1 In scope

- The `ConstantSpec` static catalog entry — per-constant value, unit,
  uncertainty, citation, domain, and display metadata.
- The `Citation` type and `CitationSource` enum — authoritative source
  attribution for every constant.
- The `Uncertainty` enum — exact, standard, or relative.
- The `Domain` enum — coarse categorization for filtering.
- The complete v0.1.0 catalog: one `ConstantSpec` per `ConstantId` variant.
- Public lookup API: by id, all, by domain.
- Serde wire format (opt-in).
- `no_std + alloc` support.

### 1.2 Out of scope

- Uncertainty propagation through arithmetic — that is a consumer
  responsibility (thales, unitalg, or mathlex-eval).
- Unit conversion or dimension checking — lives in unitalg.
- Arc<Expr> interop — mathcore-constants does not touch thales internals.
- The `ConstantId` enum definition itself — lives in mathcore-units § 4
  so that `Conversion::FromConstant` in mathcore-units can reference
  constants without holding values.
- Currency, empirical material properties, or time-varying quantities.
- Planck units (Planck length, Planck time, Planck mass) — deferred to
  a future catalog addition, not v0.1.0 scope.

---

## 2. Types

### 2.1 `ConstantSpec` — catalog entry

```rust
#[derive(Debug, Clone)]
pub struct ConstantSpec {
    /// Stable identifier, re-exported from mathcore-units.
    pub id: ConstantId,
    /// Numeric or symbolic value in canonical SI units.
    /// Uses mathlex::Expression to allow exact symbolic forms (π, ℏ = h/2π)
    /// and high-precision numeric literals side by side.
    pub value: Expression,
    /// Canonical SI unit for `value`, expressed as a `UnitExpression` from
    /// mathcore-units § 2.8. Using `UnitExpression` (not a bare `UnitId`)
    /// is required because many constants have compound units that cannot
    /// be expressed as a single `UnitId` (e.g. G: m³·kg⁻¹·s⁻², k_B: J·K⁻¹,
    /// N_A: mol⁻¹, σ: W·m⁻²·K⁻⁴). For simple single-unit constants the
    /// UnitExpression degenerates to `UnitExpression::Atom { id, prefix: None }`.
    /// See § 3 for the value/unit contract and mathcore-units § 4
    /// "ConstantSpec unit-field contract" for the authoritative rationale.
    pub unit: UnitExpression,
    /// None for exactly-defined constants; Some for measured values.
    pub uncertainty: Option<Uncertainty>,
    /// Authoritative citation. Every entry must have one (MC-11).
    pub citation: Citation,
    /// Coarse domain for filtering via `constants_by_domain`.
    pub domain: Domain,
    /// Conventional symbol used in equations. e.g. "c", "ℏ", "N_A", "α".
    pub display_symbol: &'static str,
    /// Full conventional English name. e.g. "speed of light in vacuum".
    pub display_name: &'static str,
}
```

`ConstantSpec` is not itself serde-capable as a static entry (like
`mathcore_units::UnitSpec`). Serde coverage is provided by `ConstantSpecWire`
for serialization contexts where data must cross a boundary (see § 10).

### 2.2 `Citation`

```rust
#[derive(Debug, Clone)]
#[cfg_attr(feature = "serde", derive(serde::Serialize, serde::Deserialize))]
#[cfg_attr(feature = "serde", serde(tag = "kind", content = "value"))]
pub struct Citation {
    /// Which authoritative body issued the value.
    pub source: CitationSource,
    /// Specific citation detail: DOI, URL, section reference, or textbook.
    pub reference: &'static str,
    /// Year the value was retrieved or confirmed against the source.
    pub retrieval_year: u16,
}
```

### 2.3 `CitationSource`

```rust
#[derive(Debug, Clone, Copy, PartialEq, Eq, Hash)]
#[cfg_attr(feature = "serde", derive(serde::Serialize, serde::Deserialize))]
pub enum CitationSource {
    /// CODATA 2022 recommended values of the fundamental physical constants.
    /// Primary source for physics constants.
    Codata2022,
    /// IAU 2015 nominal solar and planetary values (IAU B3 resolution).
    Iau2015,
    /// IUPAC 2021 atomic weights (for atomic masses where relevant).
    Iupac2021,
    /// Fixed by SI 2019 redefinition or equivalent formal definition with
    /// zero uncertainty by construction (exact integer or exact rational).
    DefinedExact,
    /// Exact mathematical constant (π, e, γ, φ) whose value is established
    /// by mathematical proof, not physical measurement.
    Mathematical,
    /// Derived from other defined-exact or CODATA constants by an exact
    /// algebraic relation (e.g. R = N_A · k_B).
    DerivedExact,
    /// Placeholder for future extensibility (user-supplied constants).
    UserProvided,
}
```

### 2.4 `Uncertainty`

```rust
#[derive(Debug, Clone)]
#[cfg_attr(feature = "serde", derive(serde::Serialize, serde::Deserialize))]
#[cfg_attr(feature = "serde", serde(tag = "kind", content = "value"))]
pub enum Uncertainty {
    /// Value is exact by definition — no measurement uncertainty.
    /// Used for SI-2019 defined-exact constants, derived-exact constants,
    /// and mathematical constants.
    ExactByDefinition,
    /// Standard uncertainty (1-sigma), expressed in the same units as
    /// `ConstantSpec::value`. Consumer chooses confidence interval.
    Standard(Expression),
    /// Relative standard uncertainty as a dimensionless fraction.
    /// For constants where relative precision is more natural to cite
    /// (fine-structure constant, gravitational constant).
    Relative(Expression),
}
```

Uncertainty is metadata only. This crate never propagates uncertainty
through arithmetic. A consumer wanting uncertainty propagation (e.g., a
Monte-Carlo evaluator in thales) reads the `uncertainty` field and manages
propagation itself.

### 2.5 `Domain`

```rust
#[derive(Debug, Clone, Copy, PartialEq, Eq, Hash)]
#[cfg_attr(feature = "serde", derive(serde::Serialize, serde::Deserialize))]
pub enum Domain {
    /// Fundamental physics: c, h, ℏ, e, k_B, N_A, G, α, ε₀, μ₀.
    FundamentalPhysics,
    /// Particle and atomic-scale: electron mass, proton mass, Bohr radius, etc.
    Atomic,
    /// Nuclear: neutron mass, atomic mass unit, Compton wavelength, nuclear magneton.
    Nuclear,
    /// Astronomy: solar/earth/planetary masses, radii, luminosities, AU, ly, parsec.
    Astronomy,
    /// Chemistry: molar gas constant, Faraday, Rydberg, Stefan-Boltzmann.
    Chemistry,
    /// Pure mathematics: π, e (Euler's number), Euler–Mascheroni γ, golden ratio φ.
    Mathematics,
}
```

Domain is a coarse filter only, not a formal classification. A constant
like `StefanBoltzmannConstant` sits in `Chemistry` (it bridges
thermodynamics and radiation physics) while `ElectronMass` sits in
`Atomic` even though it is a fundamental particle. Placement decisions are
documented per constant in § 6.

---

## 3. Value representation

### 3.1 Why `mathlex::Expression`

Every `ConstantSpec::value` is a `mathlex::Expression`. This choice serves
three distinct cases:

1. **Defined-exact integers and rationals.** `SpeedOfLight` is exactly
   299 792 458 m/s; `ElementaryCharge` is exactly 1.602176634 × 10⁻¹⁹ C.
   These store as `Expression::Integer` or `Expression::Float` without
   rounding error concerns (the SI definition fixes the digits).

2. **Symbolic derivations.** `ReducedPlanckConstant` is h/(2π). Storing
   `Expression::BinaryOp(Div, PlanckConstant, BinaryOp(Mul, 2, Pi))` (or
   equivalent mathlex form) lets thales evaluate it symbolically and lets
   the value remain exact without numeric truncation of π.

3. **Mathematical constants with irrational values.** `Pi` and
   `EulerMascheroni` cannot be fully represented as floating-point
   without rounding. Storing them as `Expression::Constant(MathConst::Pi)`
   (or equivalent mathlex symbolic atom) lets symbolic CAS passes through
   thales propagate them exactly.

A plain `f64` field would lose exactness for defined-exact constants and
lose symbolic form for derived or irrational ones. `Expression` covers all
three cases in a single type that both mathlex and thales already understand.

### 3.2 Value/unit contract

`ConstantSpec::value` is always expressed in the canonical SI unit
identified by `ConstantSpec::unit`. The conversion path is:

```
user quantity (arbitrary unit)
  → unitalg converts to canonical SI
  → numerical comparison with ConstantSpec::value is valid
```

No "convenience values in non-SI units" are stored. If a consumer wants
the gravitational constant in CGS, it converts via unitalg.

### 3.3 Numeric precision for measured constants

Measured constants store as many digits as the authoritative source
provides. For CODATA 2022 values this is typically 8–12 significant
digits. Values are written as `Expression::Float` with the source's
stated digits; the trailing digits beyond `f64` precision are noted in
a comment in the catalog source but the stored `Expression` carries the
full stated precision using mathlex's arbitrary-precision float
representation where available, falling back to the source's given
significant figures.

---

## 4. Exact vs. measured constants

### 4.1 SI-2019 defined-exact constants

The 2019 SI redefinition fixed seven constants to exact values by
definition. These carry `Uncertainty::ExactByDefinition` and
`CitationSource::DefinedExact`:

| Constant | Exact value |
|---|---|
| SpeedOfLight | 299 792 458 m/s |
| PlanckConstant | 6.626 070 15 × 10⁻³⁴ J·s |
| ElementaryCharge | 1.602 176 634 × 10⁻¹⁹ C |
| BoltzmannConstant | 1.380 649 × 10⁻²³ J/K |
| AvogadroNumber | 6.022 140 76 × 10²³ mol⁻¹ |

(`ReducedPlanckConstant`, `MolarGasConstant`, `FaradayConstant`, and
`StefanBoltzmannConstant` are derived-exact from these five.)

### 4.2 CODATA-exact vs. CODATA measured

"CODATA-exact" in this spec means a constant whose uncertainty is zero
because SI 2019 fixed it. "CODATA measured" means CODATA 2022 provides
a best-fit value with a stated uncertainty, and that uncertainty is
non-zero. The distinction is encoded in `CitationSource` and
`Uncertainty` together:

- `DefinedExact` + `ExactByDefinition` → fixed by SI
- `DerivedExact` + `ExactByDefinition` → exact algebraic combination of fixed values
- `Codata2022` + `ExactByDefinition` → no uncertainty by CODATA convention (unusual; used only where the standard says "exact" without SI definition, e.g., the IAU 2012 exact AU)
- `Codata2022` + `Standard(x)` → measured with stated uncertainty

### 4.3 Astronomical nominal values

IAU 2015 nominal values (solar mass, Earth mass, solar radius, etc.)
are not "measured" in the same sense as CODATA values: they are adopted
reference values that the IAU fixes by resolution for consistency across
models. They carry `CitationSource::Iau2015` and `Uncertainty::Standard`
with the IAU's stated uncertainty where published, or `ExactByDefinition`
where the IAU explicitly adopts the value as exact.

### 4.4 Mathematical constants

`Pi`, `E`, `EulerMascheroni`, and `GoldenRatio` are stored symbolically
(as mathlex symbolic atoms, not truncated floats). They carry
`CitationSource::Mathematical` and `Uncertainty::ExactByDefinition`.
A consumer requesting a numeric approximation calls mathlex-eval's
numeric evaluator, which provides IEEE 754 double precision or
arbitrary precision depending on the caller's context.

### 4.5 VacuumPermeability: the post-2019 shift

Before 2019, μ₀ = 4π × 10⁻⁷ H/m exactly (a consequence of the
definition of the ampere). The 2019 SI redefinition fixed the elementary
charge instead, making μ₀ a measured quantity with a tiny relative
uncertainty (≈ 1.5 × 10⁻¹⁰). This crate stores μ₀ as a measured CODATA
2022 value with its stated uncertainty, not as a defined-exact constant.
The catalog comment and citation note this historical change explicitly.

---

## 5. Catalog organization and module layout

### 5.1 Module structure

```
mathcore-constants/
├── Cargo.toml
├── LICENSE
├── README.md
├── CLAUDE.md
├── SPECIFICATION.md                    # this file
├── src/
│   ├── lib.rs                          # public re-exports + crate docs
│   ├── types.rs                        # ConstantSpec, Citation, CitationSource,
│   │                                   #   Uncertainty, Domain
│   ├── lookup.rs                       # lookup_constant, all_constants,
│   │                                   #   constants_by_domain
│   ├── catalog/
│   │   ├── mod.rs                      # master static slice + dispatch
│   │   ├── fundamental_exact.rs        # SpeedOfLight, PlanckConstant, ℏ,
│   │   │                               #   ElementaryCharge, BoltzmannConstant,
│   │   │                               #   AvogadroNumber
│   │   ├── fundamental_measured.rs     # G, α, ε₀, μ₀, ElectronChargeToMass
│   │   ├── particle_masses.rs          # ElectronMass, ProtonMass, NeutronMass
│   │   ├── astronomical.rs             # SolarMass, EarthMass, JupiterMass,
│   │   │                               #   LunarMass, SolarRadius, EarthRadius,
│   │   │                               #   SolarLuminosity, AU, LightYear, Parsec
│   │   ├── mathematical.rs             # Pi, E, EulerMascheroni, GoldenRatio
│   │   ├── chemical.rs                 # R, F, RydbergConstant,
│   │   │                               #   StefanBoltzmannConstant
│   │   └── nuclear_atomic.rs           # AtomicMassUnit, BohrRadius,
│   │                                   #   BohrMagneton, NuclearMagneton,
│   │                                   #   ComptonWavelength
│   └── wire.rs                         # ConstantSpecWire + serde impls
└── tests/
    ├── catalog_completeness.rs         # every ConstantId has a ConstantSpec
    ├── unit_coherence.rs               # ConstantSpec::unit matches expected dimension
    ├── citation_discipline.rs          # no empty citations, retrieval_year present
    ├── exact_serde_roundtrip.rs        # defined-exact values round-trip losslessly
    └── domain_coverage.rs              # every Domain has at least one entry
```

### 5.2 Static catalog

The catalog is a static `&'static [ConstantSpec]`. The master slice in
`catalog/mod.rs` concatenates the per-domain sub-slices. Lookup is O(1)
via a match on `ConstantId` (the compiler generates an efficient dispatch
for small enums; no runtime hash table needed).

```rust
pub fn lookup_constant(id: ConstantId) -> &'static ConstantSpec;
pub fn all_constants() -> &'static [ConstantSpec];
pub fn constants_by_domain(domain: Domain)
    -> impl Iterator<Item = &'static ConstantSpec>;
```

The `lookup_constant` function panics in debug builds and returns a
defined fallback in release builds if called with a `ConstantId` for
which no entry exists — this state is prevented by the CI completeness
test (see § 11).

---

## 6. Catalog entries

### 6.1 Fundamental physical — defined-exact (SI 2019)

All entries: `domain: FundamentalPhysics`, `uncertainty: ExactByDefinition`,
`citation.source: DefinedExact`, `citation.retrieval_year: 2024`.
Reference for all: SI Brochure 9th edition, Table 1 (exact defining constants).

| ConstantId | display_symbol | value | unit (UnitExpression) |
|---|---|---|---|
| SpeedOfLight | c | 299 792 458 | `Binary(Div, Atom(Meter), Atom(Second))` — m·s⁻¹ |
| PlanckConstant | h | 6.626 070 15 × 10⁻³⁴ | `Binary(Mul, Atom(Joule), Atom(Second))` — J·s |
| ReducedPlanckConstant | ℏ | h / (2 · π) (symbolic) | `Binary(Mul, Atom(Joule), Atom(Second))` — J·s |
| ElementaryCharge | e | 1.602 176 634 × 10⁻¹⁹ | `Atom(Coulomb)` — C |
| BoltzmannConstant | k_B | 1.380 649 × 10⁻²³ | `Binary(Div, Atom(Joule), Atom(Kelvin))` — J·K⁻¹ |
| AvogadroNumber | N_A | 6.022 140 76 × 10²³ | `Unary(Inv, Atom(Mole))` — mol⁻¹ |

**Note on `ReducedPlanckConstant`:** `value` stores the symbolic
`Expression` for h/(2π), not a truncated float. The `unit` is identical to
`PlanckConstant` (Joule·second). A numeric evaluator resolves this to
≈ 1.054 571 817 × 10⁻³⁴ J·s. Storing symbolically keeps the exact relation
visible to the CAS.

**Note on `AvogadroNumber` unit:** mol⁻¹ is expressed as a
`UnitExpression::Unary { op: UnaryOp::Inv, child: Atom { id: Mole, prefix: None } }`.
There is no standalone `PerMole` UnitId in mathcore-units; the compound
expression is the canonical representation. The `ConstantSpec::unit` field
is typed as `UnitExpression`, which handles this directly. See § 3.2.

### 6.2 Fundamental physical — measured (CODATA 2022)

All entries: `domain: FundamentalPhysics`, `citation.source: Codata2022`,
`citation.reference: "https://physics.nist.gov/cuu/Constants/ (CODATA 2022)"`,
`citation.retrieval_year: 2024`.

| ConstantId | display_symbol | value | unit (UnitExpression) | uncertainty (relative) |
|---|---|---|---|---|
| GravitationalConstant | G | 6.674 30 × 10⁻¹¹ | `Binary(Div, Binary(Pow, Atom(Meter), Literal(3)), Binary(Mul, Atom(Kilogram), Binary(Pow, Atom(Second), Literal(2))))` — m³·kg⁻¹·s⁻² | 2.2 × 10⁻⁵ |
| FineStructureConstant | α | 7.297 352 5693 × 10⁻³ | `Atom(Count)` — dimensionless | 1.5 × 10⁻¹⁰ |
| VacuumPermittivity | ε₀ | 8.854 187 8188 × 10⁻¹² | `Binary(Div, Atom(Farad), Atom(Meter))` — F·m⁻¹ | 1.7 × 10⁻¹⁰ |
| VacuumPermeability | μ₀ | 1.256 637 0621 × 10⁻⁶ | `Binary(Div, Atom(Henry), Atom(Meter))` — H·m⁻¹ | 1.5 × 10⁻¹⁰ |
| ElectronChargeToMass | e/m_e | 1.758 820 008 × 10¹¹ | `Binary(Div, Atom(Coulomb), Atom(Kilogram))` — C·kg⁻¹ | 3.0 × 10⁻¹⁰ |

**Uncertainty encoding for `GravitationalConstant`:** stored as
`Uncertainty::Relative(Expression::Float(2.2e-5))` per CODATA's convention
of listing relative standard uncertainty u_r.

**VacuumPermittivity:** ε₀ is now a measured quantity (like μ₀) after the
2019 SI redefinition. Before 2019, ε₀ = 1/(μ₀ c²) exactly; now both ε₀
and μ₀ carry small measurement uncertainties propagated from the measured
fine-structure constant α.

**FineStructureConstant** has `unit: Atom(Count)` (dimensionless). The
`domain: FundamentalPhysics` despite its dimensionless character because
it governs the strength of the electromagnetic interaction.

**Compound units in this section:** `GravitationalConstant` (m³·kg⁻¹·s⁻²)
and `ElectronChargeToMass` (C·kg⁻¹) use `UnitExpression` trees as shown in
the table above. `ConstantSpec::unit` is typed as `UnitExpression`, which
handles these directly without any workaround (see § 2.1 and
mathcore-units § 4 "ConstantSpec unit-field contract").

### 6.3 Particle masses

All entries: `domain: Atomic`, `citation.source: Codata2022`,
same reference URL as § 6.2.

| ConstantId | display_symbol | value (kg) | unit (UnitExpression) | uncertainty (relative) |
|---|---|---|---|---|
| ElectronMass | m_e | 9.109 383 7139 × 10⁻³¹ | `Atom(Kilogram)` | 1.8 × 10⁻¹⁰ |
| ProtonMass | m_p | 1.672 621 925 × 10⁻²⁷ | `Atom(Kilogram)` | 5.1 × 10⁻¹⁰ |
| NeutronMass | m_n | 1.674 927 500 × 10⁻²⁷ | `Atom(Kilogram)` | 5.7 × 10⁻¹⁰ |

Uncertainties are stored as `Uncertainty::Relative(Expression::Float(x))`.

These three constants are also referenced by mathcore-units § 5.7 via
`Conversion::FromConstant` for the units `ElectronMass`, `ProtonMass`, and
`NeutronMass`. The `ConstantId` enum shares a name with the `UnitId` enum
for these entries; they live in separate namespaces (see mathcore-units § 4
for the ConstantId definitions and § 5.7 for the UnitId entries).

### 6.4 Astronomical

All entries: `domain: Astronomy`, `citation.source: Iau2015`,
`citation.reference: "IAU 2015 Resolution B3 nominal solar and planetary values"`,
`citation.retrieval_year: 2024`.

Exceptions noted per entry.

| ConstantId | display_symbol | value | unit (UnitExpression) | uncertainty |
|---|---|---|---|---|
| SolarMass | M_⊙ | 1.988 416 × 10³⁰ | `Atom(Kilogram)` | Standard: 3 × 10²⁶ kg (IAU) |
| EarthMass | M_⊕ | 5.972 168 × 10²⁴ | `Atom(Kilogram)` | Standard: 6 × 10²⁰ kg (IAU) |
| JupiterMass | M_J | 1.898 125 × 10²⁷ | `Atom(Kilogram)` | Standard: 9 × 10²³ kg (IAU) |
| LunarMass | M_☾ | 7.342 × 10²² | `Atom(Kilogram)` | Standard: 1 × 10¹⁸ kg (IAU) |
| SolarRadius | R_⊙ | 6.957 × 10⁸ | `Atom(Meter)` | ExactByDefinition (IAU nominal) |
| EarthRadius | R_⊕ | 6.371 × 10⁶ | `Atom(Meter)` | ExactByDefinition (IAU nominal) |
| SolarLuminosity | L_⊙ | 3.828 × 10²⁶ | `Atom(Watt)` | ExactByDefinition (IAU nominal) |
| AstronomicalUnit | au | 1.495 978 707 × 10¹¹ | `Atom(Meter)` | ExactByDefinition |
| LightYear | ly | 9.460 730 472 580 800 × 10¹⁵ | `Atom(Meter)` | ExactByDefinition |
| Parsec | pc | 3.085 677 581 491 × 10¹⁶ | `Atom(Meter)` | ExactByDefinition |

**AstronomicalUnit:** IAU 2012 resolution B2 fixes au = 149 597 870 700 m
exactly. `citation.source: DefinedExact`; IAU 2012 citation.

**LightYear:** exact by construction as c · Julian year (365.25 × 86 400 s).
The Julian-year value (31 557 600 s) is a conventional exact definition.
`citation.source: DefinedExact`.

**Parsec:** derived from the AU as 648 000 · au / π meters. Stored
symbolically as `Expression::BinaryOp(Mul, 648000·au, Inv(Pi))` or
numerically with the digits listed; the symbolic form is preferred.
`citation.source: DerivedExact` (exact given exact AU and symbolic π).

**IAU nominal vs. measured:** `SolarRadius`, `EarthRadius`, and
`SolarLuminosity` are IAU-adopted nominal values (exact by IAU resolution);
they carry `Uncertainty::ExactByDefinition`. `SolarMass`, `EarthMass`,
`JupiterMass`, and `LunarMass` are measured masses with IAU-stated
uncertainties; they carry `Uncertainty::Standard(x)`.

These constants back the mathcore-units § 5.7 constants-as-units (e.g.,
`UnitId::SolarMass` has `Conversion::FromConstant { id: ConstantId::SolarMass }`).

### 6.5 Mathematical

All entries: `domain: Mathematics`, `citation.source: Mathematical`,
`uncertainty: ExactByDefinition`.

| ConstantId | display_symbol | display_name | value |
|---|---|---|---|
| Pi | π | Archimedes' constant | `Expression::Constant(MathConst::Pi)` |
| E | e | Euler's number | `Expression::Constant(MathConst::E)` |
| EulerMascheroni | γ | Euler–Mascheroni constant | `Expression::Constant(MathConst::EulerGamma)` |
| GoldenRatio | φ | golden ratio | `Expression::Constant(MathConst::Phi)` |

All four have `unit: Atom(Count)` (dimensionless). `citation.reference` for
each points to the standard mathematical definition (e.g., standard references
for π, e, γ, φ) rather than a measured source.

**Numeric approximations** (for documentation only; the stored value is
symbolic):
- π ≈ 3.141 592 653 589 793
- e ≈ 2.718 281 828 459 045
- γ ≈ 0.577 215 664 901 532
- φ = (1 + √5) / 2 ≈ 1.618 033 988 749 895

**v0.1.0 design decision for γ and φ** (see also § 16, MC-FLAG-2): whether
mathlex exposes `MathConst::EulerGamma` and `MathConst::Phi` symbolic atoms
is a mathlex-side dependency not yet confirmed. The pragmatic approach for
v0.1.0 is:
- `GoldenRatio` (φ): stored as `Expression::BinaryOp(Div, BinaryOp(Add,
  Integer(1), UnaryOp(Sqrt, Integer(5))), Integer(2))` — i.e. (1 + √5)/2 —
  because √5 is expressible in mathlex without a dedicated atom. This keeps
  the algebraic form exact and symbolic.
- `EulerMascheroni` (γ): stored as `Expression::Float` with the highest
  precision available (≥ 50 significant digits from a mathematical reference),
  with a catalog comment noting the limitation. If mathlex later exposes
  `MathConst::EulerGamma`, a minor-version update replaces the float with
  the symbolic atom.
This is a pragmatic symbolic/numeric mix determined by what mathlex supports
today. The mc-12 requirement (symbolic storage for mathematical constants) is
met fully for π, e, and φ; partially met for γ pending mathlex atom support.

### 6.6 Chemical

All entries: `domain: Chemistry`.

| ConstantId | display_symbol | display_name | value | unit (UnitExpression) | uncertainty | source |
|---|---|---|---|---|---|---|
| MolarGasConstant | R | molar gas constant | N_A · k_B (symbolic) | `Binary(Div, Atom(Joule), Binary(Mul, Atom(Mole), Atom(Kelvin)))` — J·mol⁻¹·K⁻¹ | ExactByDefinition | DerivedExact |
| FaradayConstant | F | Faraday constant | N_A · e (symbolic) | `Binary(Div, Atom(Coulomb), Atom(Mole))` — C·mol⁻¹ | ExactByDefinition | DerivedExact |
| RydbergConstant | R_∞ | Rydberg constant | 1.097 373 156 816 × 10⁷ | `Unary(Inv, Atom(Meter))` — m⁻¹ | Relative: 1.9 × 10⁻¹² | Codata2022 |
| StefanBoltzmannConstant | σ | Stefan–Boltzmann constant | (2π⁵ k_B⁴) / (15 h³ c²) (symbolic) | `Binary(Div, Atom(Watt), Binary(Mul, Binary(Pow, Atom(Meter), Literal(2)), Binary(Pow, Atom(Kelvin), Literal(4))))` — W·m⁻²·K⁻⁴ | ExactByDefinition | DerivedExact |

**MolarGasConstant:** R = N_A · k_B. Both N_A and k_B are SI-2019 defined-exact;
therefore R is derived-exact. Stored symbolically as `Expression` referring
to the product of those two constants. Numeric value: 8.314 462 618 J/(mol·K).

**FaradayConstant:** F = N_A · e. Both defined-exact; therefore F is
derived-exact. Numeric value: 96 485.332 12 C/mol.

**StefanBoltzmannConstant:** σ = 2π⁵ k_B⁴ / (15 h³ c²). All input constants
are defined-exact; therefore σ is derived-exact. The symbolic form is
preferred for CAS fidelity. Numeric value: 5.670 374 419 × 10⁻⁸ W/(m²·K⁴).

**RydbergConstant:** a measured value (not exact). CODATA 2022 relative
uncertainty ≈ 1.9 × 10⁻¹². Unit is m⁻¹, expressed as
`UnitExpression::Unary { op: UnaryOp::Inv, child: Atom { id: Meter, prefix: None } }`.

### 6.7 Nuclear and Atomic

All entries: `domain: Nuclear` (for AtomicMassUnit, ComptonWavelength,
NuclearMagneton) or `domain: Atomic` (for BohrRadius, BohrMagneton).
`citation.source: Codata2022` for all; same NIST reference URL.

| ConstantId | display_symbol | display_name | value | unit (UnitExpression) | uncertainty (relative) |
|---|---|---|---|---|---|
| AtomicMassUnit | u | unified atomic mass unit | 1.660 539 068 92 × 10⁻²⁷ | `Atom(Kilogram)` | 3.0 × 10⁻¹⁰ |
| BohrRadius | a_0 | Bohr radius | 5.291 772 105 44 × 10⁻¹¹ | `Atom(Meter)` | 1.5 × 10⁻¹⁰ |
| BohrMagneton | μ_B | Bohr magneton | 9.274 010 0783 × 10⁻²⁴ | `Binary(Div, Atom(Joule), Atom(Tesla))` — J·T⁻¹ | 3.0 × 10⁻¹⁰ |
| NuclearMagneton | μ_N | nuclear magneton | 5.050 783 7461 × 10⁻²⁷ | `Binary(Div, Atom(Joule), Atom(Tesla))` — J·T⁻¹ | 3.1 × 10⁻¹⁰ |
| ComptonWavelength | λ_C | Compton wavelength | 2.426 310 238 67 × 10⁻¹² | `Atom(Meter)` | 1.5 × 10⁻¹⁰ |

**AtomicMassUnit:** also called the dalton (Da). Defined as 1/12 the mass of
a carbon-12 atom at rest; this is a measured value (the proton mass in atomic
mass units is not exactly 1). The `display_symbol` is `u`; alias `Da` is
registered in the catalog comment but display_symbol is `u` per IUPAC.
This constant backs `UnitId::AtomicMassUnit` via
`Conversion::FromConstant { id: ConstantId::AtomicMassUnit }` in
mathcore-units § 5.7.

**BohrMagneton and NuclearMagneton:** unit is J·T⁻¹, expressed as
`UnitExpression::Binary { op: Div, left: Atom(Joule), right: Atom(Tesla) }`.
Both `UnitId::Tesla` and `UnitId::Joule` are in mathcore-units; the
`ConstantSpec::unit` field typed as `UnitExpression` encodes this directly.

---

## 7. Source discipline

Every `ConstantSpec` must satisfy all of the following before the catalog
is frozen:

1. `citation.source` is a non-`UserProvided` variant.
2. `citation.reference` is non-empty and resolves to an accessible
   authoritative source (a NIST DOI, IAU resolution URL, standard
   mathematical reference, or SI Brochure section).
3. `citation.retrieval_year` is the year the value was last verified
   against the source.
4. For `Codata2022` and `Iau2015` entries, the number of significant
   digits in `value` matches the number of digits published by the source
   (no silent truncation).
5. The `uncertainty` variant matches the source's classification:
   - Source says "exact" or "by definition" → `ExactByDefinition`
   - Source gives a standard uncertainty in the same unit → `Standard`
   - Source gives a relative standard uncertainty u_r → `Relative`

The citation discipline test (§ 11) enforces rules 1–3 at compile time via
const assertions where possible and at test time otherwise.

When CODATA 2026 (or a later revision) is published, values may be updated:
this is a minor version bump (§ 13). The citation source adds a
`Codata2026` variant at that point (it does not replace `Codata2022` —
both variants remain so consumers can detect which edition a value came from).

---

## 8. Uncertainty model

### 8.1 Scope within this crate

Uncertainty is stored as metadata. This crate:
- Records whether a constant is exact, and if not, what the standard or
  relative uncertainty is.
- Does **not** propagate uncertainty through expressions.
- Does **not** implement interval arithmetic.
- Does **not** correlate uncertainties between constants.

### 8.2 Interpretation

`Uncertainty::Standard(x)` means: the value's 68% confidence interval is
(value − x, value + x), where x is in the same unit as `ConstantSpec::value`.
`Uncertainty::Relative(x)` means: the standard deviation equals x · value,
dimensionlessly. A consumer wanting a 95% interval multiplies the standard
uncertainty by a coverage factor (typically 2 for near-Gaussian distributions
with sufficient degrees of freedom).

### 8.3 Interaction with symbolic values

For `ReducedPlanckConstant`, `MolarGasConstant`, `FaradayConstant`,
`StefanBoltzmannConstant`, and `Parsec`, the value is stored symbolically and
the uncertainty is `ExactByDefinition`. A numeric evaluator resolves the
symbolic expression to a floating-point number; any rounding introduced at
that step is an evaluator artefact, not a physical uncertainty.

### 8.4 What consumers should do

A consumer performing error-propagation analysis should:
1. Read `ConstantSpec::uncertainty` for each constant used.
2. Evaluate `Uncertainty::Standard(expr)` using mathlex-eval to get a
   numeric σ.
3. Apply first-order or Monte Carlo uncertainty propagation in their own
   logic, outside this crate.

---

## 9. Public API

```rust
/// Returns the ConstantSpec for the given id.
/// Panics in debug builds if the id has no catalog entry.
/// All ConstantId variants are guaranteed to have entries in v0.1.0 (MC-1).
pub fn lookup_constant(id: ConstantId) -> &'static ConstantSpec;

/// Returns all catalog entries as a static slice.
/// Order is unspecified; do not rely on index position.
pub fn all_constants() -> &'static [ConstantSpec];

/// Returns an iterator over entries belonging to the given domain.
/// Borrows from the static catalog — zero allocation.
pub fn constants_by_domain(domain: Domain)
    -> impl Iterator<Item = &'static ConstantSpec>;
```

Re-exports from mathcore-units so callers need only one import for
constants work:

```rust
pub use mathcore_units::{ConstantId, UnitId, UnitExpression};
```

The `Domain`, `Uncertainty`, `Citation`, `CitationSource`, and `ConstantSpec`
types are defined in this crate and publicly exported from the crate root.

---

## 10. Wire format

### 10.1 Feature flag

Serde support is opt-in:

```toml
[features]
default = ["std"]
std = ["mathcore-units/std"]
serde = ["dep:serde", "mathcore-units/serde"]
```

Enabling `serde` without also enabling `mathcore-units/serde` is a compile
error (mathcore-units types must be serializable for ConstantSpec wire to
round-trip).

### 10.2 `ConstantSpecWire`

Because `ConstantSpec` holds `&'static str` fields and a static catalog entry
is not owned, a separate `ConstantSpecWire` type owns the serialized form:

```rust
#[cfg(feature = "serde")]
#[derive(Debug, Clone, serde::Serialize, serde::Deserialize)]
#[serde(tag = "kind", content = "value")]
pub struct ConstantSpecWire {
    pub id: ConstantId,
    pub value: Expression,
    pub unit: UnitExpression,   // compound-unit capable; matches ConstantSpec::unit
    pub uncertainty: Option<Uncertainty>,
    pub citation: Citation,
    pub domain: Domain,
    pub display_symbol: String,
    pub display_name: String,
}

impl From<&'static ConstantSpec> for ConstantSpecWire { ... }
```

### 10.3 JSON example

```json
{
  "kind": "ConstantSpec",
  "value": {
    "id": "SpeedOfLight",
    "value": { "kind": "Integer", "value": "299792458" },
    "unit": "Meter",
    "uncertainty": { "kind": "ExactByDefinition" },
    "citation": {
      "source": { "kind": "DefinedExact" },
      "reference": "SI Brochure 9th ed., Table 1",
      "retrieval_year": 2024
    },
    "domain": { "kind": "FundamentalPhysics" },
    "display_symbol": "c",
    "display_name": "speed of light in vacuum"
  }
}
```

The `unit` field serializes as a `UnitExpression` JSON object. For
`SpeedOfLight` (m·s⁻¹), the wire form would be:
```json
"unit": {
  "kind": "Binary",
  "value": {
    "op": {"kind": "Div"},
    "left":  {"kind": "Atom", "value": {"id": "Meter",  "prefix": null}},
    "right": {"kind": "Atom", "value": {"id": "Second", "prefix": null}}
  }
}
```
The simple example above uses `"unit": "Meter"` as a placeholder shorthand only.
All production wire payloads use the full `UnitExpression` encoding.

### 10.4 Stability

Wire format is stable for v0.x. New `CitationSource` variants serialize to
new tag strings without breaking readers that ignore unknown tags (forward
compatibility).

---

## 11. Test strategy

### 11.1 Catalog completeness

```rust
// tests/catalog_completeness.rs
// For every ConstantId variant, lookup_constant returns a valid entry.
// This test fails to compile if a new ConstantId is added without a catalog entry.
#[test]
fn every_constant_id_has_a_spec() {
    use mathcore_units::ConstantId;
    let ids = [
        ConstantId::SpeedOfLight,
        ConstantId::PlanckConstant,
        // ... all variants listed explicitly
    ];
    for id in ids {
        let spec = lookup_constant(id);
        assert_eq!(spec.id, id);
    }
}
```

The test lists every variant explicitly rather than using a count, so adding
a new `ConstantId` without a matching catalog entry causes a compile-time
exhaustiveness failure or a runtime assertion failure.

### 11.2 Unit coherence

For each `ConstantSpec`, the test verifies that the `unit` field's dimension
matches the expected physical dimension of the constant. The expected
dimension is encoded in a per-constant lookup table in the test. Example:

- SpeedOfLight → dimension: Length·Time⁻¹
- PlanckConstant → dimension: Length²·Mass·Time⁻¹ (= J·s)
- AvogadroNumber → dimension: Amount⁻¹
- FineStructureConstant → dimension: empty (dimensionless)

Because `ConstantSpec::unit` is a `UnitExpression`, the test evaluates
the expression's dimension by recursively resolving each `UnitExpression`
leaf via `mathcore_units::catalog::lookup_unit(id)` and combining dimensions
using the `UnitExpression` operator semantics (mul adds exponents, div
subtracts, pow scales). This evaluation logic lives in `unitalg`; the test
calls `unitalg::dimension_of(spec.unit)` (or an equivalent function) and
asserts against the expected `Dimension` value.

### 11.3 Citation discipline

```rust
#[test]
fn no_empty_citations() {
    for spec in all_constants() {
        assert!(!spec.citation.reference.is_empty(),
            "{:?} has empty citation reference", spec.id);
        assert!(spec.citation.retrieval_year >= 2019,
            "{:?} has implausibly old retrieval year", spec.id);
    }
}
```

### 11.4 Defined-exact serde round-trip

For every constant with `Uncertainty::ExactByDefinition`, serialize via
`ConstantSpecWire` and deserialize; assert the value round-trips exactly
(no floating-point drift for integer or symbolic values).

### 11.5 Domain coverage

Every `Domain` variant has at least one catalog entry. This catches
accidental domain reclassifications that empty a category.

### 11.6 Constants-as-units coherence

For every `UnitId` in mathcore-units § 5.7 that uses
`Conversion::FromConstant { id }`, verify that
`lookup_constant(id)` succeeds and that the constant's unit dimension
matches the unit's dimension in the mathcore-units catalog. This test
lives in the integration test suite (requiring both crates) and may be
placed in mathcore-units's test suite rather than here to avoid circular
test dependencies.

---

## 12. Crate layout

See § 5.1 for the full directory tree.

Feature flags:

```toml
[features]
default = ["std"]
std = ["mathcore-units/std"]
serde = ["dep:serde", "mathcore-units/serde"]
```

`no_std + alloc`: when `std` is not enabled, `BTreeMap`, `Vec`, and
`String` come from `alloc`. The static catalog itself uses only
`&'static [ConstantSpec]` and `&'static str`, which require neither
`std` nor `alloc`. The `alloc` crate is required only for
`ConstantSpecWire` (owned Strings) and the `constants_by_domain`
iterator (when it is backed by a filtered `Vec`). An alternative
iterator using a raw slice scan requires no allocation.

---

## 13. Versioning

| Change type | Version bump | Coordination required |
|---|---|---|
| Add new ConstantId (mathcore-units minor bump) + matching ConstantSpec | **minor** in both crates, same release | mathcore-units bumped first; mathcore-constants bumps in same window |
| Update a measured value (e.g., CODATA 2026 → new digits) | **minor** | Document in CHANGELOG; add new CitationSource variant |
| Correct a citation reference string or retrieval_year | **patch** | None |
| Fix a bug (wrong digit, wrong unit) | **patch** | Announce prominently; bug-fix corrections are never breaking |
| Rename or remove a ConstantId | **major** | Coordinate with mathcore-units and thales; follows workspace governance (thales/CLAUDE.md) |
| Change `ConstantSpec` field set or field types | **major** | Follows workspace governance |
| Add a new `CitationSource` or `Domain` variant | **minor** | Non-breaking addition |
| Remove a `CitationSource` or `Domain` variant | **major** | Breaking |

**Release lockstep rule:** when mathcore-units ships a minor bump that adds
a new `ConstantId` variant, mathcore-constants must ship a matching minor
bump in the same release window that provides the corresponding
`ConstantSpec`. An incomplete catalog (a `ConstantId` with no `ConstantSpec`
entry) is not a valid release state. The CI completeness test (§ 11.1)
enforces this at every commit.

---

## 14. Requirements summary (MC-1..MC-N)

| ID | Requirement | Severity |
|---|---|---|
| MC-1 | Every `ConstantId` variant from mathcore-units § 4 has exactly one `ConstantSpec` entry in the catalog | Blocker |
| MC-2 | `ConstantSpec` carries id, value (Expression), unit (UnitExpression), uncertainty (Option), citation, domain, display_symbol, display_name | Blocker |
| MC-3 | `Citation` carries source (CitationSource), reference (non-empty &'static str), retrieval_year (u16) | Blocker |
| MC-4 | `Uncertainty` has three variants: ExactByDefinition, Standard(Expression), Relative(Expression) | Blocker |
| MC-5 | `lookup_constant(ConstantId) -> &'static ConstantSpec` is the single lookup entry point; O(1) dispatch | Blocker |
| MC-6 | `all_constants() -> &'static [ConstantSpec]` returns the complete static catalog slice | Required |
| MC-7 | `constants_by_domain(Domain) -> impl Iterator<Item = &'static ConstantSpec>` with zero allocation | Required |
| MC-8 | SI-2019 defined-exact constants use `Uncertainty::ExactByDefinition` and `CitationSource::DefinedExact` | Blocker |
| MC-9 | Derived-exact constants (R, F, σ, ℏ, Parsec) store symbolic `Expression` values and `DerivedExact` source | Required |
| MC-10 | Measured constants store CODATA 2022 / IAU 2015 values with full stated significant digits and matching `Uncertainty::Standard` or `Uncertainty::Relative` | Blocker |
| MC-11 | Every `ConstantSpec` has a non-empty citation reference and a `retrieval_year >= 2019`; enforced by CI test | Blocker |
| MC-12 | Mathematical constants (Pi, E, EulerMascheroni, GoldenRatio) stored as symbolic `Expression` atoms, not truncated floats | Blocker |
| MC-13 | `no_std + alloc` support: static catalog requires neither; owned types require only `alloc` | Required |
| MC-14 | Serde opt-in via `serde` feature; `ConstantSpecWire` owns the serialized form; wire format stable for v0.x | Required |
| MC-15 | Serde wire format uses `#[serde(tag = "kind", content = "value")]` matching mathcore-units convention | Required |
| MC-16 | VacuumPermeability stored as measured (not exact) with CODATA 2022 value and note on SI-2019 shift | Blocker |
| MC-17 | Catalog completeness CI test enumerates all `ConstantId` variants explicitly; fails on missing entry | Blocker |
| MC-18 | Unit coherence CI test: each `ConstantSpec::unit` dimension matches the constant's physical dimension | Required |
| MC-19 | Constants-as-units coherence: every `Conversion::FromConstant { id }` in mathcore-units maps to a valid `ConstantSpec` entry | Required |
| MC-20 | Domain tag for every constant documented in § 6; no domain left empty (enforced by domain-coverage test) | Required |
| MC-21 | `ConstantId` and `UnitId` re-exported from crate root so callers need only one import | Required |
| MC-22 | Adding a new CODATA edition (e.g., Codata2026) adds a new `CitationSource` variant; old edition variant is retained | Required |

---

## 15. Resolved decisions

Decisions confirmed during spec review (2026-04-22):

1. **`Expression` as value type, not `f64`.** Exact constants lose no
   precision; symbolic derivations (ℏ, R, F, σ, Parsec) preserve their
   algebraic form; mathematical constants stay irrational. A plain `f64`
   would require truncation comments everywhere and break the CAS's
   symbolic pipeline.

2. **`ConstantId` stays in mathcore-units.** The enum must be accessible
   to mathcore-units's `Conversion::FromConstant` without importing
   mathcore-constants (which would create a circular dependency). Values
   live in mathcore-constants; identifiers live in mathcore-units.
   See mathcore-units SPECIFICATION.md § 4.

3. **No `Conversion::Expression` in mathcore-units.** The escape hatch was
   dropped in mathcore-units (see its resolved decision 5). This does not
   affect mathcore-constants; `ConstantSpec::value` is an `Expression`
   internally but mathcore-units never touches it.

4. **Uncertainty is metadata, never propagated here.** Propagation belongs
   in thales or mathlex-eval, where the computational graph is available.
   Storing uncertainty alongside the constant is the minimal useful
   contract.

5. **Symbolic values for derived-exact constants.** Storing R = N_A · k_B
   symbolically (rather than numerically) means a future change to either
   input constant automatically updates the derived constant when the CAS
   evaluates the expression. It also keeps the derivation self-documenting
   in the catalog source.

6. **IAU nominal values are exact by IAU resolution (where adopted as
   such).** SolarRadius, EarthRadius, SolarLuminosity are fixed by IAU
   resolution and carry `ExactByDefinition`. SolarMass, EarthMass,
   JupiterMass, and LunarMass are measured and carry `Standard` uncertainty.

7. **CODATA 2022 is the current edition.** When CODATA 2026 is published
   a new `CitationSource::Codata2026` variant is added; the 2022 variant
   is retained. Consumers can query which edition sourced a given constant.

8. **`ConstantSpecWire` for serde, not direct derive on `ConstantSpec`.**
   `ConstantSpec` holds `&'static str`; serde-deriving on it forces lifetime
   annotations throughout wire code. `ConstantSpecWire` owns Strings and
   derives cleanly.

9. **`constants_by_domain` is zero-allocation.** The iterator is a filtered
   scan of the static slice. No `Vec` allocation at call time. Consumers
   who need a collected result call `.collect()` themselves.

10. **No Planck units in v0.1.0.** Planck length, Planck time, and Planck
    mass are deferred. They require consistent handling of dimensional
    reductions (Planck units are combinations of G, ℏ, c, k_B) and belong
    in a future catalog addition once the symbolic simplification in thales
    is stable enough to validate them.

---

## 16. Flagged issues

### MC-FLAG-1: `ConstantSpec::unit` field type — RESOLVED

**Resolution (2026-04-22):** `ConstantSpec::unit` is now typed as
`UnitExpression` (not `UnitId`). This resolves the compound-unit limitation
identified in the original draft. The authoritative basis is
mathcore-units § 4 "ConstantSpec unit-field contract", which explicitly
mandates `UnitExpression` for this field. All catalog tables, the
`ConstantSpecWire` struct, and the MC-2 requirement have been updated
accordingly. No further action required.

### MC-FLAG-2: mathlex `Expression` symbolic atom coverage for γ and φ — OPEN

The spec stores `Pi`, `E`, `EulerMascheroni`, and `GoldenRatio` as
`Expression::Constant(MathConst::...)` (or equivalent mathlex symbolic atom
form). Whether mathlex's `Expression` type exposes `MathConst::EulerGamma`
and `MathConst::Phi` variants (in addition to π and e) is not confirmed.

**v0.1.0 pragmatic design decision** (see also § 6.5): the catalog implements
a symbolic/numeric mix based on what mathlex supports today:
- `GoldenRatio` (φ): stored symbolically as (1 + √5)/2 — exact algebraic
  form, expressible in mathlex without a dedicated atom.
- `EulerMascheroni` (γ): stored as `Expression::Float` with ≥ 50 significant
  digits, with a catalog comment noting the limitation. Upgraded to a symbolic
  atom once mathlex exposes `MathConst::EulerGamma`.

MC-12 (symbolic storage for mathematical constants) is fully met for π, e,
and φ; partially met for γ pending mathlex atom support. This issue remains
open as a mathlex-side dependency to track.

### MC-FLAG-3: `ConstantId::AstronomicalUnit`, `LightYear`, `Parsec` — RESOLVED

**Resolution (2026-04-22):** The dual appearance of `AstronomicalUnit`,
`LightYear`, and `Parsec` in both `UnitId` and `ConstantId` is intentional
by design. mathcore-units § 4 documents this explicitly:
- As `UnitId`: user writes `5 ly` — unit context, scale factor embedded
  in the catalog as `Linear { scale }`.
- As `ConstantId`: user writes `c · t / lightyear` — main-expression
  context, where the quantity is a named physical distance constant.
The two use cases are disjoint; the split keeps mathcore-units pure-data and
mathcore-constants free to evolve its value schema independently. No
asymmetry exists; the `UnitId` side does not use `Conversion::FromConstant`
for these three precisely because the scale factor is exact and baked in
directly, which is the more efficient representation for the unit catalog.
No change is needed.

### MC-FLAG-4: `VacuumPermeability` — CODATA 2022 value confirmation needed — OPEN

This spec cites μ₀ = 1.256 637 0621 × 10⁻⁶ H/m with relative uncertainty
1.5 × 10⁻¹⁰ from CODATA 2022. The catalog implementer must verify these
digits against the current NIST CODATA 2022 published table before the
catalog is frozen. The value shown here is a best-effort citation from
available sources but should be confirmed against
`https://physics.nist.gov/cuu/Constants/` at implementation time.
This is an implementation-time concern; it does not affect the spec structure.
