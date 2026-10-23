# RAGS Engine Specification

Reverse-engineered specification of the RAGS game engine, derived from decompiled
source. Documents the SDF database schema, text substitution system, command/condition
enums, and AES encryption used in the file format.
Based on 3.0.60 (WPF). The command/condition system is identical to 2.5.2–2.6.1
(WinForms) — verified by diff of the CommandType enum.


**Browser-based player:** [rags2html](https://github.com/Kassy2048/rags2html)

**Preservation archive:** [rags game on archive.org](https://archive.org/details/rags-suite-preservation-archive)

**Decompiled source:** [RAGS-decompiled](https://github.com/laine-navi/RAGS-decompiled)

### Contents

- `SPEC.md` — Full engine specification

### AI disclosure

This is the first time I've touched C#, I used AI to help be write the grep and sed commands to research the decompiled code and also used it to help me draft the spec. The actual content I took directly from the source.

### License

CC BY 4.0
