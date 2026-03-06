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

### Export / Import (HTML, Markdown)

This library provides `block-editor.services.export-service`, `block-editor.services.import-service`, and `block-editor.services.html-parser` (migrated from `mgl-richtext-editor`):

```clojure
(ns your.export
  (:require [block-editor.services.export-service :as export]
            [block-editor.services.import-service :as import]))

;; Export
(def html (export/state->html @!state))
(def md  (export/state->md @!state))

;; Import: HTML -> blocks; or simple Markdown -> blocks
(def blocks (import/html->blocks "<p>Hello</p>"))
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
