# Python GUI Development Standards

This document defines the standard approach for building desktop GUI applications in Python. All GUI projects on this system MUST follow these conventions to ensure visual consistency, maintainability, and quality.

---

## Framework: PySide6 (Qt 6)

**Always use PySide6.** No tkinter, no PyQt, no wxPython, no Kivy.

```
pip install PySide6
```

**Why PySide6:**
- Qt 6 is the most capable cross-platform GUI toolkit
- PySide6 is the official Qt for Python binding (LGPL licensed)
- Rich widget set, custom painting, stylesheets, layout managers
- Excellent macOS integration when paired with pyobjc

**Python version:** Use Python 3.13 specifically. PySide6 is installed for 3.13. The `run` script should use `#!/usr/bin/env python3.13`.

---

## Application Style

**Always set the Fusion style** at app startup for cross-platform visual consistency:

```python
app = QApplication(sys.argv)
app.setStyle("Fusion")
```

---

## Project Structure for GUI Apps

```
README.md
requirements.txt
.gitignore
run                    # Python 3.13, argparse, venv activation
assets/                # icons, images (committed to git)
  app-name-icon.png    # application icon (256x256 or larger)
src/
  app_display.py       # main GUI module (widgets, painting, layout)
  data_access.py       # data layer (API calls, DB, EventKit, etc.)
  app_display_test.py  # tests for GUI module
  data_access_test.py  # tests for data layer
output/
local/
```

**Separation of concerns:** GUI code (widgets, painting, layout) lives in one module. Data access (API, database, EventKit) lives in a separate module. The GUI module imports the data module, never the reverse.

---

## Dependencies for macOS Desktop Apps

Standard `requirements.txt` for macOS GUI apps:

```
PySide6
pyobjc-framework-Cocoa        # macOS native integration (Dock name, etc.)
setproctitle                   # meaningful process name
pytest
ruff
```

Add domain-specific pyobjc frameworks as needed:
- `pyobjc-framework-EventKit` for Calendar/Reminders
- `pyobjc-framework-Quartz` for window capture (`CGWindowListCopyWindowInfo`)

---

## Color Scheme & Theming

**Define all colors in a single COLORS dictionary** at the module level. Never scatter hex codes or RGB values through widget code.

```python
COLORS = {
    "background": QColor(243, 243, 247),
    "column_bg": QColor(255, 255, 255),
    "header_text": QColor(30, 30, 40),
    "subheader_text": QColor(100, 100, 110),
    "accent": QColor(66, 133, 244),
    "text": QColor(255, 255, 255),
    "text_muted": QColor(255, 255, 255, 200),
}
```

**Design language:** Material Design 3 inspired:
- Light, spacious layout with generous padding
- Soft drop shadows for depth (blur 16-24px, y-offset 4px, RGBA(0,0,0,25-50))
- Rounded corners on all containers (12-16px radius) and buttons (6px)
- Gradients on colored surfaces (lighter top-left to darker bottom-right, 105% lighter/darker)
- Use `QColor.lighter(105)` / `QColor.darker(105)` for subtle gradients

**Card color palettes** - when displaying lists of colored items, use a vibrant palette and assign via hash for visual distinctness:

```python
CARD_COLORS = [
    QColor(66, 133, 244),     # Google Blue
    QColor(52, 168, 83),      # Google Green
    QColor(142, 86, 232),     # Vibrant Purple
    QColor(234, 134, 64),     # Warm Orange
    QColor(0, 150, 170),      # Teal
    QColor(219, 68, 85),      # Coral Red
]

# Assign color by content hash, not index position
import hashlib
color_index = int(hashlib.md5(item_id.encode()).hexdigest(), 16) % len(CARD_COLORS)
```

---

## Typography

**Primary font: Helvetica Neue** throughout all applications.

| Component | Size | Weight | Usage |
|-----------|------|--------|-------|
| Large display | 72px | Bold | Countdown timers, hero numbers |
| Section header | 24px | Bold (700) | Column/section titles |
| Card title time | 30px | Bold | Event time, primary card info |
| Body text | 18px | Medium (500) | Titles, labels |
| Secondary text | 14-16px | Normal | Duration, notes, descriptions |
| Subtitle/caption | 12-13px | Medium (500) | Date strings, small labels |
| Badge text | 11px | Bold | Status indicators |

