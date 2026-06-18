---
name: doc_formatting
description: Rules for formatting technical documentation, especially numerical examples and formulas.
---

# Documentation Formatting Rules

When writing or updating technical documentation (especially markdown or HTML docs), always adhere to the following rules:

1. **Strict Formula Derivation**: Whenever presenting a numerical example or calculation, NEVER just state the final result. You MUST strictly follow the pattern: `Formula = Value Substitution = Final Result`.
   - Bad: `V_cu = 16`
   - Good: `V_cu = \frac{\sum p_i^2}{N} - \mu_{cu}^2 = \frac{3690496}{256} - 120^2 = 14416 - 14400 = 16`

2. **No Magic Numbers**: Every number used in a numerical example must have a clear, traceable origin. If a value is used, explain exactly how it was calculated or where it came from in previous steps or hardware components.
   - For example, if using `B_actual = 300,000`, specify that it comes from the CABAC entropy encoder counting output bins.
   - If using `QP_avg = 20`, show the bit-weighted average calculation that produced it from individual CU QPs and Bit counts.

3. **Link to Context**: Always use hyperlinks (anchor tags) to link variables or concepts back to where they were originally defined or calculated in earlier sections of the document. This prevents the "sudden appearance" of parameters.

4. **Distinct Link Colors**: When inserting hyperlinks (anchor tags) in documentation, ensure their color is visually distinct from any bold or emphasized text colors used in the same paragraph. For example, if `<strong>` tags use a blue color (`#0288d1`), links should use a distinct color like purple (`#7c3aed`) to prevent user confusion.

5. **Diagram Node Naming**: When creating diagrams (e.g., Mermaid, Graphviz), never include document section numbers (like "2.1", "2.2") in the node titles or labels. Diagram nodes should represent logical components or actions, independent of the document structure.

6. **Professional Spec Terminology**: Eliminate colloquial or metaphorical phrasing (e.g., "零花钱", "马赛克", "死守", "榨干") in favor of strict, objective spec terminology (e.g., "基础比特配额", "主观失真", "严格遵循约束", "大幅压缩").

7. **UI Element Disambiguation**: Do not use identical visual formatting for different interactive elements. If hyperlinks use underlines, inline highlights must use a distinct style (like subtle background colors or badges) to prevent "false affordance" clicks.

8. **TOC Strictness**: Dynamic Table of Contents (TOC) generators should strictly filter out unnumbered auxiliary titles (e.g., using regex `^\d` checks) to maintain a rigorous hierarchical structure.

9. **Business Context Alignment**: Always tie abstract architectural formulas to their macro business scenarios (e.g., Live vs. VOD / ABR vs. CRF). Use distinct visual layouts (like flexbox badges) to enhance structural comprehension.

10. **Graphviz Rendering Robustness**: Avoid using font-metric-dependent formatting (like `<b>` tags) or horizontal alignment when exact element sizing is critical in Graphviz DOT files. Favor vertical text stacking or `splines=polyline` for robust, overlap-free diagrams.
