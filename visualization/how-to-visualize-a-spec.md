# How to visualize a spec

Ways to visualize a spec/idea so the holes show before you build. Most are Mermaid (the agent generates them from the spec; render at mermaid.live).

- **Flowchart** — the process: steps, branches, the loop
- **Swimlane** — who does what (user / agent / CI)
- **Sequence diagram** — interactions over time (the orchestration)
- **State diagram** — an entity's lifecycle / statuses
- **ER diagram** — the data model
- **Mindmap** — decompose the feature into parts
- **User journey** — the UX path + where it feels bad
- **Decision matrix** — compare architecture options
- **State-transition table** — every input × state → next state
- **Acceptance-criteria table** — Given / When / Then
- **Breadboard** — UI topology in words (places, affordances, links)
- **Wireframe / AI mockups** — the actual UI
- **C4 / architecture** — the system shape

> Pick the one that targets where THIS feature is most likely to be wrong.

## More — creative phase

Make it tangible, not just a diagram:
- **HTML prototype** — have AI generate a real static HTML/CSS page (even a whole simple site) and open it in the browser. See the actual UI, not a box.
- **Excalidraw drawing** — sketch it rough by hand on the canvas; thinking, not polish.
- **AI image (Gemini / Imagen)** — generate an illustration, chart, or scene of the concept.
- **Clickable prototype** — Figma/Framer or AI-built; click through the real flow.
- **Storyboard** — the user flow as comic-strip frames.
- **Annotated screenshot** — a real screenshot with arrows/notes on what changes.
- **Before / after** — the two states side by side.
- **One-pager / slide** — a single visual summary of the whole idea.
- **Loom walkthrough** — narrate the flow out loud over a screen recording.

Extra diagram types (mostly Mermaid):
- **Quadrant chart (2×2)** — prioritize, e.g. impact vs effort
- **Gantt / timeline** — phases and milestones over time
- **Kanban board** — the work as todo / doing / done
- **Block diagram** — boxes/blocks layout of components
- **Pie / bar chart** — proportions (traffic split, effort, %)
- **Sankey diagram** — flow volumes (a funnel, data throughput)
- **Requirement diagram** — formal requirements and how they relate
- **Git graph** — branch / release / rollout flow
- **Concept map** — nodes + labeled relationships (richer than a mindmap)
- **Dependency graph** — what depends on what; build/load order
- **Data flow diagram (DFD)** — how data moves through the processes
- **Component tree** — the UI component hierarchy
- **API contract (OpenAPI / Swagger)** — endpoints, inputs/outputs, rendered
- **Pseudo-code outline** — the logic as indented steps (text as structure)
- **Radar chart** — compare options across many axes at once
- **Wardley map** — strategic positioning of the pieces
