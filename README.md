# overleaf-review-comments-exporter

A small browser-console script for exporting Overleaf review comments.

Select text in the Overleaf Code Editor, run the script in DevTools Console, and it downloads a Markdown file.

The exported Markdown keeps the selected source text, each highlighted fragment, its nearby context, and the matching reviewer comment.

## Usage

1. Open Overleaf Code Editor and the Review panel.
2. Select the source text you want to export.
3. Paste `overleaf-review-comments-exporter.js` into DevTools Console.
4. Run it. A `.md` file downloads automatically.

If Chrome blocks pasting into Console, type `allow pasting` once and paste again.