**Letter spacing:** -0.5px on headers for visual tightness.

```python
font = QFont("Helvetica Neue", 24, QFont.Bold)
font.setLetterSpacing(QFont.AbsoluteSpacing, -0.5)
```

---

## Window Management

**Persist window geometry** using QSettings. Save on move, resize, and close. Restore on startup.

```python
SETTINGS_ORG = "AppName"
SETTINGS_APP = "AppName"

class MainWindow(QMainWindow):
    def __init__(self):
        super().__init__()
        self.setMinimumSize(600, 500)
        settings = QSettings(SETTINGS_ORG, SETTINGS_APP)
        geometry = settings.value("geometry")
        if geometry:
            self.restoreGeometry(geometry)
        else:
            self.resize(1100, 700)

    def moveEvent(self, event):
        super().moveEvent(event)
        QSettings(SETTINGS_ORG, SETTINGS_APP).setValue("geometry", self.saveGeometry())

    def resizeEvent(self, event):
        super().resizeEvent(event)
        QSettings(SETTINGS_ORG, SETTINGS_APP).setValue("geometry", self.saveGeometry())

    def closeEvent(self, event):
        QSettings(SETTINGS_ORG, SETTINGS_APP).setValue("geometry", self.saveGeometry())
        super().closeEvent(event)
```

**Window opacity:** 0.9 (90%) for overlay-style apps that stay on top:

```python
self.setWindowOpacity(0.9)
self.setWindowFlags(self.windowFlags() | Qt.WindowStaysOnTopHint)
```

---

## Layout Patterns

**Use Qt layout managers**, not absolute positioning:
- `QHBoxLayout` for horizontal arrangements (multi-column layouts)
- `QVBoxLayout` for vertical stacks (lists, card columns)
- `QGridLayout` for grid-based layouts

**Column-based layouts** are the standard pattern:
- Fixed-width sidebar columns (e.g., 280px for a detail/status panel)
- Flexible-width main content columns
- Consistent spacing: 14px between cards, 20px margins

```python
main_layout = QHBoxLayout()
main_layout.setContentsMargins(20, 20, 20, 20)
main_layout.setSpacing(14)

sidebar = QFrame()
sidebar.setFixedWidth(280)
main_layout.addWidget(sidebar)

content = QFrame()
main_layout.addWidget(content, stretch=1)
```

---

## Custom Painting

For rich visual elements (cards, gradients, custom shapes), use `paintEvent` with `QPainter`:

```python
def paintEvent(self, event):
    painter = QPainter(self)
    painter.setRenderHint(QPainter.Antialiasing)
    painter.setRenderHint(QPainter.TextAntialiasing)

    # Rounded rectangle with gradient
    rect = self.rect().adjusted(2, 2, -2, -2)
    gradient = QLinearGradient(rect.topLeft(), rect.bottomRight())
    gradient.setColorAtPosition(0.0, base_color.lighter(105))
    gradient.setColorAtPosition(0.5, base_color)
    gradient.setColorAtPosition(1.0, base_color.darker(105))

    path = QPainterPath()
    path.addRoundedRect(QRectF(rect), 12, 12)
    painter.fillPath(path, QBrush(gradient))

    painter.end()
```

**Always enable anti-aliasing and text anti-aliasing** on every QPainter instance.

**Text wrapping** - use QFontMetrics for measuring text width and implement manual wrapping with ellipsis:

```python
def wrap_text(painter, text, font, max_width, max_lines=2):
    metrics = QFontMetrics(font)
    words = text.split()
    lines = []
    current_line = ""
    for word in words:
        test = f"{current_line} {word}".strip()
        if metrics.horizontalAdvance(test) <= max_width:
            current_line = test
        else:
            if current_line:
                lines.append(current_line)
            current_line = word
            if len(lines) >= max_lines:
                lines[-1] = metrics.elidedText(lines[-1] + " " + current_line, Qt.ElideRight, max_width)
                return lines
    if current_line:
        lines.append(current_line)
    return lines[:max_lines]
```

---

## Drop Shadows

Use `QGraphicsDropShadowEffect` for material elevation:

```python
shadow = QGraphicsDropShadowEffect()
shadow.setBlurRadius(24)       # 24 for columns, 16 for cards
shadow.setOffset(0, 4)
shadow.setColor(QColor(0, 0, 0, 25))
widget.setGraphicsEffect(shadow)
```

