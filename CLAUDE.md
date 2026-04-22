# mathcore-constants — Project CLAUDE.md

Workspace member of `libraries/thales/`. Inherits workspace-level rules
from `../CLAUDE.md` and global rules from
`~/.config/claude/claude-ent/CLAUDE.md`.

## Governance

Subordinate to thales. API decisions serve thales's symbolic-constants
evaluation, mathlex's M-R3 annotation resolution, and mathlex-eval's
numeric substitution. Depends on `mathcore-units` for value-typing.
Once `0.1.0` ships on crates.io, additions are free; breaking changes
coordinate with thales and with mathcore-units releases.

## Status

Pre-specification. The authoritative spec (MC-1…MC-N) will land in this
repo as `SPECIFICATION.md` after `mathcore-units` freezes its spec
(because MC-* depends on MU-* types). Until the spec is frozen:

- No architecture rules yet.
- No constants data yet; the catalog structure is a spec decision.
- No tests beyond the placeholder.

Once `SPECIFICATION.md` is accepted, this file gains the crate's
architecture rules and the catalog is populated.

## Constraints that already apply

- MIT license.
- Depends on `mathcore-units` via workspace path during dev,
  version constraint on crates.io.
- Serde is opt-in via the `serde` feature, and only if
  `mathcore-units/serde` is also enabled.
- All constants must have a citation to an authoritative source
  (CODATA, IUPAC, IAU, mathematical definition). No unsourced values.
- Constants are not stored as raw `f64`; they carry
  `mathcore_units::Unit` at all times.

## Active work

None yet. Specification drafting blocked on `mathcore-units` spec
acceptance.
