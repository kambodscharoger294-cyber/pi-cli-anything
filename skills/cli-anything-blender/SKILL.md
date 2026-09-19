---
name: cli-anything-blender
description: "Blender headless steuern. Zwei Wege: cli-anything-Harness (49 Commands in 10 Gruppen, stateful JSON-Szene) für Standard-Operationen (Primitive, Material, Kamera, Licht, Modifier, Rendern), bpy-Python-Skripte für volle Kontrolle (Import/Export STL/OBJ/glTF/FBX, Spezialoperationen). Renders als PNG (EEVEE/Cycles). Für pi: 3D-Modelle erzeugen, umwandeln, rendern – ganz ohne GUI."
---

# cli-anything-blender

Blender headless (ohne GUI) steuern. Blender ist installiert und `blender` ist
im PATH (Version 5.2 LTS). Zwei Wege, je nach Aufgabe:

| | Weg 1: Harness | Weg 2: bpy-Skript |
|---|---|---|
| Wofür | Standard-Operationen: Szene aufbauen, Material, Kamera/Licht, Modifier, Render-Settings | Alles andere: STL/OBJ/glTF-Import & Export, Decimate, Booleans, Spezial-API |
| Zustand | Stateful JSON-Projekt (`.blend-cli.json`), inspectierbar & diffbar | Ein Skript = ein Durchlauf |
| Stärke | Schnell, deklarativ, Undo/Redo-History | Volle bpy-Power, keine Grenzen |

Faustregel: **Einfache Szene + Render → Harness. Export nach STL/glTF oder
Filtrierung bestehender Meshes → bpy-Skript.** Beide Wege kombinierbar (s. u.).

## Weg 1: Harness (cli-anything-blender)

Binary (als uv-Tool `cli-anything-hub` installiert, gleiche Instanz wie
`cli-anything-freecad`):
`~/.local/share/uv/tools/cli-anything-hub/bin/cli-anything-blender`

**49 Commands in 10 Gruppen:** object (7), modifier (6), scene (6), animation
(5), material (5), render (5), camera (4), light (3), session (4), preview (4).

### Verifizierter Standard-Flow (getestet 16.09.)

```bash
BIN=~/.local/share/uv/tools/cli-anything-hub/bin/cli-anything-blender
P=./szene.blend-cli.json

$BIN scene new --output $P                        # Szene anlegen + persistieren
$BIN --project $P object add cube -n Box -l 0,0,1
$BIN --project $P material create -n Rot -c 1,0,0,1
$BIN --project $P material assign 0 0             # Material 0 → Objekt 0
$BIN --project $P modifier add subdivision_surface -o 0 -p levels=2
$BIN --project $P light add sun -l 3,3,5
$BIN --project $P camera add -l 5,-5,3
$BIN --project $P camera set-active 0
$BIN --project $P render execute out.png          # ACHTUNG: generiert nur Skript!
blender --background --python _render_script.py   #echter Render-Lauf
```

### Stolperfallen (alle selbst reingetreten)

- **Der Harness ist stateless pro Aufruf.** Ohne `--project` geht der Zustand
  zwischen zwei Commands verloren ("No scene loaded"). Immer mit
  `--project $P` arbeiten.
- **`scene new` braucht `--output`**, sonst wird nichts gespeichert.
  Umgekehrt kann `--project` keine neue Datei anlegen (FileNotFoundError).
- **`render execute` führt nichts aus** – es generiert nur `_render_script.py`
  und printet den blender-Befehl. Den blender-Lauf macht man selbst dran.
- **`modifier add` will den Kleinbuchstaben-Namen** (`subdivision_surface`,
  `mirror`), NICHT den bpy_type (`SUBSURF`). Liste: `modifier list-available`.
- Nur **7 Primitive** (cube, sphere, cylinder, cone, plane, torus, monkey,
  empty). Import/Export von STL/glTF/FBX gibt es im Harness **nicht** → Weg 2.
- Render-Engine-Auswahl im Harness: `CYCLES | EEVEE | WORKBENCH`.
  Übergabe an blender beim Testlauf: `--overwrite` bei erneutem Render nötig.

## Weg 2: bpy-Skripte (volle Kontrolle)

Blender läuft ohne Fenster – man gibt ihm ein **Python-Skript** mit:

```bash
blender --background --python /pfad/zu/skript.py
```

Alles, was Blender kann (Modellieren, Modifier, Materialien, Rendern, Export),
wird im Skript über `bpy` angesprochen. pi schreibt so ein Skript in den
Arbeitsordner und führt es aus – **Nie per `--python-expr` lange Einzeiler,
immer richtige Skriptdateien** (lesbarer, wiederverwendbar, keine Anführungszeichen-Probleme).

## Vorlagen, die funktionieren (getestet)

### Szene: Objekte bauen + STL exportieren

```python
import bpy
bpy.ops.wm.read_factory_settings(use_empty=True)          # saubere, leere Szene

bpy.ops.mesh.primitive_cube_add(size=2, location=(0,0,0))
bpy.ops.mesh.primitive_torus_add(major_radius=1.2, minor_radius=0.3, location=(0,0,2))

bpy.ops.wm.stl_export(filepath="/pfad/ausgabe.stl")        # Blender 4.x/5.x Exporter
print("EXPORT_OK")
```

