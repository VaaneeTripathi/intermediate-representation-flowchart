# JSON Schema

This document serves as a walkthrough, justification of design choices, and a reference for the JSON Schema constructed as part of my Capstone Thesis. The first step to reaching here was to explore what the requirements are for an IR - that is, what data it needs to capture. For this, we constructed a table that outlines the information to be included, why it needs to be included, what literature supports this, instances observed in the videos watched for literature review, and what annotating them would look like.

The table can be found here: [Thesis IR Requirements](https://docs.google.com/spreadsheets/d/1AntonupgmWn9se1v5Xf6j6UbX7Bw4INg9vhZyAFw5qQ/edit?usp=sharing)

Then, from the table, all the needs were divided into four categories. These can be explained as follows:

---

## Logical Groupings

These groupings together describe a flowchart as a layered system that encodes visual form, organization, behavior, and meaning. The Elements group refers to the basic visible components—nodes and their connections—along with their explicit visual attributes, which make the diagram perceptible and readable. Structure captures how these components are arranged into higher-order organization, such as hierarchy, compacted or abstracted views, and links across multiple flowcharts, allowing navigation between different levels of detail. State introduces dynamism by accounting for spatial positioning, temporal evolution, and replication, enabling the flowchart to represent processes that change over time or recur. Finally, Semantics provides interpretive depth through narrative context, domain-specific symbols, and annotations, transforming the diagram from a purely graphical layout to a comprehensive representation, ensuring the computational and informational equivalence of the flowcharts.

| Category | Contents |
|----------|----------|
| **Elements** | Nodes, Connections (with explicit visual attributes) |
| **Structure** | Hierarchy, Compacted Version, Cross-flowchart Links |
| **State** | Spatial Data, Temporality, Replication |
| **Semantics** | Narrative Context, Symbol Support, Annotations |

---

## Why JSON?

The first question after all this analysis is what representation language to use to capture all this data. Some options that we considered were Mermaid, XML, and UML. However, for the long-term development goals of the project, JSON seemed the most suitable. This can be explained as follows:

- **Data model flexibility:** JSON makes no assumption about whether data is a tree or a graph, so hierarchy and graph connectivity sit naturally side by side in the same document. XML is fundamentally a tree and forces you to abandon nesting to express graph connections, ending up with the same ID reference pattern as JSON but with more syntax overhead. Mermaid is a rendering DSL — it has no document model at all, just a string that a renderer interprets. UML has a metamodel that can express both but requires a heavy toolchain (XMI serialization) to actually store it.

- **Change tracking:** JSON Patch provides a standardised way to record what changed between versions as a sequence of operations, directly satisfying the temporality requirement without storing full snapshots. XML has no equivalent standard for delta tracking — you either diff the full document or implement your own change format. Mermaid has no versioning concept whatsoever — it is a static string. UML versioning is tool-dependent and not standardised across implementations.

- **Parse simplicity:** A single `JSON.parse()` call is all a student's assistive technology needs, with no generated decoder code, no schema compilation, and no binary decoding library. XML requires a DOM or SAX parser which, while available in browsers, is substantially more verbose to use. Mermaid requires its own rendering library to be loaded before anything can be interpreted. UML/XMI has no browser-native parser at all.

- **Performance:** XML is measurably slower than JSON in transmission speed and payload size, compounding across every incremental update in a live session. Mermaid and UML are not designed for real-time streaming at all and have no performance profile relevant to this comparison.

- **Extensibility:** JSON places no restrictions on what data you add to a document, so fields like narrative context, plain-language node descriptions, symbol dictionaries, and annotation metadata can be added wherever needed without fighting the format. JSON Schema can then optionally enforce that these fields are present before a tool attempts to render or speak the content. XML technically allows custom elements but its schema system (XSD) is complex enough that adding new accessibility fields in practice means significant schema maintenance overhead. Mermaid is a closed rendering syntax — there is nowhere to attach custom contextual data like a narrative description without it breaking the renderer. UML supports tagged values for custom data but they are tool-specific, meaning custom accessibility fields added in one UML tool will not reliably survive export to another.

---

## Structure of the Schema

### JSON Schema Essentials

The metadata for the JSON specifies the stable version used, the version of the schema, title, description, type, required keys, and sets `additionalProperties` as false to take a strict approach.

### ID

This sets a unique ID for the particular flowchart being represented; this is different from the JSON `$id` field, which refers to the version of the entire schema itself. Each flowchart is identified by a timestamp by setting the format as `date-time`.

---

### Elements

The category elements has two required properties - nodes and connections. These are the most basic data fields, but cover the essential data.

#### Nodes

Node is defined as a list, and then `items` is used to enforce that all elements in the array must conform to the sub-schema defined inside items. Now, nodes have three required properties - shape, text, id. The shapes are constrained to certain common ones; this could be extended as more shapes are observed. The text inside node is stored as a string. The position and visual attributes are abstracted away for both nodes and connections and defined later with just a colour key right now. A flowchart is defined only if there is atleast 1 node.

#### Connections

Connections are also defined as a list of objects. Each connection requires three properties - id, source, and target. The source and target are string references to node IDs, establishing the graph edges. Beyond these required fields, connections carry optional metadata: `direction` indicates the arrow direction using one of four values (forward, backward, bidirectional, or none), defaulting to forward. This separates the visual arrow direction from the structural source-target relationship — a backward connection keeps its source and target as listed but draws the arrow in reverse. The `line-type` field specifies the visual style of the connection (solid, dashed, or dotted), defaulting to solid. An optional `label` field stores any text displayed on or near the connection, such as "Yes" or "No" on decision branches. Like nodes, connections also carry an optional visual attribute reference for colour.

---

### Structure

The structure category currently contains one property — hierarchy.

#### Hierarchy

Hierarchy is defined as an array of group objects, where each group has three required properties: id, parent, and children. The `parent` field accepts either a string (pointing to another group's ID for nesting) or null (for flat, top-level groupings). This dual-type approach was a deliberate choice — by keeping parent required but nullable, every group must explicitly declare whether it is nested or top-level, preventing ambiguity between "intentionally flat" and "parent was forgotten." The `children` field is an array of strings referencing node IDs or sub-group IDs, with a minimum of one item — a group with no children would be semantically meaningless. An optional `label` field provides a human-readable name for the group.

---

### State

The state category captures the current, dynamic aspects of the board at any given moment. It contains three properties: crossLinks, compacted, and boardState.

#### Cross-Links

CrossLinks handles the multi-flowchart case — when a whiteboard contains several flowcharts that reference each other. Each cross-link is an object with four required fields: `sourceFlowchart` and `targetFlowchart` (the IDs of the two flowcharts involved) and `sourceElement` and `targetElement` (the IDs of the specific elements being linked across those flowcharts). An optional `label` can describe the nature of the relationship. This means the IR is not just a single-graph format; it is aware of a broader board context where multiple flowcharts coexist and interact.

#### Compacted

Compacted provides the progressive disclosure mechanism, which is critical for accessibility. A complex flowchart with many nodes would be overwhelming for a screen reader to traverse in full. Compacted addresses this through three sub-properties. The `detailLevel` field uses `oneOf` with `const` values rather than a plain enum, so that each level carries its own description: "full" means all nodes and connections are visible, "summary" means collapsed groups are shown as single summary nodes, and "minimal" means only the top-level flow with group summaries is presented. The `collapsedGroups` field is an array of hierarchy group IDs that are currently collapsed. The `summaries` field pairs each collapsed group with a plain-text summary, so that a screen reader can announce "This section handles error recovery with 8 steps" rather than reading all 8 nodes individually.

#### Board State

BoardState is a plain-text string that captures an overall description of the current board state. This replaces what was previously a more granular spatial data object with viewport and bounds properties; a free-form string was chosen instead because the exact viewport coordinates are less useful for accessibility purposes than a human-readable description of what is currently happening on the board.

---

### Semantics

The semantics category is arguably the most important for the stated accessibility purpose of the IR. It transforms raw graph data into something meaningful for non-visual consumption. It contains three properties: narrative, symbols, and annotations.

#### Narrative

Narrative provides natural language context for the flowchart through four sub-properties. The `overview` field is a high-level description of what the flowchart represents (for example, "This flowchart shows the algorithm for binary search"). The `purpose` field explains why the flowchart exists in the learning material (for example, "Used in Week 3 to illustrate divide-and-conquer"). The `dialogue` field captures the current conversational context in which the flowchart is being discussed — this is particularly useful in a live lecture setting where the instructor's spoken explanation provides context that is otherwise lost when a student encounters the diagram in isolation. The `walkthrough` field is an ordered array of element-narration pairs that provides a guided tour through the flowchart. Each entry pairs an elementId with a narration string. The array ordering is significant — JSON arrays are ordered, so the author controls the exact narrative sequence. This imposes a linear reading order on an inherently non-linear structure, which is precisely what a screen reader needs to present the flowchart step by step.

#### Symbols

Symbols serves as a legend — a lookup table that maps visual conventions to their semantic meaning. Each entry has two required fields: `symbol` (a string like "diamond" or "dashed line") and `meaning` (a description like "decision point" or "optional path"). An optional `applicableTo` array lists the IDs of elements that use this symbol. This is critical for accessibility because sighted users absorb shape conventions visually — everyone "knows" a diamond is a decision — but that knowledge must be made explicit for non-visual consumption. By making this a per-flowchart table rather than hardcoding it, the schema supports non-standard conventions, such as an instructor using triangles for something domain-specific.

#### Annotations

Annotations captures interpretive overlays — things an instructor draws on top of the flowchart as meta-commentary, such as circling a critical node, drawing an arrow to a common mistake, or adding a cloud with a note. Each annotation has three required fields: `id`, `target` (the ID of the annotated element), and `annotationType` (one of eight values: container, separator, highlight, circle, underline, arrow, cloud, or strikethrough). An optional `content` field, which can be a string or null, stores the text attached to the annotation. For accessibility, this field is what converts a visual gesture (a red circle around a node) into a communicable description ("I circled this node because it is the most common source of bugs").

---

### Shared Definitions (`$defs`)

At the bottom of the schema, the `$defs` section contains two reusable sub-schemas referenced throughout the document via `$ref`. The **position** definition is a simple object with two required numeric fields, x and y, representing a point in 2D space. It is used by node positions and can be referenced wherever spatial coordinates are needed. The **visualAttrs** definition is an object currently containing a single optional colour field. Both definitions use `additionalProperties: false` to maintain the same strict approach as the rest of the schema. Abstracting these into `$defs` avoids duplication — any property that references position or visual attributes points to the same definition, so a change in one place propagates everywhere.
