# SketchUp setup for selectable parts

Selection in the 3D viewer uses **names** when the GLB has them; if not, it falls back to **structure** (groups at a certain level).

## Why you might not see your names ("La mairie", etc.)

Many SketchUp → GLB exporters **do not preserve** Group/Component names. So even if you name "La mairie" in Entity Info, the exported GLB may have `(no name)` on those nodes. This is a known limitation of the export format/plugins.

**What the viewer does:**

1. **First:** It looks for **named** nodes (e.g. "La mairie"). If it finds them, those become selectable parts.
2. **Fallback:** If no (or almost no) names are found, it uses **structure**: it finds the node that has many children (e.g. 10+), and treats **each of those children** as one selectable part. So you still get hover/click per “block” even without names.

So you can keep naming in SketchUp for when you switch to an exporter that preserves names; until then, selection works by structure.

## What to do in SketchUp

1. **Create one Group or Component per “part”**  
   Each thing you want to click (e.g. a building, a block, a zone) should be inside its own Group or Component.

2. **Give each of those a name**  
   - Select the Group/Component  
   - Open **Entity Info** (Window → Entity Info, or right‑click → Entity Info)  
   - In **Name**, type a unique name, e.g.:
     - `Building A`, `Building B`
     - `Part_1`, `Part_2`
     - `Tower`, `Parking`, `Road`

3. **Export to GLB**  
   File → Export → 3D Model → **GLB** (or use your usual export).

## How it works in the viewer

- **If names exist in the GLB:** For each mesh, the code uses the **first ancestor that has a name** as the selectable “part”. All meshes under the same named Group/Component are one selectable unit.
- **If names are missing (common with SketchUp GLB):** The viewer uses a **structure fallback**: it finds the node that has many children (10+) and treats **each of those children** as one selectable part, so hover/click still works per block.

## Optional: name prefix filter

In `ThreeScene.vue` you can set:

```js
const SELECTABLE_NAME_PREFIX = 'Part_';
```

Then only nodes whose name **starts with** `Part_` (e.g. `Part_1`, `Part_Building`) are selectable. Leave it as `''` to use **any** named node.

## Checklist

- [ ] Each selectable “part” is one Group or Component  
- [ ] Each of those has a **Name** in Entity Info  
- [ ] Export as GLB and reload the viewer  
- [ ] Check the console: “Selectable parts (named in SketchUp)” should list your named parts  

If “Selectable parts” is empty, the structure fallback should still create parts; check the console for “Fallback: no names in GLB — using structure”.
