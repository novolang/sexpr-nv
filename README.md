# sexpr-nv

An S-expression is a list written between parentheses whose elements are atoms, quoted strings or further lists. KiCad writes its board files (`.kicad_pcb`), schematics (`.kicad_sch`), symbol libraries (`.kicad_sym`) and netlists (`kicadsexpr` format) in this syntax. This package reads and writes it in novo-lang, and gives every node the line and column where it began, so a tool can say `board.kicad_pcb:412:9` about what it found.

The grammar is the one described on KiCad's developer documentation page "S-expression format" and the file-format pages that follow it.

## What it is

There are three kinds of element.

| Element | Written | Example |
| --- | --- | --- |
| List | `(`, the elements separated by white space, `)` | `(at 148.59 100.33 90)` |
| Atom | a run of bytes that is not white space, a parenthesis or a quote | `F.Cu`, `-0.5`, `#PWR01`, `roundrect` |
| String | `"`, the content with backslash escapes, `"` | `"Resistor_SMD:R_0603"`, `""` |

White space is every byte up to and including `0x20`. There are no comments. An atom is not typed: `148.59` is an atom whose text happens to be a number, and `"148.59"` is a string. KiCad relies on that difference, and `as_int` and `as_float` answer a number for an atom only.

Inside a string, a backslash escapes the next byte. `\n`, `\r`, `\t`, `\a`, `\b`, `\f` and `\v` are the control bytes of the same letters. `\x` and one or two hexadecimal digits is that byte. A backslash and one to three octal digits is that byte. Any other byte after a backslash stands for itself, so `\"` is a quote and `\\` is a backslash.

## Install

```
novo pkg add sexpr-nv
```

## Example

```novo
use std.bytes
use sexprtree
use sexprwrite

fn main() [io]
    let src = bytes.from_str("(footprint \"R1\"\n  (layer \"F.Cu\")\n  (at 148.59 100.33 90))")
    match sexprtree.parse(src)
        Err(e) => println(e.message())
        Ok(t)  =>
            // The child list of the footprint whose head is `at`.
            let at = sexprtree.find(t, 0, "at")
            // The element after the head is the x coordinate.
            let x = sexprtree.as_float(t, sexprtree.nth(t, at, 1)) ?? 0.0
            println("at ${sexprtree.where_is(t, at)}: x = ${x}")
            // A value built by hand is written in one line.
            let moved = SxList([SxAtom("at"), sexprwrite.float_atom(x + 10.0), SxAtom("100.33"), SxAtom("90")])
            println(sexprwrite.write(moved))
```

The program prints `at 3:3: x = 148.59` and `(at 158.59 100.33 90)`.

## What the package contains

| Module | Contents |
| --- | --- |
| `sexprscan` | A pull scanner. `next` answers one token at a time with its kind, byte range, line and column, and allocates nothing for a token. `walk` calls a function for every token. |
| `sexprtree` | `parse` builds an arena of numbered nodes. Functions read a node's kind, position, children, head, text and number. |
| `sexprwrite` | `SexprValue`, the value to write; `write` for one line; `write_pretty` for KiCad's indentation; `from_tree` to turn a parsed subtree into a value; `quote` and `is_bare`. |
| `sexprerr` | `SexprFault`, the refusal with its line and column, and its message and code. |

## How to choose an entry point

Use `sexprscan.walk` or `sexprscan.next` when one pass is enough: counting footprints, collecting net names, finding a bounding box. Memory use is the source and a few integers, whatever the size of the file.

Use `sexprtree.parse` when the program moves around the file, asks for the `at` of one footprint and the `net` of one of its pads, or needs a position for a diagnostic after the pass has ended. The tree keeps the source and seven integers per node. A generated board of 5 MB with 866,000 nodes parses in about 0.1 s and is walked in about 0.04 s, measured once on a development machine.

Use `sexprwrite` to produce text. Build a `SexprValue` by hand, or take one from a tree with `from_tree`, change it and write it back.

## The rules a user needs

1. A column counts bytes from the start of the line, from 1. A line ends at a line feed, so a file with CRLF line ends has the line numbers of the same file with LF.
2. A fault carries the position of the byte it is about. An unclosed list names the innermost list that was never closed. An unterminated string names the quote that opened it.
3. `parse` reads exactly one top-level form, which is how KiCad's files are made. A second form after it is a fault, and so is an empty input.
4. Lists may nest to 256 levels by default. `parse_with` and `sexprscan.scanner` take another limit. Deeper nesting is a fault and never a stack overflow, because neither the scanner nor the parser recurses.
5. `text` answers the decoded content of a string and the bytes of an atom. `source_of` answers the bytes as written, which for a list is the whole form.
6. `head` of a list is the text of its first element when that element is an atom or a string. `find` and `find_all` match a child list by its head.
7. An atom that cannot be written bare, because it is empty or holds white space, a parenthesis or a quote, is written as a string. The writer never produces text that reads back as something else.
8. A string is written with `\\`, `\"`, `\n`, `\r` and `\t`, and `\xHH` for any other byte below `0x20`. Bytes from `0x20` up are written as they are, so UTF-8 passes through.
9. `write_pretty` produces KiCad's layout: a list with no list in it stays on one line, and a list with lists in it puts each element after its leading atoms on its own line, one level deeper, with the closing parenthesis on its own line. KiCad 8 indents with one tab. Neither writer ends with a newline.

## Design

The tree is an arena of parallel integer lists and not a value per node. A board file has hundreds of thousands of nodes, and a heap object per atom would cost more than the file. The text of a node is sliced out of the source when it is asked for.

Two choices have another reasonable answer.

- **No comments.** KiCad's own syntax has no comment form in the files it writes, and `#` begins a net name such as `#PWR01`. A reader of the Specctra DSN format, which does have `#` comments, would need a switch. This package has none.
- **A raw line feed inside a string is accepted.** KiCad writes a line feed in a string as `\n`. A stricter reader would refuse the raw byte. This one keeps the content, and counts the line, so a hand-edited file still reads.

## What is not included

- **Files and the file system.** The caller reads the bytes and writes the text. Every function declares no effect.
- **KiCad's object model.** A footprint, a pad and a track are lists with heads. This package does not know what any head means.
- **Number formatting to KiCad's precision.** `float_atom` writes the shortest text that reads back as the same number. A caller that needs six decimals builds the atom itself.
- **Several top-level forms.** A stream of forms is read by calling the scanner.

## Related packages

- [toml-nv](https://novo-lang.org/packages/toml-nv) and [yaml-nv](https://novo-lang.org/packages/yaml-nv) read the configuration formats that KiCad does not use.
- [xml-nv](https://novo-lang.org/packages/xml-nv) reads the XML netlist that KiCad can also write.

## Tests

```bash
novo test tests/sexpr_tests.nv         # the scanner, the tree and the navigation, on a board and a netlist
novo test tests/sexpr_edge_tests.nv    # every fault, every escape, the limits
novo test tests/sexpr_write_tests.nv   # quoting, both layouts, the round trip
bash tests/coverage.sh                 # line coverage over src/, merged across the suites
```

The fixtures are written for this package in the shapes KiCad's file-format documentation describes: the `kicad_pcb` root with `version`, `generator`, `layers`, `net` and `footprint` forms, and the `export` netlist with `design`, `components` and `nets`. The round-trip suite parses a board in KiCad 8's layout and writes it back byte for byte.

## Licence

Apache-2.0. See `LICENSE`.
