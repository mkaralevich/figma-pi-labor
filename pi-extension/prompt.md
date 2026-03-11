## figma-labor

Live connection to Figma via the figma-labor bridge (Plugin API). Plugin: {{plugin_status}}

Use these tools to **read and manipulate the canvas** — creating, updating, and deleting nodes. For design inspection or code generation from a design, prefer the `get_design_context` / `get_screenshot` tools (figma-mcp) instead.

**Node IDs:** strings like `"123:456"` · paste a Figma URL and the ID is extracted automatically.
**Colors:** r/g/b/a in 0–1 (e.g. Shopify green = {r:0, g:0.502, b:0.376}).

**Non-obvious API behaviors:**
- Children are back-to-front: index 0 = bottommost layer, last = topmost
- `figma_create_instance` needs a COMPONENT id, not COMPONENT_SET — use `figma_get_children` on the set to find the right variant
- `setProperties()` on an instance: TEXT, BOOLEAN, and INSTANCE_SWAP property names must be suffixed with `#<id>` (from `componentPropertyDefinitions`); VARIANT does not need the suffix
- Text nodes require the font to be loaded before setting characters, fontSize, fontName, or alignment — font must exist in the document
- `figma_detach_instance` also detaches all ancestor instances, not just the target
- `figma_set_layout` requires `layoutMode` to be HORIZONTAL or VERTICAL before alignment or sizing modes take effect
- Alternating writes to a ComponentNode and reads from its InstanceNode is slow — batch reads first, then writes

**`figma_run_script` — key rules:**
- Both `figma.getNodeById(id)` and `figma.getNodeByIdAsync(id)` are patched to handle compound instance IDs (containing `;`) safely — they fall back to `findOne` when needed
- Never access `node.mainComponent` (sync) — always use `await node.getMainComponentAsync()`
- `combineAsVariants()` may throw "proxy: inconsistent get" — clone an existing COMPONENT_SET instead
- **Performance on large pages:** scope `findAll` to a section/frame, not the entire page. Use `sec.findAll(pred)` instead of `figma.currentPage.findAll(pred)` when possible
- **Batching:** pre-compute fill/paint objects once and reuse in loops — don't call `solidPaint()` or `setBoundVariableForPaint()` per node

**Variable-bound fills — correct pattern:**
When binding a design variable to a fill, the base paint RGB+opacity MUST match the variable's resolved value. Otherwise Figma renders the base color, not the variable.
```js
// ✅ Correct — base color matches variable value
const paint = { type:'SOLID', visible:true, opacity:0.04, blendMode:'NORMAL',
  color: { r:0.135, g:0.195, b:0.254 } }; // matches variable's actual RGB
const bound = figma.variables.setBoundVariableForPaint(paint, 'color', variable);
node.fills = [bound];

// ❌ Wrong — solidPaint('#FF0000') sets red base, variable won't display
```

**Workflow:** read selection → propose change → apply → undo immediately if wrong → verify.
