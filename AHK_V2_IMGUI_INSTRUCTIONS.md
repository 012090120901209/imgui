# Building an ImGui-Style UI Library for AutoHotkey v2

A complete guide to creating an immediate-mode GUI library in pure AHK v2, inspired by Dear ImGui.

---

## Table of Contents

1. [Overview](#overview)
2. [Prerequisites](#prerequisites)
3. [Project Structure](#project-structure)
4. [Core Concepts](#core-concepts)
5. [Implementation Guide](#implementation-guide)
6. [Complete Starter Code](#complete-starter-code)
7. [Testing Your Implementation](#testing-your-implementation)
8. [Extending the Library](#extending-the-library)
9. [Troubleshooting](#troubleshooting)

---

## Overview

### What We're Building

An immediate-mode GUI library for AHK v2 that allows you to write UI code like this:

```autohotkey
#Include AhkImGui.ahk

app := AhkImGui()
app.Run(MainUI)

MainUI() {
    global myValue := 50

    if ImGui.Begin("My Tool Window") {
        ImGui.Text("Welcome to AHK ImGui!")

        if ImGui.Button("Click Me")
            MsgBox("Button was clicked!")

        if ImGui.SliderInt("Value", &myValue, 0, 100)
            ToolTip("Value changed to: " myValue)

        ImGui.Checkbox("Enable Feature", &isEnabled)

        ImGui.End()
    }
}
```

### Why Immediate Mode?

- **No callbacks**: UI state is checked directly in your code flow
- **Simple mental model**: What you see in code = what renders
- **Easy dynamic UIs**: Show/hide elements with simple `if` statements
- **Self-documenting**: UI code reads like a description of the interface

---

## Prerequisites

### Required Software

1. **AutoHotkey v2.0+**
   - Download from: https://www.autohotkey.com/
   - Verify installation: Run `AutoHotkey64.exe --version`

2. **Text Editor** (recommended)
   - VS Code with AHK v2 extension
   - Or any text editor

### Required Knowledge

- Basic AHK v2 syntax (classes, functions, Maps, Buffers)
- Understanding of Windows GDI/GDI+ concepts (helpful but not required)

---

## Project Structure

Create this folder structure:

```
AhkImGui/
├── lib/
│   ├── AhkImGui.ahk        # Main library entry point
│   ├── ImGuiContext.ahk    # State management
│   ├── ImGuiStyle.ahk      # Colors and styling
│   ├── ImGuiIO.ahk         # Input/Output handling
│   ├── ImGuiWidgets.ahk    # Widget implementations
│   ├── ImGuiDrawList.ahk   # Drawing primitives
│   └── GdipHelper.ahk      # GDI+ wrapper functions
├── examples/
│   ├── 01_basic_window.ahk
│   ├── 02_widgets_demo.ahk
│   ├── 03_multiple_windows.ahk
│   └── 04_styling.ahk
├── tests/
│   └── test_widgets.ahk
└── README.md
```

---

## Core Concepts

### 1. The Immediate Mode Loop

```
┌─────────────────────────────────────────┐
│           APPLICATION LOOP              │
│                                         │
│  1. Poll Input (mouse, keyboard)        │
│  2. NewFrame() - reset state            │
│  3. User UI Code (Begin, Button, etc.)  │
│  4. EndFrame() - finalize layout        │
│  5. Render() - draw everything          │
│  6. Repeat @ 60fps                      │
└─────────────────────────────────────────┘
```

### 2. ID System

Every widget needs a unique ID for state tracking:

```autohotkey
; Implicit ID from label
ImGui.Button("Save")        ; ID = hash("Save")
ImGui.Button("Save")        ; CONFLICT! Same ID

; Explicit ID with ##
ImGui.Button("Save##file1") ; ID = hash("##file1"), displays "Save"
ImGui.Button("Save##file2") ; ID = hash("##file2"), displays "Save"

; ID stack for loops
Loop 5 {
    ImGui.PushID(A_Index)
    ImGui.Button("Click")   ; Each gets unique ID
    ImGui.PopID()
}
```

### 3. Drawing Model

```
User Code          Draw List              Screen
─────────────────────────────────────────────────
Button("Hi")  →  [RectFilled, Text]  →  GDI+ renders
SliderFloat() →  [RectFilled, Rect]  →  to window
```

### 4. State Management

```autohotkey
; Widget state stored by ID
class WidgetState {
    static states := Map()

    static Get(id, default := 0) {
        return this.states.Has(id) ? this.states[id] : default
    }

    static Set(id, value) {
        this.states[id] := value
    }
}
```

---

## Implementation Guide

### Step 1: GDI+ Helper (lib/GdipHelper.ahk)

```autohotkey
; GDI+ Helper Functions for AHK v2
; This wraps the low-level GDI+ calls

class Gdip {
    static pToken := 0
    static hModule := 0

    static Startup() {
        this.hModule := DllCall("LoadLibrary", "Str", "gdiplus")
        si := Buffer(24, 0)
        NumPut("UInt", 1, si)
        DllCall("gdiplus\GdiplusStartup", "Ptr*", &this.pToken, "Ptr", si, "Ptr", 0)
    }

    static Shutdown() {
        DllCall("gdiplus\GdiplusShutdown", "Ptr", this.pToken)
        DllCall("FreeLibrary", "Ptr", this.hModule)
    }

    static CreateBitmap(width, height) {
        pBitmap := 0
        DllCall("gdiplus\GdipCreateBitmapFromScan0"
            , "Int", width, "Int", height
            , "Int", 0
            , "Int", 0x26200A  ; PixelFormat32bppARGB
            , "Ptr", 0
            , "Ptr*", &pBitmap)
        return pBitmap
    }

    static DeleteBitmap(pBitmap) {
        DllCall("gdiplus\GdipDisposeImage", "Ptr", pBitmap)
    }

    static GetGraphicsFromBitmap(pBitmap) {
        pGraphics := 0
        DllCall("gdiplus\GdipGetImageGraphicsContext", "Ptr", pBitmap, "Ptr*", &pGraphics)
        ; Enable anti-aliasing
        DllCall("gdiplus\GdipSetSmoothingMode", "Ptr", pGraphics, "Int", 4)
        DllCall("gdiplus\GdipSetTextRenderingHint", "Ptr", pGraphics, "Int", 5)
        return pGraphics
    }

    static DeleteGraphics(pGraphics) {
        DllCall("gdiplus\GdipDeleteGraphics", "Ptr", pGraphics)
    }

    static GraphicsClear(pGraphics, color := 0x00000000) {
        DllCall("gdiplus\GdipGraphicsClear", "Ptr", pGraphics, "UInt", color)
    }

    static CreateSolidBrush(color) {
        pBrush := 0
        DllCall("gdiplus\GdipCreateSolidFill", "UInt", color, "Ptr*", &pBrush)
        return pBrush
    }

    static DeleteBrush(pBrush) {
        DllCall("gdiplus\GdipDeleteBrush", "Ptr", pBrush)
    }

    static FillRectangle(pGraphics, pBrush, x, y, w, h) {
        DllCall("gdiplus\GdipFillRectangle"
            , "Ptr", pGraphics
            , "Ptr", pBrush
            , "Float", x, "Float", y
            , "Float", w, "Float", h)
    }

    static FillRoundedRect(pGraphics, pBrush, x, y, w, h, radius) {
        if (radius <= 0) {
            this.FillRectangle(pGraphics, pBrush, x, y, w, h)
            return
        }

        pPath := 0
        DllCall("gdiplus\GdipCreatePath", "Int", 0, "Ptr*", &pPath)

        d := radius * 2
        ; Top-left arc
        DllCall("gdiplus\GdipAddPathArc", "Ptr", pPath
            , "Float", x, "Float", y, "Float", d, "Float", d
            , "Float", 180, "Float", 90)
        ; Top-right arc
        DllCall("gdiplus\GdipAddPathArc", "Ptr", pPath
            , "Float", x + w - d, "Float", y, "Float", d, "Float", d
            , "Float", 270, "Float", 90)
        ; Bottom-right arc
        DllCall("gdiplus\GdipAddPathArc", "Ptr", pPath
            , "Float", x + w - d, "Float", y + h - d, "Float", d, "Float", d
            , "Float", 0, "Float", 90)
        ; Bottom-left arc
        DllCall("gdiplus\GdipAddPathArc", "Ptr", pPath
            , "Float", x, "Float", y + h - d, "Float", d, "Float", d
            , "Float", 90, "Float", 90)

        DllCall("gdiplus\GdipClosePathFigure", "Ptr", pPath)
        DllCall("gdiplus\GdipFillPath", "Ptr", pGraphics, "Ptr", pBrush, "Ptr", pPath)
        DllCall("gdiplus\GdipDeletePath", "Ptr", pPath)
    }

    static DrawRectangle(pGraphics, pPen, x, y, w, h) {
        DllCall("gdiplus\GdipDrawRectangle"
            , "Ptr", pGraphics
            , "Ptr", pPen
            , "Float", x, "Float", y
            , "Float", w, "Float", h)
    }

    static CreatePen(color, width := 1) {
        pPen := 0
        DllCall("gdiplus\GdipCreatePen1", "UInt", color, "Float", width, "Int", 2, "Ptr*", &pPen)
        return pPen
    }

    static DeletePen(pPen) {
        DllCall("gdiplus\GdipDeletePen", "Ptr", pPen)
    }

    static CreateFont(fontName, size, style := 0) {
        hFamily := 0
        hFont := 0
        DllCall("gdiplus\GdipCreateFontFamilyFromName"
            , "WStr", fontName
            , "Ptr", 0
            , "Ptr*", &hFamily)

        if (!hFamily) {
            ; Fallback to generic sans-serif
            DllCall("gdiplus\GdipGetGenericFontFamilySansSerif", "Ptr*", &hFamily)
        }

        DllCall("gdiplus\GdipCreateFont"
            , "Ptr", hFamily
            , "Float", size
            , "Int", style
            , "Int", 0  ; Unit: World
            , "Ptr*", &hFont)

        DllCall("gdiplus\GdipDeleteFontFamily", "Ptr", hFamily)
        return hFont
    }

    static DeleteFont(hFont) {
        DllCall("gdiplus\GdipDeleteFont", "Ptr", hFont)
    }

    static DrawString(pGraphics, str, hFont, pBrush, x, y) {
        hFormat := 0
        DllCall("gdiplus\GdipCreateStringFormat", "Int", 0, "Int", 0, "Ptr*", &hFormat)

        rc := Buffer(16)
        NumPut("Float", x, rc, 0)
        NumPut("Float", y, rc, 4)
        NumPut("Float", 10000, rc, 8)
        NumPut("Float", 10000, rc, 12)

        DllCall("gdiplus\GdipDrawString"
            , "Ptr", pGraphics
            , "WStr", str
            , "Int", -1
            , "Ptr", hFont
            , "Ptr", rc
            , "Ptr", hFormat
            , "Ptr", pBrush)

        DllCall("gdiplus\GdipDeleteStringFormat", "Ptr", hFormat)
    }

    static MeasureString(pGraphics, str, hFont) {
        hFormat := 0
        DllCall("gdiplus\GdipCreateStringFormat", "Int", 0, "Int", 0, "Ptr*", &hFormat)

        rc := Buffer(16)
        NumPut("Float", 0, rc, 0)
        NumPut("Float", 0, rc, 4)
        NumPut("Float", 10000, rc, 8)
        NumPut("Float", 10000, rc, 12)

        boundingBox := Buffer(16)

        DllCall("gdiplus\GdipMeasureString"
            , "Ptr", pGraphics
            , "WStr", str
            , "Int", -1
            , "Ptr", hFont
            , "Ptr", rc
            , "Ptr", hFormat
            , "Ptr", boundingBox
            , "Ptr", 0
            , "Ptr", 0)

        DllCall("gdiplus\GdipDeleteStringFormat", "Ptr", hFormat)

        return {
            width: NumGet(boundingBox, 8, "Float"),
            height: NumGet(boundingBox, 12, "Float")
        }
    }

    ; Update layered window from bitmap
    static UpdateLayeredWindowFromBitmap(hwnd, pBitmap, width, height) {
        hdc := DllCall("GetDC", "Ptr", hwnd, "Ptr")
        hdcMem := DllCall("CreateCompatibleDC", "Ptr", hdc, "Ptr")

        hBitmap := 0
        DllCall("gdiplus\GdipCreateHBITMAPFromBitmap", "Ptr", pBitmap, "Ptr*", &hBitmap, "UInt", 0)

        hOld := DllCall("SelectObject", "Ptr", hdcMem, "Ptr", hBitmap, "Ptr")

        pt := Buffer(8, 0)
        ptSrc := Buffer(8, 0)
        sz := Buffer(8)
        NumPut("UInt", width, sz, 0)
        NumPut("UInt", height, sz, 4)

        blend := Buffer(4)
        NumPut("UChar", 0, blend, 0)      ; BlendOp
        NumPut("UChar", 0, blend, 1)      ; BlendFlags
        NumPut("UChar", 255, blend, 2)    ; SourceConstantAlpha
        NumPut("UChar", 1, blend, 3)      ; AlphaFormat (AC_SRC_ALPHA)

        DllCall("UpdateLayeredWindow"
            , "Ptr", hwnd
            , "Ptr", hdc
            , "Ptr", 0        ; pptDst (use current position)
            , "Ptr", sz
            , "Ptr", hdcMem
            , "Ptr", ptSrc
            , "UInt", 0
            , "Ptr", blend
            , "UInt", 2)      ; ULW_ALPHA

        DllCall("SelectObject", "Ptr", hdcMem, "Ptr", hOld)
        DllCall("DeleteObject", "Ptr", hBitmap)
        DllCall("DeleteDC", "Ptr", hdcMem)
        DllCall("ReleaseDC", "Ptr", hwnd, "Ptr", hdc)
    }
}
```

### Step 2: Style Configuration (lib/ImGuiStyle.ahk)

```autohotkey
; ImGui Style Configuration
; Colors use ARGB format: 0xAARRGGBB

class ImGuiStyle {
    ; Sizing
    WindowPadding := {x: 8, y: 8}
    FramePadding := {x: 4, y: 3}
    CellPadding := {x: 4, y: 2}
    ItemSpacing := {x: 8, y: 4}
    ItemInnerSpacing := {x: 4, y: 4}
    IndentSpacing := 21
    ScrollbarSize := 14
    GrabMinSize := 12

    ; Rounding
    WindowRounding := 6
    FrameRounding := 3
    PopupRounding := 4
    ScrollbarRounding := 9
    GrabRounding := 3
    TabRounding := 4

    ; Borders
    WindowBorderSize := 1
    FrameBorderSize := 0
    PopupBorderSize := 1

    ; Font
    FontName := "Segoe UI"
    FontSize := 14

    ; Colors (Dark theme - ARGB)
    Colors := Map(
        "Text",               0xFFFFFFFF,
        "TextDisabled",       0xFF808080,
        "WindowBg",           0xF0181818,
        "ChildBg",            0x00000000,
        "PopupBg",            0xF0202020,
        "Border",             0x80606060,
        "BorderShadow",       0x00000000,
        "FrameBg",            0x8A404040,
        "FrameBgHovered",     0x99505050,
        "FrameBgActive",      0xA9606060,
        "TitleBg",            0xFF141414,
        "TitleBgActive",      0xFF1E1E1E,
        "TitleBgCollapsed",   0x82141414,
        "MenuBarBg",          0xFF242424,
        "ScrollbarBg",        0x87050505,
        "ScrollbarGrab",      0xFF4A4A4A,
        "ScrollbarGrabHovered", 0xFF5A5A5A,
        "ScrollbarGrabActive",  0xFF6A6A6A,
        "CheckMark",          0xFFFF9020,
        "SliderGrab",         0xFFB0B0B0,
        "SliderGrabActive",   0xFFFFFFFF,
        "Button",             0x66505050,
        "ButtonHovered",      0x99606060,
        "ButtonActive",       0xFFFF9020,
        "Header",             0x4DFFB020,
        "HeaderHovered",      0x80FFB020,
        "HeaderActive",       0xFFFFB020,
        "Separator",          0x80606060,
        "SeparatorHovered",   0xC0909090,
        "SeparatorActive",    0xFFB0B0B0,
        "ResizeGrip",         0x40FFFFFF,
        "ResizeGripHovered",  0xAAFFFFFF,
        "ResizeGripActive",   0xF2FFFFFF,
        "Tab",                0x97353535,
        "TabHovered",         0xFFFF9020,
        "TabActive",          0xDEFF9020,
        "TabUnfocused",       0x97202020,
        "TabUnfocusedActive", 0x9AFF9020,
        "PlotLines",          0xFF9B9B9B,
        "PlotLinesHovered",   0xFFFF6D59,
        "PlotHistogram",      0xFFE5B200,
        "PlotHistogramHovered", 0xFFFF9900,
        "TextSelectedBg",     0x59FF9020,
        "DragDropTarget",     0xFFFFFF00,
        "NavHighlight",       0xFFFF9020,
        "NavWindowingHighlight", 0xE6FFFFFF,
        "NavWindowingDimBg",  0x33CCCCCC,
        "ModalWindowDimBg",   0x59000000
    )

    ; Create a light theme
    static LightTheme() {
        style := ImGuiStyle()
        style.Colors := Map(
            "Text",               0xFF000000,
            "TextDisabled",       0xFF606060,
            "WindowBg",           0xF0F0F0F0,
            "ChildBg",            0x00000000,
            "PopupBg",            0xF0FFFFFF,
            "Border",             0x80000000,
            "FrameBg",            0xFFFFFFFF,
            "FrameBgHovered",     0xFFE8E8E8,
            "FrameBgActive",      0xFFD8D8D8,
            "TitleBg",            0xFFE0E0E0,
            "TitleBgActive",      0xFFD0D0D0,
            "TitleBgCollapsed",   0x82FFFFFF,
            "Button",             0xFFE0E0E0,
            "ButtonHovered",      0xFFD0D0D0,
            "ButtonActive",       0xFF0078D7,
            "CheckMark",          0xFF0078D7,
            "SliderGrab",         0xFF0078D7,
            "SliderGrabActive",   0xFF005A9E
            ; ... add more as needed
        )
        return style
    }
}
```

### Step 3: Input/Output Handler (lib/ImGuiIO.ahk)

```autohotkey
; ImGui Input/Output System

class ImGuiIO {
    ; Display
    DisplaySize := {x: 800, y: 600}
    DeltaTime := 0.016

    ; Input state
    MousePos := {x: 0, y: 0}
    MousePosPrev := {x: 0, y: 0}
    MouseDown := [false, false, false, false, false]
    MouseDownPrev := [false, false, false, false, false]
    MouseClicked := [false, false, false, false, false]
    MouseReleased := [false, false, false, false, false]
    MouseDoubleClicked := [false, false, false, false, false]
    MouseWheel := 0
    MouseDownDuration := [0, 0, 0, 0, 0]
    MouseClickedTime := [0, 0, 0, 0, 0]

    KeysDown := Map()
    KeysDownPrev := Map()
    KeyCtrl := false
    KeyShift := false
    KeyAlt := false

    ; Text input
    InputChars := ""

    ; Output flags
    WantCaptureMouse := false
    WantCaptureKeyboard := false
    WantTextInput := false

    ; Timing
    _lastFrameTime := 0
    _doubleClickTime := 500  ; ms

    __New() {
        this._lastFrameTime := A_TickCount
    }

    ; Call at start of each frame
    NewFrame() {
        ; Update delta time
        currentTime := A_TickCount
        this.DeltaTime := (currentTime - this._lastFrameTime) / 1000
        if (this.DeltaTime <= 0)
            this.DeltaTime := 0.016
        this._lastFrameTime := currentTime

        ; Store previous state
        this.MousePosPrev := {x: this.MousePos.x, y: this.MousePos.y}
        Loop 5 {
            this.MouseDownPrev[A_Index] := this.MouseDown[A_Index]
        }

        ; Update mouse position
        CoordMode("Mouse", "Screen")
        MouseGetPos(&mx, &my)
        this.MousePos := {x: mx, y: my}

        ; Update mouse buttons
        this.MouseDown[1] := GetKeyState("LButton", "P")
        this.MouseDown[2] := GetKeyState("RButton", "P")
        this.MouseDown[3] := GetKeyState("MButton", "P")
        this.MouseDown[4] := GetKeyState("XButton1", "P")
        this.MouseDown[5] := GetKeyState("XButton2", "P")

        ; Calculate clicked/released
        Loop 5 {
            wasDown := this.MouseDownPrev[A_Index]
            isDown := this.MouseDown[A_Index]

            this.MouseClicked[A_Index] := !wasDown && isDown
            this.MouseReleased[A_Index] := wasDown && !isDown

            ; Double click detection
            if (this.MouseClicked[A_Index]) {
                now := A_TickCount
                if (now - this.MouseClickedTime[A_Index] < this._doubleClickTime) {
                    this.MouseDoubleClicked[A_Index] := true
                } else {
                    this.MouseDoubleClicked[A_Index] := false
                }
                this.MouseClickedTime[A_Index] := now
            } else {
                this.MouseDoubleClicked[A_Index] := false
            }

            ; Track hold duration
            if (isDown) {
                this.MouseDownDuration[A_Index] += this.DeltaTime
            } else {
                this.MouseDownDuration[A_Index] := 0
            }
        }

        ; Update modifier keys
        this.KeyCtrl := GetKeyState("Ctrl", "P")
        this.KeyShift := GetKeyState("Shift", "P")
        this.KeyAlt := GetKeyState("Alt", "P")

        ; Reset wheel (should be set externally via mouse hook)
        this.MouseWheel := 0

        ; Reset output flags
        this.WantCaptureMouse := false
        this.WantCaptureKeyboard := false
    }

    ; Check if mouse is hovering a rectangle (screen coordinates)
    IsMouseHoveringRect(x1, y1, x2, y2) {
        return this.MousePos.x >= x1 && this.MousePos.x < x2
            && this.MousePos.y >= y1 && this.MousePos.y < y2
    }

    ; Get mouse delta since last frame
    GetMouseDelta() {
        return {
            x: this.MousePos.x - this.MousePosPrev.x,
            y: this.MousePos.y - this.MousePosPrev.y
        }
    }
}
```

### Step 4: Draw List (lib/ImGuiDrawList.ahk)

```autohotkey
; ImGui Draw List - collects draw commands for rendering

class ImDrawCmd {
    type := ""          ; "RectFilled", "Rect", "Text", "Line", "Circle"
    clipRect := 0       ; {x1, y1, x2, y2} or 0 for none
    data := Map()       ; Type-specific data

    __New(type, data*) {
        this.type := type
        for k, v in data {
            this.data[k] := v
        }
    }
}

class ImDrawList {
    commands := []
    _clipStack := []

    Clear() {
        this.commands := []
        this._clipStack := []
    }

    PushClipRect(x1, y1, x2, y2) {
        this._clipStack.Push({x1: x1, y1: y1, x2: x2, y2: y2})
    }

    PopClipRect() {
        if (this._clipStack.Length > 0)
            this._clipStack.Pop()
    }

    GetCurrentClipRect() {
        if (this._clipStack.Length > 0)
            return this._clipStack[-1]
        return 0
    }

    AddRectFilled(x1, y1, x2, y2, color, rounding := 0, roundingFlags := 0xF) {
        cmd := ImDrawCmd("RectFilled")
        cmd.data["x1"] := x1
        cmd.data["y1"] := y1
        cmd.data["x2"] := x2
        cmd.data["y2"] := y2
        cmd.data["color"] := color
        cmd.data["rounding"] := rounding
        cmd.data["roundingFlags"] := roundingFlags
        cmd.clipRect := this.GetCurrentClipRect()
        this.commands.Push(cmd)
    }

    AddRect(x1, y1, x2, y2, color, thickness := 1, rounding := 0) {
        cmd := ImDrawCmd("Rect")
        cmd.data["x1"] := x1
        cmd.data["y1"] := y1
        cmd.data["x2"] := x2
        cmd.data["y2"] := y2
        cmd.data["color"] := color
        cmd.data["thickness"] := thickness
        cmd.data["rounding"] := rounding
        cmd.clipRect := this.GetCurrentClipRect()
        this.commands.Push(cmd)
    }

    AddText(x, y, text, color, font := 0) {
        cmd := ImDrawCmd("Text")
        cmd.data["x"] := x
        cmd.data["y"] := y
        cmd.data["text"] := text
        cmd.data["color"] := color
        cmd.data["font"] := font
        cmd.clipRect := this.GetCurrentClipRect()
        this.commands.Push(cmd)
    }

    AddLine(x1, y1, x2, y2, color, thickness := 1) {
        cmd := ImDrawCmd("Line")
        cmd.data["x1"] := x1
        cmd.data["y1"] := y1
        cmd.data["x2"] := x2
        cmd.data["y2"] := y2
        cmd.data["color"] := color
        cmd.data["thickness"] := thickness
        cmd.clipRect := this.GetCurrentClipRect()
        this.commands.Push(cmd)
    }

    AddCircleFilled(cx, cy, radius, color, numSegments := 0) {
        cmd := ImDrawCmd("CircleFilled")
        cmd.data["cx"] := cx
        cmd.data["cy"] := cy
        cmd.data["radius"] := radius
        cmd.data["color"] := color
        cmd.data["segments"] := numSegments
        cmd.clipRect := this.GetCurrentClipRect()
        this.commands.Push(cmd)
    }

    AddTriangleFilled(x1, y1, x2, y2, x3, y3, color) {
        cmd := ImDrawCmd("TriangleFilled")
        cmd.data["x1"] := x1
        cmd.data["y1"] := y1
        cmd.data["x2"] := x2
        cmd.data["y2"] := y2
        cmd.data["x3"] := x3
        cmd.data["y3"] := y3
        cmd.data["color"] := color
        cmd.clipRect := this.GetCurrentClipRect()
        this.commands.Push(cmd)
    }
}
```

### Step 5: Context Management (lib/ImGuiContext.ahk)

```autohotkey
; ImGui Context - Central state management

class ImGuiWindow {
    name := ""
    id := 0
    pos := {x: 0, y: 0}
    size := {x: 0, y: 0}
    contentSize := {x: 0, y: 0}
    scroll := {x: 0, y: 0}
    collapsed := false
    flags := 0

    ; Layout state
    cursorPos := {x: 0, y: 0}
    cursorMaxPos := {x: 0, y: 0}
    cursorStartPos := {x: 0, y: 0}

    ; For dragging/resizing
    isDragging := false
    isResizing := false
    dragOffset := {x: 0, y: 0}

    __New(name, id) {
        this.name := name
        this.id := id
    }
}

class ImGuiContext {
    ; Global state
    io := ImGuiIO()
    style := ImGuiStyle()
    drawList := ImDrawList()

    ; Font resources
    hFont := 0
    pGraphics := 0

    ; Window system
    windows := Map()           ; id -> ImGuiWindow
    windowStack := []          ; Current window stack
    currentWindow := 0         ; Active window
    windowFocusOrder := []     ; Z-order for windows
    hoveredWindow := 0
    movingWindow := 0
    resizingWindow := 0

    ; ID system
    idStack := []

    ; Widget state
    activeId := 0              ; Currently active widget (being interacted with)
    hoveredId := 0             ; Currently hovered widget
    lastActiveId := 0

    ; Persistent widget state (survives frames)
    widgetState := Map()       ; id -> state data

    ; Popup/Modal state
    openPopupId := 0
    modalStack := []

    ; Frame state
    frameCount := 0

    __New() {
        ; Initialize GDI+
        Gdip.Startup()

        ; Create font
        this.hFont := Gdip.CreateFont(this.style.FontName, this.style.FontSize)
    }

    __Delete() {
        if (this.hFont)
            Gdip.DeleteFont(this.hFont)
        Gdip.Shutdown()
    }

    ; ID System
    GetID(strOrInt) {
        seed := this.idStack.Length > 0 ? this.idStack[-1] : 0

        if (strOrInt is Integer)
            return this.HashCombine(seed, strOrInt)

        ; String - check for ## separator
        str := String(strOrInt)
        hashPos := InStr(str, "##")
        if (hashPos > 0)
            str := SubStr(str, hashPos)

        return this.HashString(str, seed)
    }

    PushID(id) {
        newId := this.GetID(id)
        this.idStack.Push(newId)
    }

    PopID() {
        if (this.idStack.Length > 0)
            this.idStack.Pop()
    }

    HashString(str, seed := 0) {
        ; FNV-1a hash
        hash := seed ? seed : 2166136261
        Loop Parse, str {
            hash := hash ^ Ord(A_LoopField)
            hash := (hash * 16777619) & 0xFFFFFFFF
        }
        return hash
    }

    HashCombine(a, b) {
        return ((a * 16777619) ^ b) & 0xFFFFFFFF
    }

    ; State management
    GetWidgetState(id, key, default := "") {
        if (!this.widgetState.Has(id))
            return default
        state := this.widgetState[id]
        return state.Has(key) ? state[key] : default
    }

    SetWidgetState(id, key, value) {
        if (!this.widgetState.Has(id))
            this.widgetState[id] := Map()
        this.widgetState[id][key] := value
    }

    ; Frame management
    NewFrame() {
        this.frameCount++
        this.io.NewFrame()
        this.drawList.Clear()
        this.hoveredId := 0
        this.hoveredWindow := 0
        this.windowStack := []
        this.currentWindow := 0
    }

    EndFrame() {
        ; Process any end-of-frame logic
    }

    ; Text measurement helper
    CalcTextSize(text) {
        ; We need a temporary graphics context for measurement
        ; This is a simplified version - in production, cache this
        pBitmap := Gdip.CreateBitmap(1, 1)
        pGraphics := Gdip.GetGraphicsFromBitmap(pBitmap)
        size := Gdip.MeasureString(pGraphics, text, this.hFont)
        Gdip.DeleteGraphics(pGraphics)
        Gdip.DeleteBitmap(pBitmap)
        return size
    }
}

; Global context instance
global g_ImGuiCtx := 0
```

### Step 6: Main Library Entry Point (lib/AhkImGui.ahk)

```autohotkey
; AhkImGui - Main Library Entry Point
; AutoHotkey v2 Immediate Mode GUI Library

#Requires AutoHotkey v2.0

#Include GdipHelper.ahk
#Include ImGuiStyle.ahk
#Include ImGuiIO.ahk
#Include ImGuiDrawList.ahk
#Include ImGuiContext.ahk
#Include ImGuiWidgets.ahk

class AhkImGui {
    hwnd := 0
    ctx := 0
    width := 800
    height := 600
    running := false
    userCallback := 0

    ; Rendering resources
    pBitmap := 0
    pGraphics := 0
    brushCache := Map()

    __New(width := 800, height := 600, title := "AhkImGui Window") {
        this.width := width
        this.height := height

        ; Create context
        this.ctx := ImGuiContext()
        this.ctx.io.DisplaySize := {x: width, y: height}
        global g_ImGuiCtx := this.ctx

        ; Create layered window (for alpha transparency)
        this.hwnd := this.CreateLayeredWindow(title, width, height)

        ; Create rendering resources
        this.pBitmap := Gdip.CreateBitmap(width, height)
        this.pGraphics := Gdip.GetGraphicsFromBitmap(this.pBitmap)
    }

    __Delete() {
        this.Cleanup()
    }

    CreateLayeredWindow(title, w, h) {
        ; WS_EX_LAYERED = 0x80000, WS_EX_TOPMOST = 0x8
        exStyle := 0x80000

        ; Create GUI
        myGui := Gui("+AlwaysOnTop -Caption +E" exStyle)
        myGui.Title := title
        myGui.Show("w" w " h" h)

        ; Store reference to prevent GC
        this._gui := myGui

        return myGui.Hwnd
    }

    Run(callback) {
        this.userCallback := callback
        this.running := true

        ; Set up mouse wheel hook
        OnMessage(0x20A, this.OnMouseWheel.Bind(this))  ; WM_MOUSEWHEEL

        ; Main loop using timer (non-blocking)
        SetTimer(this.MainLoop.Bind(this), 16)  ; ~60 FPS

        ; Keep script running
        Persistent(true)
    }

    Stop() {
        this.running := false
        SetTimer(this.MainLoop.Bind(this), 0)
    }

    MainLoop() {
        if (!this.running)
            return

        ; Begin frame
        this.ctx.NewFrame()

        ; Update window position for input
        try {
            WinGetPos(&wx, &wy,,, "ahk_id " this.hwnd)
            this.ctx.io.MousePos.x -= wx
            this.ctx.io.MousePos.y -= wy
        }

        ; Call user UI code
        if (this.userCallback)
            this.userCallback()

        ; End frame
        this.ctx.EndFrame()

        ; Render
        this.Render()
    }

    Render() {
        ; Clear bitmap
        Gdip.GraphicsClear(this.pGraphics, 0x00000000)

        ; Execute draw commands
        for cmd in this.ctx.drawList.commands {
            this.ExecuteDrawCommand(cmd)
        }

        ; Update window
        Gdip.UpdateLayeredWindowFromBitmap(this.hwnd, this.pBitmap, this.width, this.height)
    }

    ExecuteDrawCommand(cmd) {
        switch cmd.type {
            case "RectFilled":
                brush := this.GetBrush(cmd.data["color"])
                Gdip.FillRoundedRect(this.pGraphics, brush,
                    cmd.data["x1"], cmd.data["y1"],
                    cmd.data["x2"] - cmd.data["x1"],
                    cmd.data["y2"] - cmd.data["y1"],
                    cmd.data["rounding"])

            case "Rect":
                pen := Gdip.CreatePen(cmd.data["color"], cmd.data["thickness"])
                ; TODO: Implement rounded rect outline
                Gdip.DrawRectangle(this.pGraphics, pen,
                    cmd.data["x1"], cmd.data["y1"],
                    cmd.data["x2"] - cmd.data["x1"],
                    cmd.data["y2"] - cmd.data["y1"])
                Gdip.DeletePen(pen)

            case "Text":
                brush := this.GetBrush(cmd.data["color"])
                Gdip.DrawString(this.pGraphics, cmd.data["text"],
                    this.ctx.hFont, brush,
                    cmd.data["x"], cmd.data["y"])

            case "Line":
                pen := Gdip.CreatePen(cmd.data["color"], cmd.data["thickness"])
                DllCall("gdiplus\GdipDrawLine", "Ptr", this.pGraphics, "Ptr", pen
                    , "Float", cmd.data["x1"], "Float", cmd.data["y1"]
                    , "Float", cmd.data["x2"], "Float", cmd.data["y2"])
                Gdip.DeletePen(pen)

            case "CircleFilled":
                brush := this.GetBrush(cmd.data["color"])
                r := cmd.data["radius"]
                DllCall("gdiplus\GdipFillEllipse", "Ptr", this.pGraphics, "Ptr", brush
                    , "Float", cmd.data["cx"] - r
                    , "Float", cmd.data["cy"] - r
                    , "Float", r * 2, "Float", r * 2)

            case "TriangleFilled":
                ; Create path for triangle
                brush := this.GetBrush(cmd.data["color"])
                pPath := 0
                DllCall("gdiplus\GdipCreatePath", "Int", 0, "Ptr*", &pPath)

                points := Buffer(24)  ; 3 points * 8 bytes each
                NumPut("Float", cmd.data["x1"], points, 0)
                NumPut("Float", cmd.data["y1"], points, 4)
                NumPut("Float", cmd.data["x2"], points, 8)
                NumPut("Float", cmd.data["y2"], points, 12)
                NumPut("Float", cmd.data["x3"], points, 16)
                NumPut("Float", cmd.data["y3"], points, 20)

                DllCall("gdiplus\GdipAddPathPolygon", "Ptr", pPath, "Ptr", points, "Int", 3)
                DllCall("gdiplus\GdipFillPath", "Ptr", this.pGraphics, "Ptr", brush, "Ptr", pPath)
                DllCall("gdiplus\GdipDeletePath", "Ptr", pPath)
        }
    }

    GetBrush(color) {
        if (!this.brushCache.Has(color)) {
            this.brushCache[color] := Gdip.CreateSolidBrush(color)
        }
        return this.brushCache[color]
    }

    OnMouseWheel(wParam, lParam, msg, hwnd) {
        delta := (wParam >> 16) & 0xFFFF
        if (delta > 32767)
            delta -= 65536
        this.ctx.io.MouseWheel := delta / 120
    }

    Cleanup() {
        ; Clean up brushes
        for color, brush in this.brushCache {
            Gdip.DeleteBrush(brush)
        }
        this.brushCache := Map()

        ; Clean up rendering resources
        if (this.pGraphics)
            Gdip.DeleteGraphics(this.pGraphics)
        if (this.pBitmap)
            Gdip.DeleteBitmap(this.pBitmap)
    }
}

; Global ImGui API (static functions for user convenience)
class ImGui {
    ; Window
    static Begin(name, p_open := 0, flags := 0) => ImGuiWidgets.Begin(name, p_open, flags)
    static End() => ImGuiWidgets.End()

    ; Widgets - Basic
    static Text(text) => ImGuiWidgets.Text(text)
    static TextColored(color, text) => ImGuiWidgets.TextColored(color, text)
    static Button(label, width := 0, height := 0) => ImGuiWidgets.Button(label, width, height)
    static SmallButton(label) => ImGuiWidgets.SmallButton(label)
    static Checkbox(label, &value) => ImGuiWidgets.Checkbox(label, &value)
    static RadioButton(label, &value, buttonValue) => ImGuiWidgets.RadioButton(label, &value, buttonValue)

    ; Widgets - Sliders/Drags
    static SliderInt(label, &value, min, max, format := "%d") => ImGuiWidgets.SliderInt(label, &value, min, max, format)
    static SliderFloat(label, &value, min, max, format := "%.3f") => ImGuiWidgets.SliderFloat(label, &value, min, max, format)
    static DragInt(label, &value, speed := 1, min := 0, max := 0) => ImGuiWidgets.DragInt(label, &value, speed, min, max)
    static DragFloat(label, &value, speed := 1.0, min := 0.0, max := 0.0) => ImGuiWidgets.DragFloat(label, &value, speed, min, max)

    ; Widgets - Input
    static InputText(label, &value, flags := 0) => ImGuiWidgets.InputText(label, &value, flags)
    static InputInt(label, &value) => ImGuiWidgets.InputInt(label, &value)
    static InputFloat(label, &value, step := 0.0, stepFast := 0.0) => ImGuiWidgets.InputFloat(label, &value, step, stepFast)

    ; Widgets - Combo/List
    static BeginCombo(label, previewValue, flags := 0) => ImGuiWidgets.BeginCombo(label, previewValue, flags)
    static EndCombo() => ImGuiWidgets.EndCombo()
    static Selectable(label, selected := false, flags := 0) => ImGuiWidgets.Selectable(label, selected, flags)

    ; Widgets - Trees
    static TreeNode(label) => ImGuiWidgets.TreeNode(label)
    static TreePop() => ImGuiWidgets.TreePop()
    static CollapsingHeader(label, flags := 0) => ImGuiWidgets.CollapsingHeader(label, flags)

    ; Layout
    static Separator() => ImGuiWidgets.Separator()
    static SameLine(offsetX := 0, spacing := -1) => ImGuiWidgets.SameLine(offsetX, spacing)
    static NewLine() => ImGuiWidgets.NewLine()
    static Spacing() => ImGuiWidgets.Spacing()
    static Indent(width := 0) => ImGuiWidgets.Indent(width)
    static Unindent(width := 0) => ImGuiWidgets.Unindent(width)

    ; ID Stack
    static PushID(id) => g_ImGuiCtx.PushID(id)
    static PopID() => g_ImGuiCtx.PopID()

    ; Utilities
    static IsItemHovered() => ImGuiWidgets.IsItemHovered()
    static IsItemActive() => ImGuiWidgets.IsItemActive()
    static IsItemClicked(button := 0) => ImGuiWidgets.IsItemClicked(button)
    static SetNextWindowPos(x, y, cond := 0) => ImGuiWidgets.SetNextWindowPos(x, y, cond)
    static SetNextWindowSize(w, h, cond := 0) => ImGuiWidgets.SetNextWindowSize(w, h, cond)

    ; Style
    static PushStyleColor(idx, color) => ImGuiWidgets.PushStyleColor(idx, color)
    static PopStyleColor(count := 1) => ImGuiWidgets.PopStyleColor(count)
}
```

### Step 7: Widget Implementations (lib/ImGuiWidgets.ahk)

```autohotkey
; ImGui Widget Implementations

class ImGuiWidgets {
    ; State for "last item"
    static lastItemRect := {x1: 0, y1: 0, x2: 0, y2: 0}
    static lastItemId := 0
    static lastItemHovered := false
    static lastItemActive := false

    ; Next window settings
    static nextWindowPos := 0
    static nextWindowSize := 0

    ; Style stack
    static colorStack := []

    ; Combo state
    static comboPopupOpen := false
    static comboPopupId := 0

    ; ============== WINDOW ==============

    static Begin(name, p_open := 0, flags := 0) {
        ctx := g_ImGuiCtx
        id := ctx.GetID(name)

        ; Get or create window
        if (!ctx.windows.Has(id)) {
            win := ImGuiWindow(name, id)
            win.pos := this.nextWindowPos ? this.nextWindowPos : {x: 50 + ctx.windows.Count * 20, y: 50 + ctx.windows.Count * 20}
            win.size := this.nextWindowSize ? this.nextWindowSize : {x: 300, y: 200}
            ctx.windows[id] := win
        }

        win := ctx.windows[id]

        ; Clear next window settings
        this.nextWindowPos := 0
        this.nextWindowSize := 0

        ; Push to window stack
        ctx.windowStack.Push(win)
        ctx.currentWindow := win
        ctx.PushID(id)

        style := ctx.style
        io := ctx.io
        dl := ctx.drawList

        ; Check window hover
        isHovered := io.IsMouseHoveringRect(win.pos.x, win.pos.y,
            win.pos.x + win.size.x, win.pos.y + win.size.y)

        if (isHovered)
            ctx.hoveredWindow := win

        ; Title bar dimensions
        titleBarHeight := 22
        titleBarRect := {
            x1: win.pos.x,
            y1: win.pos.y,
            x2: win.pos.x + win.size.x,
            y2: win.pos.y + titleBarHeight
        }

        ; Handle dragging
        titleHovered := io.IsMouseHoveringRect(titleBarRect.x1, titleBarRect.y1,
            titleBarRect.x2, titleBarRect.y2)

        if (titleHovered && io.MouseClicked[1]) {
            win.isDragging := true
            win.dragOffset := {x: io.MousePos.x - win.pos.x, y: io.MousePos.y - win.pos.y}
            ctx.movingWindow := win
        }

        if (win.isDragging) {
            if (io.MouseDown[1]) {
                win.pos.x := io.MousePos.x - win.dragOffset.x
                win.pos.y := io.MousePos.y - win.dragOffset.y
            } else {
                win.isDragging := false
                ctx.movingWindow := 0
            }
        }

        ; Handle resizing (bottom-right corner)
        resizeGripSize := 16
        resizeRect := {
            x1: win.pos.x + win.size.x - resizeGripSize,
            y1: win.pos.y + win.size.y - resizeGripSize,
            x2: win.pos.x + win.size.x,
            y2: win.pos.y + win.size.y
        }

        resizeHovered := io.IsMouseHoveringRect(resizeRect.x1, resizeRect.y1,
            resizeRect.x2, resizeRect.y2)

        if (resizeHovered && io.MouseClicked[1]) {
            win.isResizing := true
            ctx.resizingWindow := win
        }

        if (win.isResizing) {
            if (io.MouseDown[1]) {
                win.size.x := Max(100, io.MousePos.x - win.pos.x)
                win.size.y := Max(50, io.MousePos.y - win.pos.y)
            } else {
                win.isResizing := false
                ctx.resizingWindow := 0
            }
        }

        ; Draw window background
        dl.AddRectFilled(win.pos.x, win.pos.y,
            win.pos.x + win.size.x, win.pos.y + win.size.y,
            style.Colors["WindowBg"], style.WindowRounding)

        ; Draw border
        dl.AddRect(win.pos.x, win.pos.y,
            win.pos.x + win.size.x, win.pos.y + win.size.y,
            style.Colors["Border"], 1, style.WindowRounding)

        ; Draw title bar
        dl.AddRectFilled(titleBarRect.x1, titleBarRect.y1,
            titleBarRect.x2, titleBarRect.y2,
            style.Colors["TitleBgActive"], style.WindowRounding, 0x3)  ; Top corners only

        ; Draw title text
        dl.AddText(win.pos.x + style.WindowPadding.x, win.pos.y + 3,
            name, style.Colors["Text"])

        ; Draw resize grip
        gripColor := resizeHovered ? style.Colors["ResizeGripHovered"] : style.Colors["ResizeGrip"]
        dl.AddTriangleFilled(
            resizeRect.x2, resizeRect.y2,
            resizeRect.x1, resizeRect.y2,
            resizeRect.x2, resizeRect.y1,
            gripColor)

        ; Set cursor start position (content area)
        win.cursorStartPos := {
            x: win.pos.x + style.WindowPadding.x,
            y: win.pos.y + titleBarHeight + style.WindowPadding.y
        }
        win.cursorPos := {x: win.cursorStartPos.x, y: win.cursorStartPos.y}
        win.cursorMaxPos := {x: win.cursorPos.x, y: win.cursorPos.y}

        ; Set up clipping for content
        dl.PushClipRect(
            win.pos.x, win.pos.y + titleBarHeight,
            win.pos.x + win.size.x, win.pos.y + win.size.y)

        return true
    }

    static End() {
        ctx := g_ImGuiCtx

        if (ctx.windowStack.Length > 0) {
            ctx.drawList.PopClipRect()
            ctx.PopID()
            ctx.windowStack.Pop()
            ctx.currentWindow := ctx.windowStack.Length > 0 ? ctx.windowStack[-1] : 0
        }
    }

    ; ============== TEXT ==============

    static Text(text) {
        ctx := g_ImGuiCtx
        win := ctx.currentWindow
        if (!win)
            return

        style := ctx.style
        textSize := ctx.CalcTextSize(text)

        ctx.drawList.AddText(win.cursorPos.x, win.cursorPos.y, text, style.Colors["Text"])

        ; Update last item
        this.lastItemRect := {
            x1: win.cursorPos.x,
            y1: win.cursorPos.y,
            x2: win.cursorPos.x + textSize.width,
            y2: win.cursorPos.y + textSize.height
        }

        ; Advance cursor
        win.cursorPos.y += textSize.height + style.ItemSpacing.y
        win.cursorMaxPos.x := Max(win.cursorMaxPos.x, this.lastItemRect.x2)
        win.cursorMaxPos.y := Max(win.cursorMaxPos.y, this.lastItemRect.y2)
    }

    static TextColored(color, text) {
        ctx := g_ImGuiCtx
        win := ctx.currentWindow
        if (!win)
            return

        style := ctx.style
        textSize := ctx.CalcTextSize(text)

        ctx.drawList.AddText(win.cursorPos.x, win.cursorPos.y, text, color)

        win.cursorPos.y += textSize.height + style.ItemSpacing.y
    }

    ; ============== BUTTON ==============

    static Button(label, width := 0, height := 0) {
        ctx := g_ImGuiCtx
        win := ctx.currentWindow
        if (!win)
            return false

        style := ctx.style
        io := ctx.io
        id := ctx.GetID(label)

        ; Calculate button size
        textSize := ctx.CalcTextSize(label)
        btnWidth := width > 0 ? width : textSize.width + style.FramePadding.x * 2
        btnHeight := height > 0 ? height : textSize.height + style.FramePadding.y * 2

        ; Button rect
        x1 := win.cursorPos.x
        y1 := win.cursorPos.y
        x2 := x1 + btnWidth
        y2 := y1 + btnHeight

        ; Interaction
        hovered := io.IsMouseHoveringRect(x1, y1, x2, y2)
        held := hovered && io.MouseDown[1]
        pressed := hovered && io.MouseClicked[1]

        ; Determine color
        if (held)
            bgColor := style.Colors["ButtonActive"]
        else if (hovered)
            bgColor := style.Colors["ButtonHovered"]
        else
            bgColor := style.Colors["Button"]

        ; Draw
        ctx.drawList.AddRectFilled(x1, y1, x2, y2, bgColor, style.FrameRounding)

        ; Center text
        textX := x1 + (btnWidth - textSize.width) / 2
        textY := y1 + (btnHeight - textSize.height) / 2
        ctx.drawList.AddText(textX, textY, label, style.Colors["Text"])

        ; Update state
        this.lastItemRect := {x1: x1, y1: y1, x2: x2, y2: y2}
        this.lastItemId := id
        this.lastItemHovered := hovered
        this.lastItemActive := held

        if (hovered)
            ctx.hoveredId := id
        if (held)
            ctx.activeId := id

        ; Advance cursor
        win.cursorPos.y += btnHeight + style.ItemSpacing.y

        return pressed
    }

    static SmallButton(label) {
        ctx := g_ImGuiCtx
        ; Temporarily reduce frame padding
        oldPadding := ctx.style.FramePadding
        ctx.style.FramePadding := {x: 2, y: 1}
        result := this.Button(label)
        ctx.style.FramePadding := oldPadding
        return result
    }

    ; ============== CHECKBOX ==============

    static Checkbox(label, &value) {
        ctx := g_ImGuiCtx
        win := ctx.currentWindow
        if (!win)
            return false

        style := ctx.style
        io := ctx.io
        id := ctx.GetID(label)

        ; Checkbox size
        boxSize := ctx.style.FontSize
        textSize := ctx.CalcTextSize(label)

        ; Calculate total size
        totalWidth := boxSize + style.ItemInnerSpacing.x + textSize.width
        totalHeight := Max(boxSize, textSize.height)

        ; Rects
        x1 := win.cursorPos.x
        y1 := win.cursorPos.y
        boxX2 := x1 + boxSize
        boxY2 := y1 + boxSize

        ; Interaction (entire label area is clickable)
        hovered := io.IsMouseHoveringRect(x1, y1, x1 + totalWidth, y1 + totalHeight)
        clicked := hovered && io.MouseClicked[1]

        if (clicked)
            value := !value

        ; Draw box background
        bgColor := hovered ? style.Colors["FrameBgHovered"] : style.Colors["FrameBg"]
        ctx.drawList.AddRectFilled(x1, y1, boxX2, boxY2, bgColor, style.FrameRounding)
        ctx.drawList.AddRect(x1, y1, boxX2, boxY2, style.Colors["Border"], 1, style.FrameRounding)

        ; Draw checkmark if checked
        if (value) {
            ; Simple checkmark using lines
            pad := 3
            ctx.drawList.AddLine(x1 + pad, y1 + boxSize/2, x1 + boxSize/2 - 1, boxY2 - pad, style.Colors["CheckMark"], 2)
            ctx.drawList.AddLine(x1 + boxSize/2 - 1, boxY2 - pad, boxX2 - pad, y1 + pad, style.Colors["CheckMark"], 2)
        }

        ; Draw label
        textX := boxX2 + style.ItemInnerSpacing.x
        textY := y1 + (boxSize - textSize.height) / 2
        ctx.drawList.AddText(textX, textY, label, style.Colors["Text"])

        ; Update state
        this.lastItemRect := {x1: x1, y1: y1, x2: x1 + totalWidth, y2: y1 + totalHeight}
        this.lastItemId := id
        this.lastItemHovered := hovered

        ; Advance cursor
        win.cursorPos.y += totalHeight + style.ItemSpacing.y

        return clicked
    }

    ; ============== RADIO BUTTON ==============

    static RadioButton(label, &value, buttonValue) {
        ctx := g_ImGuiCtx
        win := ctx.currentWindow
        if (!win)
            return false

        style := ctx.style
        io := ctx.io
        id := ctx.GetID(label)

        ; Radio size
        radius := ctx.style.FontSize / 2
        textSize := ctx.CalcTextSize(label)

        totalWidth := radius * 2 + style.ItemInnerSpacing.x + textSize.width
        totalHeight := Max(radius * 2, textSize.height)

        x1 := win.cursorPos.x
        y1 := win.cursorPos.y
        cx := x1 + radius
        cy := y1 + radius

        ; Interaction
        hovered := io.IsMouseHoveringRect(x1, y1, x1 + totalWidth, y1 + totalHeight)
        clicked := hovered && io.MouseClicked[1]

        if (clicked)
            value := buttonValue

        isSelected := (value == buttonValue)

        ; Draw outer circle
        bgColor := hovered ? style.Colors["FrameBgHovered"] : style.Colors["FrameBg"]
        ctx.drawList.AddCircleFilled(cx, cy, radius, bgColor)

        ; Draw inner circle if selected
        if (isSelected) {
            ctx.drawList.AddCircleFilled(cx, cy, radius - 3, style.Colors["CheckMark"])
        }

        ; Draw label
        textX := x1 + radius * 2 + style.ItemInnerSpacing.x
        textY := y1 + (radius * 2 - textSize.height) / 2
        ctx.drawList.AddText(textX, textY, label, style.Colors["Text"])

        ; Advance cursor
        win.cursorPos.y += totalHeight + style.ItemSpacing.y

        return clicked && isSelected
    }

    ; ============== SLIDER ==============

    static SliderInt(label, &value, min, max, format := "%d") {
        return this._SliderScalar(label, &value, min, max, format, true)
    }

    static SliderFloat(label, &value, min, max, format := "%.3f") {
        return this._SliderScalar(label, &value, min, max, format, false)
    }

    static _SliderScalar(label, &value, min, max, format, isInt) {
        ctx := g_ImGuiCtx
        win := ctx.currentWindow
        if (!win)
            return false

        style := ctx.style
        io := ctx.io
        id := ctx.GetID(label)

        ; Layout
        labelSize := ctx.CalcTextSize(label)
        sliderWidth := 150
        sliderHeight := ctx.style.FontSize + style.FramePadding.y * 2

        ; Slider track rect
        x1 := win.cursorPos.x
        y1 := win.cursorPos.y
        x2 := x1 + sliderWidth
        y2 := y1 + sliderHeight

        ; Interaction
        hovered := io.IsMouseHoveringRect(x1, y1, x2, y2)

        ; Start dragging
        if (hovered && io.MouseClicked[1]) {
            ctx.activeId := id
        }

        ; Handle dragging
        valueChanged := false
        if (ctx.activeId == id) {
            if (io.MouseDown[1]) {
                ; Calculate new value from mouse position
                t := Clamp((io.MousePos.x - x1) / (x2 - x1), 0, 1)
                newValue := min + t * (max - min)

                if (isInt)
                    newValue := Round(newValue)

                if (newValue != value) {
                    value := newValue
                    valueChanged := true
                }
            } else {
                ctx.activeId := 0
            }
        }

        ; Clamp value
        value := Clamp(value, min, max)

        ; Calculate grab position
        t := (max > min) ? (value - min) / (max - min) : 0
        grabWidth := Max(style.GrabMinSize, sliderHeight * 0.8)
        grabX := x1 + t * (sliderWidth - grabWidth)

        ; Draw track
        bgColor := (hovered || ctx.activeId == id) ? style.Colors["FrameBgHovered"] : style.Colors["FrameBg"]
        ctx.drawList.AddRectFilled(x1, y1, x2, y2, bgColor, style.FrameRounding)

        ; Draw grab
        grabColor := (ctx.activeId == id) ? style.Colors["SliderGrabActive"] : style.Colors["SliderGrab"]
        ctx.drawList.AddRectFilled(grabX, y1 + 2, grabX + grabWidth, y2 - 2, grabColor, style.GrabRounding)

        ; Draw value text centered
        valueText := Format(format, value)
        valueSize := ctx.CalcTextSize(valueText)
        ctx.drawList.AddText(x1 + (sliderWidth - valueSize.width) / 2,
            y1 + (sliderHeight - valueSize.height) / 2,
            valueText, style.Colors["Text"])

        ; Draw label
        ctx.drawList.AddText(x2 + style.ItemInnerSpacing.x, y1 + (sliderHeight - labelSize.height) / 2,
            label, style.Colors["Text"])

        ; Update state
        this.lastItemRect := {x1: x1, y1: y1, x2: x2 + style.ItemInnerSpacing.x + labelSize.width, y2: y2}
        this.lastItemId := id
        this.lastItemHovered := hovered
        this.lastItemActive := (ctx.activeId == id)

        ; Advance cursor
        win.cursorPos.y += sliderHeight + style.ItemSpacing.y

        return valueChanged
    }

    ; ============== INPUT TEXT ==============

    static InputText(label, &value, flags := 0) {
        ctx := g_ImGuiCtx
        win := ctx.currentWindow
        if (!win)
            return false

        style := ctx.style
        io := ctx.io
        id := ctx.GetID(label)

        ; Layout
        labelSize := ctx.CalcTextSize(label)
        inputWidth := 150
        inputHeight := ctx.style.FontSize + style.FramePadding.y * 2

        x1 := win.cursorPos.x
        y1 := win.cursorPos.y
        x2 := x1 + inputWidth
        y2 := y1 + inputHeight

        ; Interaction
        hovered := io.IsMouseHoveringRect(x1, y1, x2, y2)

        if (hovered && io.MouseClicked[1]) {
            ctx.activeId := id
            ctx.io.WantCaptureKeyboard := true
        }

        isActive := (ctx.activeId == id)
        valueChanged := false

        ; Handle keyboard input when active
        if (isActive) {
            ctx.io.WantCaptureKeyboard := true

            ; This is simplified - full implementation would need hotkey hooks
            ; For now, we'll use InputBox or similar
        }

        ; Draw background
        bgColor := isActive ? style.Colors["FrameBgActive"]
                 : hovered ? style.Colors["FrameBgHovered"]
                 : style.Colors["FrameBg"]
        ctx.drawList.AddRectFilled(x1, y1, x2, y2, bgColor, style.FrameRounding)
        ctx.drawList.AddRect(x1, y1, x2, y2, style.Colors["Border"], 1, style.FrameRounding)

        ; Draw text
        displayText := value
        if (StrLen(displayText) > 20)
            displayText := SubStr(displayText, 1, 20) "..."
        ctx.drawList.AddText(x1 + style.FramePadding.x, y1 + style.FramePadding.y,
            displayText, style.Colors["Text"])

        ; Draw cursor if active
        if (isActive) {
            cursorX := x1 + style.FramePadding.x + ctx.CalcTextSize(value).width
            if (Mod(A_TickCount, 1000) < 500)
                ctx.drawList.AddLine(cursorX, y1 + 2, cursorX, y2 - 2, style.Colors["Text"], 1)
        }

        ; Draw label
        ctx.drawList.AddText(x2 + style.ItemInnerSpacing.x, y1 + (inputHeight - labelSize.height) / 2,
            label, style.Colors["Text"])

        ; Advance cursor
        win.cursorPos.y += inputHeight + style.ItemSpacing.y

        return valueChanged
    }

    ; ============== LAYOUT ==============

    static Separator() {
        ctx := g_ImGuiCtx
        win := ctx.currentWindow
        if (!win)
            return

        style := ctx.style

        y := win.cursorPos.y + style.ItemSpacing.y / 2
        ctx.drawList.AddLine(
            win.cursorStartPos.x, y,
            win.pos.x + win.size.x - style.WindowPadding.x, y,
            style.Colors["Separator"], 1)

        win.cursorPos.y += style.ItemSpacing.y
    }

    static SameLine(offsetX := 0, spacing := -1) {
        ctx := g_ImGuiCtx
        win := ctx.currentWindow
        if (!win)
            return

        style := ctx.style

        if (spacing < 0)
            spacing := style.ItemSpacing.x

        win.cursorPos.x := this.lastItemRect.x2 + spacing + offsetX
        win.cursorPos.y := this.lastItemRect.y1
    }

    static NewLine() {
        ctx := g_ImGuiCtx
        win := ctx.currentWindow
        if (!win)
            return

        win.cursorPos.x := win.cursorStartPos.x
        win.cursorPos.y := win.cursorMaxPos.y + ctx.style.ItemSpacing.y
    }

    static Spacing() {
        ctx := g_ImGuiCtx
        win := ctx.currentWindow
        if (!win)
            return

        win.cursorPos.y += ctx.style.ItemSpacing.y
    }

    static Indent(width := 0) {
        ctx := g_ImGuiCtx
        win := ctx.currentWindow
        if (!win)
            return

        if (width == 0)
            width := ctx.style.IndentSpacing

        win.cursorPos.x += width
        win.cursorStartPos.x += width
    }

    static Unindent(width := 0) {
        ctx := g_ImGuiCtx
        win := ctx.currentWindow
        if (!win)
            return

        if (width == 0)
            width := ctx.style.IndentSpacing

        win.cursorPos.x -= width
        win.cursorStartPos.x -= width
    }

    ; ============== TREE ==============

    static TreeNode(label) {
        ctx := g_ImGuiCtx
        win := ctx.currentWindow
        if (!win)
            return false

        style := ctx.style
        io := ctx.io
        id := ctx.GetID(label)

        ; Get open state
        isOpen := ctx.GetWidgetState(id, "open", false)

        ; Layout
        textSize := ctx.CalcTextSize(label)
        arrowSize := textSize.height
        totalHeight := textSize.height + style.FramePadding.y * 2

        x1 := win.cursorPos.x
        y1 := win.cursorPos.y
        x2 := win.pos.x + win.size.x - style.WindowPadding.x
        y2 := y1 + totalHeight

        ; Interaction
        hovered := io.IsMouseHoveringRect(x1, y1, x2, y2)
        clicked := hovered && io.MouseClicked[1]

        if (clicked) {
            isOpen := !isOpen
            ctx.SetWidgetState(id, "open", isOpen)
        }

        ; Draw background on hover
        if (hovered)
            ctx.drawList.AddRectFilled(x1, y1, x2, y2, style.Colors["HeaderHovered"], style.FrameRounding)

        ; Draw arrow
        arrowX := x1 + style.FramePadding.x
        arrowY := y1 + totalHeight / 2
        if (isOpen) {
            ; Down arrow
            ctx.drawList.AddTriangleFilled(
                arrowX, arrowY - 4,
                arrowX + 8, arrowY - 4,
                arrowX + 4, arrowY + 4,
                style.Colors["Text"])
        } else {
            ; Right arrow
            ctx.drawList.AddTriangleFilled(
                arrowX, arrowY - 4,
                arrowX + 8, arrowY,
                arrowX, arrowY + 4,
                style.Colors["Text"])
        }

        ; Draw label
        ctx.drawList.AddText(arrowX + arrowSize + style.ItemInnerSpacing.x, y1 + style.FramePadding.y,
            label, style.Colors["Text"])

        ; Advance cursor
        win.cursorPos.y += totalHeight + style.ItemSpacing.y

        ; If open, indent for children
        if (isOpen) {
            ctx.PushID(id)
            this.Indent()
        }

        return isOpen
    }

    static TreePop() {
        this.Unindent()
        g_ImGuiCtx.PopID()
    }

    static CollapsingHeader(label, flags := 0) {
        ; Similar to TreeNode but full-width and no indent
        ctx := g_ImGuiCtx
        win := ctx.currentWindow
        if (!win)
            return false

        style := ctx.style
        io := ctx.io
        id := ctx.GetID(label)

        isOpen := ctx.GetWidgetState(id, "open", true)

        textSize := ctx.CalcTextSize(label)
        totalHeight := textSize.height + style.FramePadding.y * 2

        x1 := win.cursorStartPos.x
        y1 := win.cursorPos.y
        x2 := win.pos.x + win.size.x - style.WindowPadding.x
        y2 := y1 + totalHeight

        hovered := io.IsMouseHoveringRect(x1, y1, x2, y2)
        clicked := hovered && io.MouseClicked[1]

        if (clicked) {
            isOpen := !isOpen
            ctx.SetWidgetState(id, "open", isOpen)
        }

        ; Draw background
        bgColor := hovered ? style.Colors["HeaderHovered"] : style.Colors["Header"]
        ctx.drawList.AddRectFilled(x1, y1, x2, y2, bgColor, style.FrameRounding)

        ; Draw arrow and label
        arrowX := x1 + style.FramePadding.x
        arrowY := y1 + totalHeight / 2
        if (isOpen) {
            ctx.drawList.AddTriangleFilled(arrowX, arrowY - 4, arrowX + 8, arrowY - 4, arrowX + 4, arrowY + 4, style.Colors["Text"])
        } else {
            ctx.drawList.AddTriangleFilled(arrowX, arrowY - 4, arrowX + 8, arrowY, arrowX, arrowY + 4, style.Colors["Text"])
        }

        ctx.drawList.AddText(arrowX + 16, y1 + style.FramePadding.y, label, style.Colors["Text"])

        win.cursorPos.y += totalHeight + style.ItemSpacing.y

        return isOpen
    }

    ; ============== COMBO ==============

    static BeginCombo(label, previewValue, flags := 0) {
        ctx := g_ImGuiCtx
        win := ctx.currentWindow
        if (!win)
            return false

        style := ctx.style
        io := ctx.io
        id := ctx.GetID(label)

        ; Layout
        labelSize := ctx.CalcTextSize(label)
        comboWidth := 150
        comboHeight := ctx.style.FontSize + style.FramePadding.y * 2

        x1 := win.cursorPos.x
        y1 := win.cursorPos.y
        x2 := x1 + comboWidth
        y2 := y1 + comboHeight

        ; Check if popup is open
        isOpen := (this.comboPopupId == id && this.comboPopupOpen)

        ; Interaction
        hovered := io.IsMouseHoveringRect(x1, y1, x2, y2)
        clicked := hovered && io.MouseClicked[1]

        if (clicked) {
            this.comboPopupOpen := !isOpen
            this.comboPopupId := id
            isOpen := this.comboPopupOpen
        }

        ; Draw combo box
        bgColor := hovered ? style.Colors["FrameBgHovered"] : style.Colors["FrameBg"]
        ctx.drawList.AddRectFilled(x1, y1, x2, y2, bgColor, style.FrameRounding)
        ctx.drawList.AddRect(x1, y1, x2, y2, style.Colors["Border"], 1, style.FrameRounding)

        ; Draw preview text
        ctx.drawList.AddText(x1 + style.FramePadding.x, y1 + style.FramePadding.y,
            previewValue, style.Colors["Text"])

        ; Draw dropdown arrow
        arrowX := x2 - comboHeight + 4
        arrowY := y1 + comboHeight / 2
        ctx.drawList.AddTriangleFilled(
            arrowX, arrowY - 3,
            arrowX + 8, arrowY - 3,
            arrowX + 4, arrowY + 3,
            style.Colors["Text"])

        ; Draw label
        ctx.drawList.AddText(x2 + style.ItemInnerSpacing.x, y1 + (comboHeight - labelSize.height) / 2,
            label, style.Colors["Text"])

        ; Store combo rect for popup positioning
        if (isOpen) {
            ctx.SetWidgetState(id, "comboRect", {x1: x1, y1: y2, x2: x2, width: comboWidth})
        }

        ; Advance cursor
        win.cursorPos.y += comboHeight + style.ItemSpacing.y

        return isOpen
    }

    static EndCombo() {
        ; Close popup area
        this.comboPopupOpen := false
    }

    static Selectable(label, selected := false, flags := 0) {
        ctx := g_ImGuiCtx
        win := ctx.currentWindow
        if (!win)
            return false

        style := ctx.style
        io := ctx.io

        textSize := ctx.CalcTextSize(label)
        itemHeight := textSize.height + style.FramePadding.y * 2

        x1 := win.cursorPos.x
        y1 := win.cursorPos.y
        x2 := win.pos.x + win.size.x - style.WindowPadding.x
        y2 := y1 + itemHeight

        hovered := io.IsMouseHoveringRect(x1, y1, x2, y2)
        clicked := hovered && io.MouseClicked[1]

        ; Draw background
        if (selected)
            ctx.drawList.AddRectFilled(x1, y1, x2, y2, style.Colors["HeaderActive"], style.FrameRounding)
        else if (hovered)
            ctx.drawList.AddRectFilled(x1, y1, x2, y2, style.Colors["HeaderHovered"], style.FrameRounding)

        ; Draw text
        ctx.drawList.AddText(x1 + style.FramePadding.x, y1 + style.FramePadding.y,
            label, style.Colors["Text"])

        win.cursorPos.y += itemHeight + style.ItemSpacing.y

        return clicked
    }

    ; ============== UTILITY ==============

    static IsItemHovered() {
        return this.lastItemHovered
    }

    static IsItemActive() {
        return this.lastItemActive
    }

    static IsItemClicked(button := 0) {
        return this.lastItemHovered && g_ImGuiCtx.io.MouseClicked[button + 1]
    }

    static SetNextWindowPos(x, y, cond := 0) {
        this.nextWindowPos := {x: x, y: y}
    }

    static SetNextWindowSize(w, h, cond := 0) {
        this.nextWindowSize := {x: w, y: h}
    }

    static PushStyleColor(idx, color) {
        ctx := g_ImGuiCtx
        oldColor := ctx.style.Colors.Has(idx) ? ctx.style.Colors[idx] : 0
        this.colorStack.Push({idx: idx, color: oldColor})
        ctx.style.Colors[idx] := color
    }

    static PopStyleColor(count := 1) {
        ctx := g_ImGuiCtx
        Loop count {
            if (this.colorStack.Length > 0) {
                item := this.colorStack.Pop()
                ctx.style.Colors[item.idx] := item.color
            }
        }
    }
}

; Helper functions
Clamp(v, min, max) {
    return v < min ? min : (v > max ? max : v)
}

Max(a, b) {
    return a > b ? a : b
}

Min(a, b) {
    return a < b ? a : b
}
```

---

## Complete Starter Code

Save this as `examples/01_basic_window.ahk`:

```autohotkey
#Requires AutoHotkey v2.0
#SingleInstance Force

; Add the library path
#Include ../lib/AhkImGui.ahk

; Create application
app := AhkImGui(600, 400, "AhkImGui Demo")

; State variables
global counter := 0
global sliderValue := 50
global checkboxValue := true
global radioValue := 1
global inputText := "Hello"

; Run the app with our UI function
app.Run(MainUI)

MainUI() {
    global

    ImGui.SetNextWindowPos(20, 20)
    ImGui.SetNextWindowSize(250, 350)

    if ImGui.Begin("Demo Window") {

        ImGui.Text("Welcome to AhkImGui!")
        ImGui.Separator()

        ; Button
        if ImGui.Button("Click Me!") {
            counter++
        }
        ImGui.SameLine()
        ImGui.Text("Count: " counter)

        ImGui.Spacing()

        ; Slider
        if ImGui.SliderInt("My Slider", &sliderValue, 0, 100)
            ToolTip("Slider: " sliderValue)

        ; Checkbox
        ImGui.Checkbox("Enable Feature", &checkboxValue)

        ; Radio buttons
        ImGui.RadioButton("Option 1", &radioValue, 1)
        ImGui.RadioButton("Option 2", &radioValue, 2)
        ImGui.RadioButton("Option 3", &radioValue, 3)

        ImGui.Separator()

        ; Collapsing section
        if ImGui.CollapsingHeader("More Options") {
            ImGui.Text("Hidden content here!")
            ImGui.Button("Nested Button")
        }

        ; Tree node
        if ImGui.TreeNode("Tree Section") {
            ImGui.Text("Tree content")
            if ImGui.TreeNode("Nested Node") {
                ImGui.Text("Deeply nested!")
                ImGui.TreePop()
            }
            ImGui.TreePop()
        }

        ImGui.End()
    }

    ; Second window
    ImGui.SetNextWindowPos(290, 20)
    ImGui.SetNextWindowSize(200, 150)

    if ImGui.Begin("Another Window") {
        ImGui.Text("This is another window.")

        if checkboxValue {
            ImGui.TextColored(0xFF00FF00, "Feature is ON")
        } else {
            ImGui.TextColored(0xFFFF0000, "Feature is OFF")
        }

        ImGui.Text("Radio selection: " radioValue)

        ImGui.End()
    }
}

; Exit on Escape
Escape::ExitApp()
```

---

## Testing Your Implementation

### Test Checklist

1. **Window System**
   - [ ] Windows display correctly
   - [ ] Windows can be dragged by title bar
   - [ ] Windows can be resized from corner
   - [ ] Multiple windows work independently

2. **Basic Widgets**
   - [ ] Text displays correctly
   - [ ] Buttons respond to clicks
   - [ ] Checkboxes toggle state
   - [ ] Radio buttons work in groups
   - [ ] Sliders are draggable

3. **Layout**
   - [ ] SameLine() places items horizontally
   - [ ] Separator() draws a line
   - [ ] Spacing() adds vertical space
   - [ ] Indent/Unindent work correctly

4. **Interaction**
   - [ ] Hover states change colors
   - [ ] Active states (while clicking) work
   - [ ] Mouse is captured correctly

### Debug Tips

```autohotkey
; Add debug output
ImGui.Text("Mouse: " g_ImGuiCtx.io.MousePos.x ", " g_ImGuiCtx.io.MousePos.y)
ImGui.Text("Active ID: " g_ImGuiCtx.activeId)
ImGui.Text("Hovered ID: " g_ImGuiCtx.hoveredId)
ImGui.Text("FPS: " Round(1 / g_ImGuiCtx.io.DeltaTime))
```

---

## Extending the Library

### Adding a New Widget

Follow this pattern:

```autohotkey
static MyNewWidget(label, &value, options*) {
    ctx := g_ImGuiCtx
    win := ctx.currentWindow
    if (!win)
        return false

    style := ctx.style
    io := ctx.io
    id := ctx.GetID(label)

    ; 1. Calculate size and position
    ; 2. Check interaction (hover, click, drag)
    ; 3. Draw background
    ; 4. Draw content
    ; 5. Draw label
    ; 6. Update lastItemRect, lastItemId, etc.
    ; 7. Advance cursor
    ; 8. Return whether value changed

    return valueChanged
}
```

### Useful Widgets to Add

1. **ProgressBar** - Simple fill rectangle
2. **ColorEdit** - RGB/HSV color picker
3. **Image** - Display bitmap/icon
4. **ListBox** - Scrollable list selection
5. **TabBar** - Tabbed interface
6. **Tooltip** - Hover information
7. **Modal/Popup** - Overlay dialogs
8. **Menu** - Dropdown menus
9. **Plot** - Simple line/bar graphs
10. **Table** - Data grid

---

## Troubleshooting

### Common Issues

**1. Window doesn't appear**
- Check if GDI+ initialized: `MsgBox(Gdip.pToken)`
- Verify window handle: `MsgBox(app.hwnd)`

**2. Drawing looks wrong**
- Colors are ARGB (Alpha-Red-Green-Blue), not RGB
- Check coordinate system (0,0 is top-left)

**3. Input not working**
- Verify mouse coordinates are window-relative
- Check `io.MouseDown` array is updating

**4. Performance issues**
- Reduce timer interval (32ms = 30fps)
- Cache brush objects (already done)
- Profile with `A_TickCount`

**5. Text not rendering**
- Verify font was created: `MsgBox(ctx.hFont)`
- Check text color alpha channel (should be 0xFF...)

### Performance Optimization

```autohotkey
; 1. Reduce redraw frequency for static UIs
SetTimer(MainLoop, 32)  ; 30 FPS instead of 60

; 2. Only redraw when needed
static needsRedraw := true
if (needsRedraw) {
    this.Render()
    needsRedraw := false
}

; 3. Use dirty rectangles (advanced)
; Only redraw changed regions
```

---

## Next Steps

1. **Copy the library files** to your project
2. **Run the basic example** to verify it works
3. **Experiment with widgets** - modify values, add new elements
4. **Read the imgui demo** (imgui_demo.cpp) for widget ideas
5. **Add features incrementally** - start simple, build up

Good luck building your AhkImGui library!
