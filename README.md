# pi-cli-anything

**Headless-Steuerung von 3D/CAD-Apps für [pi](https://pi.dev)** — ein Paket, das
mit der cli-anything-Toolfamilie mitwächst. Jeder Skill ist in sich vollständig.

## Skills

| Skill | Was |
|---|---|
| `cli-anything-blender` | Blender headless: Harness-Weg (49 Commands, 10 Gruppen, stateful JSON-Szene) + bpy-Python-Skripte (Import/Export STL/OBJ/glTF/FBX, Spezial-API). Renders als PNG (EEVEE/Cycles). |
| `cli-anything-freecad` | FreeCAD headless: kompletter Harness (258 Commands: Part, Sketcher, PartDesign, Assembly, TechDraw, FEM, CAM …) + Export STEP/IGES/STL/OBJ/DXF/PDF/glTF/3MF. |

## Voraussetzungen

1. **App installieren:** Blender und/oder FreeCAD (z. B. `brew install --cask blender freecad`)
2. **Harness installieren:**
   ```bash
   uv tool install cli-anything-hub
   # Binaries dann unter ~/.local/share/uv/tools/cli-anything-hub/bin/
   ```
3. python3 mit `openpyxl` ist für einzelne Excel-Exporte hilfreich

## Installieren

```bash
pi install https://github.com/kambodscharoger294-cyber/pi-cli-anything
```

Einen Skill abschalten (z. B. FreeCAD nicht nutzen): `pi config` → Skill deaktivieren.

## Architektur-Notiz

Das **gemeinsame Kernstück** der Toolfamilie ist das uv-Tool `cli-anything-hub`
(Session-State, JSON-Protokoll, Previews) — die Skills referenzieren nur dessen
Binaries. Neue cli-anything-Tools kommen als zusätzliche `skills/cli-anything-<app>/`-
Ordner in dieses Paket, nicht als eigene Repos: pi-Skills können sich nicht
gegenseitig importieren, also bleibt jeder Skill in sich vollständig.

## Herkunft

Gepflegt im pi-Werkstatt-Starter-Kit (`skills/`), 19.09.2026 in ein eigenes
pi-Paket überführt.
