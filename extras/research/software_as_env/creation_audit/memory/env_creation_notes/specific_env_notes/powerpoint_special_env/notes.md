> **Note:** This file was generated against an earlier version of the gym-anything
> library. Some paths (e.g. `gym_anything/runners/...`, `examples/<env>/...`,
> `constants.py`) and APIs (e.g. `env.verify()`, `env._runner.ssh_port`) referenced
> below may have moved or been renamed. Cross-check against the current source tree
> (`src/gym_anything/...`, `benchmarks/cua_world/environments/...`,
> `env.get_session_info()`) before relying on any path or import here.

# VOID Presenter (powerpoint_special_env) - Environment Notes

## Application Overview
- **Name**: VOID Presenter ("A Presentation Organism")
- **Type**: Custom Electron app (React/Vite, frameless window)
- **Package**: `void-presenter_1.0.0_amd64.deb` (90MB), installs to `/opt/VOID/`
- **Concept**: Infinite canvas with floating slide "fragments", radial "tool constellation" menu, in-memory React state (no file save/load)

## Key Technical Decisions

### Verification via Chrome DevTools Protocol (CDP)
Since VOID stores all state in React memory (no files, no database), verification uses CDP:
1. Launch Electron with `--remote-debugging-port=9222 --remote-allow-origins='*'`
2. `cdp_extract.py` connects via websocket, evaluates JavaScript to read DOM state
3. Extracts: title, slide count, text elements, shape elements, present mode flag
4. Works in both edit mode and present mode (different DOM structure)

### Critical: `--remote-allow-origins='*'` flag
Without this flag, CDP websocket connections fail with `403 Forbidden`. This was the first major bug discovered during testing. The flag must be present in ALL launch commands (setup_task.sh, setup_void.sh, desktop entry).

### Window Offset for xdotool Coordinates
- GNOME dock: 70px left, 27px top panel
- Window position: `X=70, Y=27` (from `xdotool getactivewindow getwindowgeometry --shell`)
- CSS coordinates (from CDP getBoundingClientRect) must be offset: `screen_x = css_x + 70`, `screen_y = css_y + 27`
- ask_cua.py coordinates are in 1280x720 space, scale with `* 1920/1280` and `* 1080/720`

## Interaction Patterns Discovered

### Tool Constellation (Right-Click Radial Menu)
- **observe** - top, default active tool (selection/pointer)
- **inscribe** - top-right, text placement tool
- **manifest** - right, shape placement tool
- **portal** - bottom-right, unknown
- **spawn** - bottom, creates new slide fragment
- **replicate** - bottom-left, duplicates element
- **dissolve** - left, deletes element
- **project** - top-left, enters present mode

### Text Elements (inscribe)
1. Right-click on slide fragment to open tool constellation
2. Click "inscribe" to activate text placement mode (diamond cursor appears)
3. Click on **empty area** of a fragment to place text element
4. **CRITICAL**: Element starts with `contenteditable="false"` - must **double-click** to enter edit mode (`contenteditable="true"`)
5. Type text with xdotool
6. Press Escape or click outside to exit edit mode

### Shape Elements (manifest)
1. Right-click → click "manifest" → click on empty area of fragment
2. Default shape is always **circle** (red outline)
3. After placement, a property "halo" appears with shape variant buttons
4. Click shape buttons in halo to change type (rect, triangle, diamond, hexagon, line)

### Slide Management (spawn)
1. Right-click → click "spawn" → new fragment appears
2. Fragments are positioned on the infinite canvas
3. Navigate between fragments by scrolling/panning

### Present Mode
- Enter: `Ctrl+Enter` or use "project" tool from constellation
- Exit: `Escape` key or click "ESC - RETURN TO VOID" button
- In present mode, regular DOM elements are hidden; text/shapes rendered in `.present-mode__slide`
- Slide counter at bottom: "01 / 03" format
- Navigation arrows (← →) visible

## CDP DOM Selectors
| Element | Edit Mode Selector | Present Mode Selector |
|---------|-------------------|----------------------|
| Title | `.title-spine__title` (input) | Not visible |
| Slides | `.fragment` | `.present-mode__counter` (parse "/ N") |
| Text | `.slide-text-element div[contenteditable]` | `.present-mode__slide div` (leaf nodes) |
| Shapes | `.slide-shape-element svg` | `.present-mode__slide svg` |
| Present flag | `.present-mode` | `.present-mode` |
| Slide counter | `.title-spine__meta span` | `.present-mode__counter` |

## Known Issues & Workarounds
1. **Frameless window**: `frame: false` in Electron config. wmctrl detects window by title "VOID"
2. **Text won't type**: Must double-click element to set `contenteditable="true"` before xdotool type works
3. **Shape placement miss**: Must click on empty area of fragment, not on existing elements
4. **CDP 403**: Must use `--remote-allow-origins='*'` flag
5. **Window not maximized**: wmctrl maximize doesn't always take effect immediately; added to setup_task.sh with 1-second wait

## File Structure
```
benchmarks/cua_world/environments/powerpoint_special_env/
├── env.json                    # Base: ubuntu-gnome-systemd_highres (1920x1080)
├── scripts/
│   ├── install_void.sh         # pre_start: install .deb + deps
│   └── setup_void.sh           # post_start: create launcher + desktop entry
├── data/
│   └── void-presenter_1.0.0_amd64.deb
├── utils/
│   └── cdp_extract.py          # Shared CDP state extraction (edit + present mode)
├── tasks/
│   ├── rename_presentation/    # Easy: change title text
│   ├── add_text_element/       # Easy: add text via inscribe tool
│   ├── create_three_slides/    # Easy: spawn 2 more fragments
│   ├── add_shape_elements/     # Medium: add circle + rectangle
│   ├── build_title_slide/      # Hard: title + subtitle + shape
│   └── enter_present_mode/     # Medium: add text + Ctrl+Enter
└── evidence_docs/              # Testing screenshots
```

## Testing Evidence
- `01_app_launched.png` - VOID Presenter after launch
- `02_title_renamed.png` - Title changed to "Quarterly Report Q4"
- `03_radial_menu.png` - Tool constellation radial menu
- `04_text_added.png` - Text element "Welcome to the Future" on slide
- `05_shape_added.png` - Circle shape with property halo
- `06_present_mode.png` - Full-screen presentation mode
- `07_final_edit_mode.png` - Final state in edit mode
