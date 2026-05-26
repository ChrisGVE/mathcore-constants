# mathcore-constants

Physical, mathematical, and chemical constants with units, uncertainty, and
source citations for the mathcore/thales ecosystem.

Provides:

- A registry of named constants (speed of light, Planck's constant, Avogadro's
  number, π, e, γ, …) keyed by stable enum ids
- Each constant carries a value, its `mathcore_units::Unit`, an uncertainty
  (where applicable), a citation to the authoritative source (CODATA, IUPAC,
  IAU, mathematical definition), and a domain tag (physics, math, chemistry,
  astronomy, …)
- Serde-capable wire format (opt-in via `serde` feature)
- Category filters (`constants_by_domain(Domain::Physics)`, etc.)

## Status

**Pre-release.** The specification (MC-1…MC-N) is being drafted; API will
settle before the `0.1.0` release to crates.io. Integration with
mathcore-units is the blocking dependency.

## Relationship to thales

This crate is a **subordinate member** of the thales workspace. It is
consumed by thales for symbolic-with-constants evaluation, by mathlex for
M-R3 annotation resolution, and by mathlex-eval for numeric substitution.
Additive-only backward compatibility once `0.1.0` ships.

See the workspace-level `CLAUDE.md` for governance notes.

## Install

Not yet published to crates.io. Track this repository or the thales workspace
for release readiness.

## License

Apache License 2.0. See `LICENSE`.
