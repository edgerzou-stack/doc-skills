---
name: verilog-coding-style
description: Apply the team Verilog/SystemVerilog RTL coding style. Use when Codex writes, edits, reviews, documents, or refactors Verilog/SystemVerilog RTL, AXI/AXI-Stream interfaces, ready/valid pipelines, FSMs, DFF/register logic, generate blocks, macros, or module port declarations according to this team's conventions.
---

# Verilog Coding Style

## Workflow

Use this skill when working on RTL code or reviewing RTL style.

1. Read `references/verilog_style.md` before making style-sensitive RTL changes.
2. Preserve existing project conventions unless they conflict with the style reference or the user explicitly asks to update them.
3. Prefer small, local edits that improve consistency without broad refactors.
4. When reviewing code, report concrete violations with file and line references.
5. When writing new RTL, use the style reference as the default contract.

## Core Defaults

- Use ANSI-style ports, but do not write `input wire` or `output wire`.
- Use `_i` and `_o` suffixes for module input/output ports.
- Use `_d` and `_q` for register D and Q sides.
- Prefer DFF/DFFE/DFFR/DFFRE wrappers for registers; do not phrase this as a hard ban on handwritten clocked always blocks unless the user asks for strict enforcement.
- Prefer `assign` for simple combinational logic; use `always @(*)` only for complex branch logic, especially `case`.
- Use only three-part FSM style: state register, next-state combinational logic, output combinational logic.
- Use standard `valid/ready` handshakes for upstream/downstream interfaces.
- Prefer SystemVerilog `interface` + `modport` for AXI/AXI-Stream.
- Name AXI interfaces with `<module_tag>[_<purpose>]_<role>_<axi_type>`, for example `dma_reader_mem_m_axi`.
- Name every `generate` block with `begin : GEN_BLOCK_NAME`.
- Pair local `` `define `` usage with a corresponding `` `undef ``.

## Reference

Detailed rules, naming tables, examples, AXI channel suffixes, and review checklist are in:

- `references/verilog_style.md`