Ausführen: `blender --background --python skript.py`
Wichtige Primitive: `primitive_cube_add`, `primitive_cylinder_add(radius=…, depth=…)`,
`primitive_sphere_add`, `primitive_torus_add`, `primitive_cone_add`, `primitive_monkey_add`
(Suzanne-Testkopf), `primitive_plane_add`.

### Rendern (PNG)

```python
import bpy
bpy.ops.wm.read_factory_settings(use_empty=True)
bpy.ops.mesh.primitive_monkey_add(location=(0,0,1))
bpy.ops.object.light_add(type='SUN', location=(3,3,5))
bpy.ops.object.camera_add(location=(4,-4,3), rotation=(1.1, 0, 0.78))
bpy.context.scene.camera = bpy.context.object            # Kamera aktiv setzen!

scn = bpy.context.scene
scn.render.resolution_x = 800                             # klein halten für Tests
scn.render.resolution_y = 800
scn.render.filepath = "/pfad/render.png"
bpy.ops.render.render(write_still=True)
print("RENDER_OK")
```

- Renderer: `scn.render.engine = 'BLENDER_EEVEE_NEXT'` (schnell) oder `'CYCLES'`
  (photorealistisch, langsamer; dort zusätzlich `scn.cycles.samples = 64`).
- Am Anfang `read_factory_settings(use_empty=True)` – sonst schleppt man die
  Standard-Szene (Würfel, Licht, Kamera) herum.

### Modifier (z. B. Bohrung = Boolean)

```python
import bpy
cube = bpy.data.objects["Cube"]
cutter = bpy.data.objects["Cylinder"]
mod = cube.modifiers.new(name="Loch", type='BOOLEAN')
mod.operation = 'DIFFERENCE'
mod.object = cutter
bpy.context.view_layer.objects.active = cube
bpy.ops.object.modifier_apply(modifier=mod.name)          # anwenden (Objekt-Modus)
bpy.data.objects.remove(cutter, do_unlink=True)           # Schneidwerkzeug weg
```

### Export-Formate (Blender 4/5)

| Format | Aufruf |
|---|---|
| STL | `bpy.ops.wm.stl_export(filepath=…)` |
| OBJ | `bpy.ops.wm.obj_export(filepath=…)` |
| glTF | `bpy.ops.export_scene.gltf(filepath=…, export_format='GLTF')` |
| FBX | `bpy.ops.export_scene.fbx(filepath=…)` |
| PLY | `bpy.ops.export_mesh.ply(filepath=…)` |
| Blender-eigen | `bpy.ops.wm.save_as_mainfile(filepath=…)` (`.blend`-Datei) |

Import entsprechend: `wm.stl_import`, `wm.obj_import`, `import_scene.gltf`, …

### Vorhandene Datei laden und weiterverarbeiten

```bash
blender --background datei.blend --python skript.py
```

(Kein `read_factory_settings` im Skript, sonst wird die geladene Datei verworfen!)

## Beide Wege kombinieren

Der Harness generiert bei `render execute` / `render script` ein normales
bpy-Skript (`_render_script.py`). Man kann es als Ausgangspunkt nehmen und
vor dem Lauf per Hand erweitern (z. B. STL-Import vor den Render-Schritt
einfügen). Umgekehrt: Szene per Harness bauen, dann im generierten Skript
die Export-Ops ergänzen.

## Tipps & Stolperfallen

- **Immer `--background`** – ohne es öffnet Blender ein Fenster und blockiert.
- Objekte selektieren: `bpy.context.view_layer.objects.active = obj` und ggf.
  `obj.select_set(True)` – viele `bpy.ops.*` wirken auf die Auswahl.
- Alles, was `bpy.ops.object.*` macht, braucht den **Objekt-Modus**; nach dem
  Anlegen von Primitiven ist man schon dort.
- Materialfarbe minimal:
  ```python
  mat = bpy.data.materials.new(name="Rot")
  mat.use_nodes = True
  mat.node_tree.nodes["Principled BSDF"].inputs["Base Color"].default_value = (1, 0, 0, 1)
  obj.data.materials.append(mat)
  ```
- Rendern kann Sekunden bis Minuten dauern – beim Testen Auflösung klein (320 px) und
  `EEVEE` wählen.
- Bei Fehlern gibt Blender alles auf stderr aus; `print("MARKER")`-Zeilen im Skript
  helfen, den Fortschritt zu sehen.

## Typische pi-Aufgaben mit diesem Skill

- "Baue mir einen Würfel mit abgerundeten Kanten und exportiere STL" (drucken!) → Weg 2
- "Schnelle Vorschau-Szene mit Material + Render" → Weg 1
- "Mach ein Renderbild von dem FreeCAD-Modell" (STL importieren → hübsch rendern) → Weg 2
- "Vereinfache dieses STL" (Decimate-Modifier) → Weg 2
- "Rendere die Szene aus 4 Blickwinkeln" → Weg 1 (camera + render pro Blick) oder Weg 2
- Kombination mit FreeCAD: `cli-anything-freecad` baut das präzise CAD-Teil →
  STL → Blender macht das hübsche Bild.
