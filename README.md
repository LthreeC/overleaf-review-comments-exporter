# Overleaf Review Comments Exporter

Export selected Overleaf review comments to Markdown.

This tiny browser-console script helps you copy comments from an Overleaf LaTeX project.  
Select a block of text in Overleaf, run the script in DevTools Console, and it downloads a Markdown file.

The exported Markdown focuses on the important review-revision mapping:

```text
Fragment → Fragment context → Comment
```



## Usage

1. Open Overleaf Code Editor and the Review panel.
2. Select the source text you want to export.
3. Paste `overleaf-review-comments-exporter.js` into DevTools Console.
4. Run it. A `.md` file downloads automatically.

If Chrome blocks pasting into Console, type `allow pasting` once and paste again.





## Script



```js
(() => {
  "use strict";
  const CFG = { prefix: "overleaf_review_comments_export", maxJoinGap: 12, highlightSelector: ".cm-content .ol-cm-change.ol-cm-change-c" };
  const $ = (s, r = document) => r.querySelector(s);
  const $$ = (s, r = document) => Array.from(r.querySelectorAll(s));
  const clean = s => String(s ?? "").replace(/\u00a0/g, " ").replace(/\r\n?/g, "\n").trim();
  const oneLine = s => clean(s).replace(/\s+/g, " ");
  const safeName = s => (s || "overleaf").replace(/[^\w\u4e00-\u9fa5.-]+/g, "_").slice(0, 90);
  function fence(s, lang = "tex") { s = String(s ?? ""); let f = "```"; while (s.includes(f)) f += "`"; return `${f}${lang}\n${s}\n${f}`; }
  function dedupe(a, key) { const seen = new Set(); return a.filter(x => { const k = key(x); if (seen.has(k)) return false; seen.add(k); return true; }); }
  function getView() {
    for (const el of $$(".cm-editor, .cm-scroller, .cm-content, .cm-line")) {
      let cur = el;
      while (cur) {
        const v = cur.cmView;
        if (v?.view?.state?.doc) return v.view;
        if (v?.rootView?.view?.state?.doc) return v.rootView.view;
        if (v?.editorView?.state?.doc) return v.editorView;
        cur = cur.parentElement;
      }
    }
    return null;
  }
  const view = getView();
  if (!view?.state?.doc) { alert("找不到 CodeMirror 源文档。请切换到 Overleaf 的 Code 编辑器后再运行。"); return; }
  const doc = view.state.doc, source = doc.toString();
  const content = $(".cm-content[contenteditable='true']") || $(".cm-content");
  if (!content) { alert("找不到 CodeMirror 编辑区。"); return; }
  // Export comments whose source anchors lie within the editor selection.
  let selected = view.state.selection.ranges.filter(r => !r.empty).map(r => ({ from: Math.min(r.from, r.to), to: Math.max(r.from, r.to) }));
  if (!selected.length && window.getSelection()?.rangeCount) {
    const sel = window.getSelection();
    for (let i = 0; i < sel.rangeCount; i++) {
      const r = sel.getRangeAt(i);
      if (r.collapsed || !content.contains(r.startContainer) || !content.contains(r.endContainer)) continue;
      try { const a = view.posAtDOM(r.startContainer, r.startOffset), b = view.posAtDOM(r.endContainer, r.endOffset); if (a !== b) selected.push({ from: Math.min(a,b), to: Math.max(a,b) }); } catch (_) {}
    }
  }
  selected = dedupe(selected.sort((a,b) => a.from-b.from || a.to-b.to), r => `${r.from}:${r.to}`);
  if (!selected.length) { alert("没有取到编辑器选区。请先在源代码中选中包含评论锚点的段落，再运行脚本。"); return; }
  const isSelected = p => selected.some(r => p >= r.from && p <= r.to);
  // Keep the selected source in the export; only the redundant per-comment paragraph context is omitted.
  const selectedSource = selected.map(r => doc.sliceString(r.from, r.to)).join("\n\n--- selection gap ---\n\n");
  // Overleaf's actual review highlight is rendered with .ol-cm-change.ol-cm-change-c.
  // Do not infer highlight from color alone: the active line/selection can also be colored.
  // Map each highlighted text node back to source offsets using CodeMirror's posAtDOM.
  function collectYellowRanges() {
    const raw = [];
    const walker = document.createTreeWalker(content, NodeFilter.SHOW_TEXT);
    let node;
    while ((node = walker.nextNode())) {
      if (!node.nodeValue || !node.nodeValue.length || !node.parentElement) continue;
      const mark = node.parentElement.closest('.ol-cm-change.ol-cm-change-c');
      if (!mark || !content.contains(mark)) continue;
      // Ignore synthetic/empty decoration text where CodeMirror cannot map a real source span.
      try {
        const from = view.posAtDOM(node, 0);
        const to = view.posAtDOM(node, node.nodeValue.length);
        if (Number.isFinite(from) && Number.isFinite(to) && to > from) raw.push({ from, to });
      } catch (_) {}
    }
    raw.sort((a, b) => a.from - b.from || a.to - b.to);
    const merged = [];
    for (const item of raw) {
      const last = merged[merged.length - 1];
      if (!last) {
        merged.push({ from: item.from, to: item.to });
        continue;
      }
      if (item.from <= last.to) {
        last.to = Math.max(last.to, item.to);
        continue;
      }
      const gap = doc.sliceString(last.to, item.from);
      // Syntax/highlight decorations split a visually continuous fragment into multiple spans.
      // Join only short gaps consisting exclusively of whitespace or punctuation/symbols;
      // never bridge unhighlighted words, so separate comments are not casually conflated.
      const onlySeparators = gap.length <= CFG.maxJoinGap && /^[\s\p{P}\p{S}]*$/u.test(gap);
      if (onlySeparators) last.to = Math.max(last.to, item.to);
      else merged.push({ from: item.from, to: item.to });
    }
    return merged
      .map(range => ({ ...range, text: doc.sliceString(range.from, range.to) }))
      .filter(range => clean(range.text));
  }
  const yellowRanges = collectYellowRanges();
  function lineAt(pos) { const l = doc.lineAt(Math.max(0,Math.min(doc.length,pos))); return { number:l.number, from:l.from, to:l.to, text:l.text }; }
  function parseComment(entry, index) {
    const notes = $$(".review-panel-comment", entry).map(box => {
      const author = clean($(".review-panel-entry-user",box)?.textContent);
      const time = clean($(".review-panel-entry-time",box)?.textContent);
      const body = clean($(".review-panel-comment-body",box)?.textContent || $(".review-panel-expandable-content",box)?.textContent);
      return body ? { author,time,body } : null;
    }).filter(Boolean);
    return { index, pos:Number(entry.dataset.pos), notes:dedupe(notes,n => `${n.author}|${n.time}|${oneLine(n.body)}`) };
  }
  const reviewRoot = $("#review-panel-current-file") || $(".review-panel-container") || document;
  const comments = dedupe(
    $$(".review-panel-entry-comment", reviewRoot)
      .map(parseComment)
      .filter(c => c.notes.length && Number.isFinite(c.pos) && isSelected(c.pos)),
    c => `${c.pos}|${c.notes.map(n => `${n.author}|${n.time}|${oneLine(n.body)}`).join("||")}`
  ).sort((a, b) => a.pos - b.pos);
  if (!comments.length) { alert("选区内没有找到评论锚点。请确认 Review 面板已打开，并且选区覆盖了评论所在位置。"); return; }
  // Match comments to real highlight intervals by source positions only. Do not consume a
  // range after one match: several comment cards/replies may refer to the same highlighted span.
  function matchHighlight(comment) {
    const containing = yellowRanges
      .filter(range => comment.pos >= range.from && comment.pos <= range.to)
      .sort((a, b) => (a.to - a.from) - (b.to - b.from));
    if (containing.length) return { ...containing[0], match: "anchor_inside_highlight" };
    const near = yellowRanges.map(range => ({
      ...range,
      distance: comment.pos < range.from ? range.from - comment.pos : comment.pos - range.to
    })).filter(range => range.distance <= 3)
      .sort((a, b) => a.distance - b.distance || (a.to - a.from) - (b.to - b.from));
    if (near.length) return { ...near[0], match: "within_3_source_chars" };
    return null;
  }
  const rows = comments.map(comment => {
    const highlight = matchHighlight(comment);
    const anchorLine = lineAt(comment.pos);
    const rangeStartLine = highlight ? lineAt(highlight.from) : anchorLine;
    const rangeEndLine = highlight ? lineAt(highlight.to) : anchorLine;
    const fullLines = {
      first: rangeStartLine.number,
      last: rangeEndLine.number,
      text: source.slice(rangeStartLine.from, rangeEndLine.to)
    };
    return { comment, highlight, line: anchorLine, fullLines };
  });
  const project = clean($("meta[name='ol-projectName']")?.content || document.title || "Overleaf project");
  const file = clean($$(".ol-cm-breadcrumbs div").map(x=>x.textContent).filter(Boolean).join("/")) || "current_file";
  const md=[]; const add=s=>md.push(s);
  add("# Overleaf review comments export"); add(""); add(`Project: ${project}`); add(`File: ${file}`); add(`Generated: ${new Date().toLocaleString()}`); add(`Yellow ranges detected in selected source: ${yellowRanges.filter(h=>selected.some(r=>h.to>=r.from&&h.from<=r.to)).length}`); add(`Comments matched: ${rows.length}`); add(""); add("## Selected source"); add(""); add(fence(selectedSource)); add(""); add("## Fragment → complete line → comment");
  rows.forEach((row,i) => {
    const h=row.highlight,c=row.comment;
    add(""); add(`### ${i+1}. Source position ${c.pos}; line ${row.line.number}`); add("");
    add("**Actual yellow fragment**"); add(""); add(fence(h ? h.text : "(没有从网页实际黄色背景中识别到与此评论锚点对应的范围；未猜测 fragment。)")); add("");
    if (h) { add(`Highlight source range: ${h.from}-${h.to} (${h.match})`); add(""); }
    add(`**Complete source line(s) covering fragment (lines ${row.fullLines.first}-${row.fullLines.last})**`); add(""); add(fence(row.fullLines.text)); add("");
    add("**Comment**");
    c.notes.forEach((n,j) => { const label=[n.author,n.time].filter(Boolean).join(" · ") || `Reply ${j+1}`; add(`- **${label}:** ${oneLine(n.body)}`); });
  });
  const blob = new Blob([md.join("\n")],{type:"text/markdown;charset=utf-8"});
  const url = URL.createObjectURL(blob), a=document.createElement("a");
  a.href=url; a.download=`${CFG.prefix}_${safeName(file)}_${new Date().toISOString().replace(/[:.]/g,"-")}.md`;
  a.style.display="none"; document.body.appendChild(a); a.click(); a.remove();
  setTimeout(()=>URL.revokeObjectURL(url),10000);
  console.log("[Overleaf review export]", {documentLength:source.length, highlightElements: $$(CFG.highlightSelector).length, yellowRanges:yellowRanges.length, comments:comments.length, matched:rows.filter(r=>r.highlight).length, file});
  console.table(rows.map((r,i)=>({index:i+1,pos:r.comment.pos,line:r.line.number,fragment:(r.highlight?.text||"(not detected)").slice(0,100),comments:r.comment.notes.map(n=>n.body).join(" | ").slice(0,140)})));
})();
```