---

## Animations

**Flash/pulse animations** for urgent states - use a single shared QTimer, not per-widget timers:

```python
class MainWindow(QMainWindow):
    def __init__(self):
        self.flash_timer = QTimer()
        self.flash_timer.timeout.connect(self._update_flash)
        self.flash_timer.start(50)  # 50ms for smooth animation
        self.flash_phase = 0.0

    def _update_flash(self):
        self.flash_phase = (self.flash_phase + 0.025) % 1.0
        # Notify all animated widgets to repaint
        for card in self.animated_cards:
            card.update()
```

**Hue rotation** for attention-grabbing effects:

```python
hue_shift = int(self.flash_phase * 360)
h, s, lightness, a = base_color.getHsl()
new_hue = (h + hue_shift) % 360
animated_color = QColor.fromHsl(new_hue, s, lightness, a)
```

**Key principle:** One shared timer per window, not per widget. Per-widget timers accumulate during refresh cycles and cause memory issues.

---

## Data Refresh & Live Updates

**SSE (Server-Sent Events)** is the preferred pattern for live data:

```python
class DataWorker(QThread):
    data_update = Signal(dict)
    reconnecting = Signal()

    def run(self):
        while not self.isInterruptionRequested():
            try:
                self.reconnecting.emit()  # Clear stale data before reconnect
                response = requests.get(url, stream=True, timeout=30)
                for line in response.iter_lines():
                    if self.isInterruptionRequested():
                        break
                    if line.startswith(b"data: "):
                        data = json.loads(line[6:])
                        self.data_update.emit(data)
            except Exception:
                time.sleep(5)  # Reconnect delay
```

**Snapshot-then-live pattern:** When reconnecting, clear stale state BEFORE the new snapshot arrives. Wire a `reconnecting` signal that clears the UI state.

**Polling fallback:** For simpler apps, use a QTimer with 10-second refresh interval:

```python
self.refresh_timer = QTimer()
self.refresh_timer.timeout.connect(self._refresh_data)
self.refresh_timer.start(10000)
```

---

## Configuration

**Use a dataclass** for display configuration with sensible defaults:

```python
@dataclass
class DisplayConfig:
    refresh_interval_ms: int = 10000
    card_height: int = 160
    card_margin: int = 12
    column_padding: int = 20
    font_size: int = 21
    line_count: int = 12
```

---

## Stylesheet Patterns

Use Qt stylesheets for button and label styling, not custom painting:

```python
button.setStyleSheet("""
    QPushButton {
        background-color: #4285f4;
        color: white;
        border: none;
        border-radius: 6px;
        padding: 8px 20px;
        font-size: 14px;
        font-weight: bold;
    }
    QPushButton:hover {
        background-color: #3367d6;
    }
""")
```

**When to use stylesheets vs custom painting:**
- Stylesheets: buttons, labels, simple frames, text styling
- Custom painting: cards with gradients, complex shapes, animated elements, data visualization

---

## Application Icons (MANDATORY)

**Every GUI application MUST have a proper icon.** No placeholder boxes. No missing icons.

### Generating Icons

Use `~/bin/generate_image` to create application icons:

```bash
# Generate the base icon image (256x256 for app icon)
~/bin/generate_image \
    --prompt "modern flat design calendar icon, vibrant blue gradient, clean minimal style, white background" \
    --width 256 --height 256 \
    --output assets/app-name-icon.jpg

# Remove background for transparency
~/bin/remove-background assets/app-name-icon.jpg assets/app-name-icon.png
```

**Critical notes about image generation:**
- `generate_image` writes the requested output path using the canonical service artifact format; use the path it reports and verify the actual file format before any follow-up processing. Do not assume PNG requests become JPG files.
- Never use `--steps` parameter - let the tool use its default.
- Image generation takes 1-3 minutes per image. This is normal.
- Never run `generate_image` in parallel - it's too memory-intensive.
- Use saturated colored designs (not white/light text on white) - light areas get stripped by remove-background.

### Icon Dimensions Reference

| Use Case | Dimensions |
|----------|-----------|
| App icon | 256x256 |
| Favicon | 64x64 |
| System tray icon | 32x32 |
| Banner/hero | 1920x600 |
| Toolbar icon | 24x24 |
| Notification icon | 128x128 |

### Loading Icons

