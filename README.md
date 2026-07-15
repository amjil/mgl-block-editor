# mgl-block-editor

A **block editor** for Traditional Mongolian, built on top of [`mgl-richtext-editor`](https://github.com/amjil/mgl-richtext-editor).

It owns the outer layer: nested block tree, cross-block navigation, collapse, slash commands, toolbar, and block-level mutations. Rich-text editing and IME behavior inside each focused block (Enter to split, Backspace to merge, spans, selection, etc.) are reused from `mgl-richtext-editor`.

Layout scrolls **horizontally** with vertical Mongol text — suitable for notes and documents in Traditional Mongolian script.

## Features

- Nested blocks with indent / outdent (Tab / Shift+Tab)
- Collapse subtrees (block handle arrow, or shortcut)
- Active / inactive blocks: only the focused block mounts the editable rich-text widget; other blocks render as static Mongol text
- Slash menu (`/`) for block types and date/time inserts
- Bottom toolbar for converting / inserting common block types
- Multi-select (Shift+drag marquee) with bulk actions
- Built-in plugins: task, blockquote, image, divider
- Mobile virtual keyboard and desktop IME integration (via sibling packages)

## Dependencies

This library expects sibling local packages (see `deps.edn`), including:

| Package | Role |
|---------|------|
| `mgl-richtext-editor` | Per-block rich text, IME, transactions |
| `mgl-components` / `mongol` stack | Mongol UI / typography |
| `mongol-virtual-keyboard`, `mongol-desktop-ime`, `mgl-ime-core` | Input methods |

## Install (local)

Add to your CLJD `deps.edn`:

```clojure
amjil/mgl-block-editor {:local/root "../mgl-block-editor"}
```

## Quick start

Document state is a plain atom with `:root` (top-level block ids), `:blocks` (id → block map), and optional UI fields such as `:active-block-id`.

```clojure
(ns your.app
  (:require ["package:flutter/material.dart" :as m]
            [block-editor.widgets.editor :as editor]
            [block-editor.widgets.toolbar :as toolbar]))

(defonce !state
  (atom {:root ["b-1"]
         :active-block-id nil
         :blocks {"b-1" {:id "b-1"
                         :type :paragraph
                         :content ""
                         :content-spans []
                         :children []}}}))

(def editor-config
  {:text-block-types #{:paragraph :heading-1 :heading-2 :heading-3
                       :blockquote :list-item :task}
   :on-link-tap (fn [target] (println "link tap:" target))
   :block-style (fn [b-type]
                  (case b-type
                    :heading-1 (m/TextStyle .fontSize 32.0
                                            .fontWeight m.FontWeight/bold
                                            .fontFamily "OyunQaganTig")
                    (m/TextStyle .fontSize 18.0
                                 .fontFamily "OyunQaganTig")))})

(defn screen []
  (m/Scaffold
   .body
   (editor/mongol-block-editor
    !state
    :editor-config editor-config
    :toolbar (toolbar/editor-toolbar !state editor-config)
    :show-block-handles? true
    :padding (m.EdgeInsets/all 24.0))))
```

See `example/src/block_editor_example/main.cljd` for a fuller demo (nested blocks, tasks, images, IME setup).

## `mongol-block-editor` options

| Option | Default | Description |
|--------|---------|-------------|
| `:editor-config` | `nil` | Passed through `block-editor.config/build-config` |
| `:padding` | `EdgeInsets.all 24` | Scroll view padding |
| `:background-color` | `transparent` | Editor background |
| `:show-block-handles?` | `false` | Drag / collapse handles |
| `:toolbar` | none | Widget under the editor (e.g. `editor-toolbar`) |
| `:leading-widget` | none | Reserved for leading chrome |

## Editor config

`block-editor.config/build-config` merges user config with rich-editor defaults and expands `:block-plugins`.

Useful keys:

| Key | Description |
|-----|-------------|
| `:text-block-types` | Set of types treated as editable text |
| `:block-style` | `(fn [block-type] -> TextStyle)` |
| `:on-link-tap` | Called for link/tag taps in read mode |
| `:block-plugins` | Vector of plugin maps (merged over built-ins by `:type`) |

Built-in plugins are always registered unless you override the same `:type`:  
`task`, `blockquote`, `image`, `divider` — under `block-editor.plugins.*`, re-exported from `block-editor.plugins.core`.

## Inactive read mode

Only the focused block (`:active-block-id`) mounts the editable rich-text view. All other blocks use `block-editor.widgets.reader/block-reader-view` (static Mongol text, or a plugin’s `:render-read`). Tap an inactive block to focus it.

## Block plugins

Extend the editor with `:block-plugins` in `editor-config`. Each plugin is a plain map:

| Key | Required | Description |
|-----|----------|-------------|
| `:type` | yes | Block type keyword (e.g. `:my-widget`) |
| `:text-block-type?` | no | If true, treat as editable text |
| `:render` / `:render-edit` | for custom UI | `(fn [block-id !state index cfg] -> Widget)` — active mode |
| `:render-read` | no | `(fn [block base-style on-link-tap !state] -> Widget)` — inactive |
| `:text-block-wrapper` | no | `(fn [text-widget id !state idx block style cfg] -> Widget)` |

Example:

```clojure
(ns your.app
  (:require ["package:flutter/material.dart" :as m]
            [block-editor.widgets.editor :as editor]))

(def my-block-plugin
  {:type :my-widget
   :text-block-type? false
   :render (fn [id !state _index _cfg]
             (m/Container .child (m/Text (str "Custom: " id))))
   :render-read (fn [block _style _on-link-tap _!state]
                  (m/Text (str "Read: " (get block :content ""))))})

(editor/mongol-block-editor !state
  :editor-config {:block-plugins [my-block-plugin]
                  :on-link-tap (fn [t] (println t))})
```

## Document operations

Structural mutations live in `block-editor.commands.ops` (also wired as `:block-ops` on the built config):

| Function | Purpose |
|----------|---------|
| `split-block-at-cursor!` | Enter — split at caret |
| `merge-with-previous!` | Backspace at start — merge with previous sibling |
| `indent-block!` / `outdent-block!` | Tab / Shift+Tab nesting |
| `reorder-block!` | Reorder among root blocks |
| `delete-block!` / `duplicate-block!` | Single-block delete / duplicate |
| `convert-block!` | Change block type |
| `toggle-block-selection!` / `set-block-selection!` | Multi-select |
| `delete-blocks!` / `duplicate-blocks!` / `convert-blocks!` | Bulk ops |
| `execute-slash-command!` | Apply slash menu command |

## Slash menu & toolbar

- **Slash menu** (`block-editor.widgets.slash-menu`): type `/` at the end of a block to change type or insert today’s / tomorrow’s date and current time.
- **Toolbar** (`block-editor.widgets.toolbar/editor-toolbar`): horizontal strip for paragraph, headings, list, task, quote, image, divider, and table.

## Example app

Run the demo from `example/`:

```bash
cd example
flutter pub get
clj -M:cljd compile
# Hot reload (macOS desktop):
clj -M:cljd flutter -d macos
```

The example initializes Mongolian IME assets and shows nested headings, tasks, and an image block.

## Layout of source

```
src/block_editor/
  widgets/     editor, block_tree, reader, toolbar, slash_menu, selection_bar
  commands/    ops (tree-aware structural mutations)
  handlers/    keyboard
  plugins/     task, blockquote, image, divider, core
  config.cljd  build-config / plugin expansion
  utils/       collapse, text, date
```
