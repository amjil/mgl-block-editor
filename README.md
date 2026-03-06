## mgl-block-editor (Mongolian Block Editor Library)

A **block editor** wrapper built on top of `../mgl-richtext-editor`: it provides block list, cross-block navigation, and block-level commands at the outer layer; rich-text editing and IME behavior inside each block (Enter to split blocks, Backspace to merge, cross-block selection, etc.) are reused from `mgl-richtext-editor`.

### Using as a library

Add the dependency to your CLJD project’s `deps.edn` (for local development):

```clojure
amjil/mgl-block-editor {:local/root "../mgl-block-editor"}
```

Minimal usage example:

```clojure
(ns your.app
  (:require [block-editor.state :as be-state]
            [block-editor.widgets.editor-container :as be-widget]))

(defonce !state (atom (be-state/initial-state)))

;; Render in your widget tree:
;; (be-widget/editor-container !state nil)
```

### Block plugins (custom block types)

You can extend the editor with custom block types by passing a `:block-plugins` vector in `editor-config`. Each plugin is a map with:

| Key | Required | Description |
|-----|----------|-------------|
| `:type` | yes | Block type keyword (e.g. `:my-widget`) |
| `:render` | yes | `(fn [id !state root-index block style cfg] -> Widget)` — same as rich-editor custom-block-renderers |
| `:block-style` | no | `TextStyle` or `(fn [block-type] -> TextStyle)` for line height / layout |
| `:text-block-type?` | no | If true, this type is treated as editable text (Enter splits, etc.) |
| `:html-tags` | no | Vector of HTML tag names that map to this type on import (e.g. `["my-widget"]`) |
| `:block->html` | no | `(fn [block] -> string)` for HTML export |
| `:block->md` | no | `(fn [block] -> string)` for Markdown export |
| `:make-node` | no | `(fn [id] -> node)` to create a new block; default is `(node/make-node id type "")` |

Example:

```clojure
(ns your.app
  (:require ["package:flutter/material.dart" :as m]
            [block-editor.widgets.editor-container :as be-widget]
            [block-editor.services.export-service :as export]
            [block-editor.services.import-service :as import]
            [block-editor.commands.block-commands :as block-cmd]))

(def my-block-plugin
  {:type :my-widget
   :render (fn [id !state _root-index block _style _cfg]
             (m/Container
              .child (m/Text (str "Custom: " (get block :content "")))))
   :block-style (m/TextStyle. .fontSize 18.0)
   :html-tags ["my-widget"]
   :block->html (fn [block] (str "<my-widget>" (get block :content "") "</my-widget>\n"))
   :block->md  (fn [block] (str "[" (get block :content "") "]\n\n"))})

;; Editor with plugin
(be-widget/editor-container !state {:block-plugins [my-block-plugin]})

;; Export with same plugins so custom types are serialized
(export/state->html @!state {:block-plugins [my-block-plugin]})
(export/state->md  @!state {:block-plugins [my-block-plugin]})

;; Import HTML that contains <my-widget> (use same plugin for tag mapping)
(import/html->blocks html-str {:block-plugins [my-block-plugin]})

;; Insert a block of custom type (optionally with :make-node and data)
(block-cmd/insert-block-after-with-type! !state active-id :my-widget {:custom-key "value"})
```

### Export / Import (HTML, Markdown)

This library provides `block-editor.services.export-service`, `block-editor.services.import-service`, and `block-editor.services.html-parser` (migrated from `mgl-richtext-editor`):

```clojure
(ns your.export
  (:require [block-editor.services.export-service :as export]
            [block-editor.services.import-service :as import]))

;; Export (optionally with :block-plugins for custom block types)
(def html (export/state->html @!state))
(def html (export/state->html @!state {:block-plugins [my-block-plugin]}))
(def md  (export/state->md @!state))
(def md  (export/state->md @!state {:block-plugins [my-block-plugin]}))

;; Import: HTML -> blocks; or simple Markdown -> blocks
(def blocks (import/html->blocks "<p>Hello</p>"))
(def blocks (import/html->blocks html-str {:block-plugins [my-block-plugin]}))
(def blocks (import/md->blocks "# Title\n\nParagraph"))
```

### Save / Load (reusing rich-editor state shape)

The library reuses the `rich-editor.services.serialization-service` shape `{:blocks :root :selection :active-block-id}`:

```clojure
(ns your.save
  (:require [block-editor.services.serialization-service :as ser]))

(def snapshot (ser/export-state @!state))
(ser/load-state! !state snapshot)
```

### Example app (must be run from `example/`)

Run the demo (e.g. on macOS desktop):

```bash
cd example
flutter pub get
clj -M:cljd compile
# Or run with hot reload:
clj -M:cljd flutter -d macos
```