```python
ICON_PATH = Path(__file__).resolve().parent.parent / "assets" / "app-name-icon.png"

app = QApplication(sys.argv)
if ICON_PATH.exists():
    icon = QIcon(str(ICON_PATH))
    app.setWindowIcon(icon)
    window.setWindowIcon(icon)
```

**Always check icon file exists** before loading to avoid crashes. Store icons in `assets/` directory (committed to git). Keep a `.bak` copy for safety.

---

## macOS Integration

For macOS desktop apps, set the Dock and process name:

```python
import setproctitle
setproctitle.setproctitle("app-name")

# Set macOS Dock name (requires pyobjc-framework-Cocoa)
try:
    from Foundation import NSBundle
    bundle = NSBundle.mainBundle()
    info = bundle.localizedInfoDictionary() or bundle.infoDictionary()
    if info:
        info["CFBundleName"] = "App Name"
except ImportError:
    pass
```

---

## Dialog Patterns

**Notification/popup dialogs:**

```python
class NotificationDialog(QDialog):
    def __init__(self, parent=None):
        super().__init__(parent)
        self.setFixedSize(360, 200)
        self.setWindowFlags(
            Qt.WindowStaysOnTopHint | Qt.Dialog | Qt.WindowCloseButtonHint
        )
```

**Key patterns:**
- Fixed size for notification dialogs (360x200 is a good default)
- `WindowStaysOnTopHint` for alerts that need attention
- Track shown dialogs by key (e.g., `event_id@timestamp`) to prevent duplicates

---

## Context Menus

```python
def contextMenuEvent(self, event):
    menu = QMenu(self)
    action = menu.addAction("Open Link")
    action.triggered.connect(self._open_link)
    menu.exec(event.globalPos())
```

---

## Signal Architecture

Use Qt signals for communication between components. Never call widget methods directly across the hierarchy:

```python
class DataWorker(QThread):
    data_update = Signal(dict)
    snapshot_complete = Signal()
    reconnecting = Signal()

class MainWindow(QMainWindow):
    def __init__(self):
        self.worker = DataWorker()
        self.worker.data_update.connect(self._on_data_update)
        self.worker.reconnecting.connect(self._on_reconnecting)
```

---

## Memory & Performance

- **PySide6/Qt on macOS shows high virtual memory** (VSZ ~400GB) - this is normal Metal backend reservation. Real memory (RSS) should be <100MB.
- **One shared timer per animation type** - never create per-widget timers. They accumulate during refresh cycles.
- **Use `update()` not `repaint()`** - `update()` coalesces multiple requests into one paint event.
- **Custom paint is fast** - don't be afraid of `paintEvent` for rich visuals. It's more efficient than complex widget hierarchies.
- **Avoid creating QFont/QFontMetrics in paintEvent** - create them once and store as instance attributes.

---

## Keyboard Shortcuts

Add keyboard shortcuts for common actions:

```python
def keyPressEvent(self, event):
    if event.key() in (Qt.Key_Delete, Qt.Key_Backspace):
        self._delete_active_item()
    elif event.key() == Qt.Key_Escape:
        self._close_detail_panel()
    super().keyPressEvent(event)
```

---

## Testing GUI Code

GUI tests should verify data flow and logic, not pixel-perfect rendering:
- Test data access layer independently
- Test configuration defaults
- Test utility functions (text wrapping, color assignment, formatting)
- Test signal connections and data flow
- Use Playwright for E2E screenshot verification when needed (see screenshot skill)

---

## Checklist for Every GUI App

Before declaring a GUI app complete, verify:

1. [ ] Uses PySide6 with Fusion style
2. [ ] Has a proper application icon (generated with `generate_image` + `remove-background`)
3. [ ] Icon set on both QApplication and QMainWindow
4. [ ] All colors defined in a COLORS dictionary
5. [ ] Font is Helvetica Neue throughout
6. [ ] Window geometry persisted via QSettings
7. [ ] Process name set via setproctitle
8. [ ] macOS Dock name set via NSBundle
9. [ ] Anti-aliasing enabled on all QPainter instances
10. [ ] Drop shadows on containers for material elevation
11. [ ] Rounded corners (12-16px) on all visual containers
12. [ ] Single shared timer for animations (not per-widget)
13. [ ] Data layer separated from UI layer
14. [ ] Proper cleanup on close (timers stopped, threads interrupted)
