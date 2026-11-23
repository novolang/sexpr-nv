# Changelog

All notable changes to sexpr-nv are recorded here. The format is
[Keep a Changelog](https://keepachangelog.com/en/1.1.0/), and this
package follows [Semantic Versioning](https://semver.org/spec/v2.0.0.html)
with the pre-1.0 rule that a breaking change bumps the MINOR number.

## [0.1.0] — 2026-11-23

### Added

- `sexprscan`, a pull scanner over the bytes of an S-expression, and `walk`, a one-pass visit of every token. A token carries its kind, byte range, line and column and costs no allocation.
- `sexprtree`, an arena of numbered nodes with the position of each, `parse` and `parse_with`, navigation by child, sibling and head, `text`, `source_of`, `as_int`, `as_float` and `unquote`.
- `sexprwrite`, `write` and `write_pretty` in KiCad's layout, `quote`, `is_bare` and `from_tree`.
- `sexprerr`, the faults of a reader, each with a line and a column.
