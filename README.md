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
  const CFG = {
    topTolerance: 60,
    contextChars: 180,
    downloadPrefix: "overleaf_review_comments_export"
  };

  const $ = (s, r = document) => r.querySelector(s);
  const $$ = (s, r = document) => Array.from(r.querySelectorAll(s));

  const clean = (s) =>
    (s ?? "")
      .replace(/\u00a0/g, " ")
      .replace(/[ \t]+\n/g, "\n")
      .replace(/\n{3,}/g, "\n\n")
      .trim();

  const oneLine = (s) => clean(s).replace(/\s+/g, " ");
  const norm = (s) => oneLine(s);

  const safeName = (s) =>
    (s || "overleaf")
      .replace(/[^\w\u4e00-\u9fa5.-]+/g, "_")
      .slice(0, 90);

  function fence(text, lang = "") {
    text = text ?? "";
    let f = "```";
    while (text.includes(f)) f += "`";
    return `${f}${lang}\n${text}\n${f}`;
  }

  function dedupeBy(arr, keyFn) {
    const seen = new Set();
    return arr.filter((x) => {
      const key = keyFn(x);
      if (seen.has(key)) return false;
      seen.add(key);
      return true;
    });
  }

  function getCMView() {
    const candidates = $$(".cm-editor, .cm-scroller, .cm-content, .cm-line");

    for (const el of candidates) {
      let cur = el;
      while (cur) {
        const cmv = cur.cmView;
        if (cmv?.view?.state?.doc) return cmv.view;
        if (cmv?.rootView?.view?.state?.doc) return cmv.rootView.view;
        if (cmv?.editorView?.state?.doc) return cmv.editorView;
        cur = cur.parentElement;
      }
    }

    return null;
  }

  const view = getCMView();
  const doc = view?.state?.doc || null;
  const cmContent = $(".cm-content");

  const cmRanges = view
    ? view.state.selection.ranges
        .filter((r) => !r.empty)
        .map((r) => ({
          from: Math.min(r.from, r.to),
          to: Math.max(r.from, r.to)
        }))
    : [];

  const browserSel = window.getSelection();
  const browserRanges = [];

  if (browserSel?.rangeCount) {
    for (let i = 0; i < browserSel.rangeCount; i++) {
      const r = browserSel.getRangeAt(i);
      if (r.collapsed) continue;

      const inEditor =
        !cmContent ||
        cmContent.contains(r.startContainer) ||
        cmContent.contains(r.endContainer) ||
        cmContent.contains(r.commonAncestorContainer);

      if (inEditor) browserRanges.push(r);
    }
  }

  let selectedText = "";

  if (doc && cmRanges.length) {
    selectedText = cmRanges
      .map((r) => doc.sliceString(r.from, r.to))
      .join("\n\n--- selection gap ---\n\n");
  } else if (browserSel) {
    selectedText = browserSel.toString();
  }

  if (!clean(selectedText)) {
    alert("没有取到编辑器选区。请先在 Overleaf 代码编辑器里滑选文本，再运行脚本。");
    return;
  }

  function posInSelectedRanges(pos) {
    return cmRanges.some((r) => pos >= r.from && pos <= r.to);
  }

  function contextAroundPos(pos) {
    if (!doc || !Number.isFinite(pos)) return "";

    const p = Math.max(0, Math.min(doc.length, pos));
    const line = doc.lineAt(p);
    const rel = p - line.from;
    const start = Math.max(0, rel - CFG.contextChars);
    const end = Math.min(line.text.length, rel + CFG.contextChars);

    return `${start > 0 ? "…" : ""}${line.text.slice(start, end)}${
      end < line.text.length ? "…" : ""
    }`;
  }

  const contentRect = cmContent?.getBoundingClientRect();

  function rectToContentTop(rect) {
    if (!contentRect) return rect.top;
    return rect.top - contentRect.top;
  }

  function getRangeYRanges() {
    if (!contentRect) return [];

    const ranges = [];

    for (const r of browserRanges) {
      const rects = Array.from(r.getClientRects()).filter(
        (x) => x.width > 0 && x.height > 0
      );

      if (!rects.length) continue;

      const tops = rects.map((x) => rectToContentTop(x));
      const bottoms = rects.map((x) => x.bottom - contentRect.top);

      ranges.push({
        min: Math.min(...tops) - CFG.topTolerance,
        max: Math.max(...bottoms) + CFG.topTolerance
      });
    }

    return ranges;
  }

  const yRanges = getRangeYRanges();

  function topInSelectedYRanges(top) {
    return yRanges.some((r) => top >= r.min && top <= r.max);
  }

  function parseCommentEntry(entry, idx) {
    const notes = $$(".review-panel-comment", entry)
      .map((box) => {
        const author = clean($(".review-panel-entry-user", box)?.textContent);
        const time = clean($(".review-panel-entry-time", box)?.textContent);

        const body = clean(
          $(".review-panel-comment-body", box)?.textContent ||
            $(".review-panel-expandable-content", box)?.textContent
        );

        return body ? { author, time, body } : null;
      })
      .filter(Boolean);

    const uniqueNotes = dedupeBy(
      notes,
      (n) => `${n.author}|${n.time}|${norm(n.body)}`
    );

    return {
      idx,
      el: entry,
      pos: Number(entry.dataset.pos),
      top: Number(entry.dataset.top),
      notes: uniqueNotes
    };
  }

  const reviewRoot =
    $("#review-panel-current-file") || $(".review-panel-container") || document;

  const allComments = dedupeBy(
    $$(".review-panel-entry-comment", reviewRoot)
      .map(parseCommentEntry)
      .filter((x) => x.notes.length),
    (c) =>
      `${Number.isFinite(c.pos) ? c.pos : ""}|${
        Number.isFinite(c.top) ? c.top : ""
      }|${c.notes.map((n) => `${n.author}|${n.time}|${norm(n.body)}`).join("||")}`
  );

  let selectedComments = allComments.filter((c) => {
    if (cmRanges.length && Number.isFinite(c.pos)) {
      return posInSelectedRanges(c.pos);
    }

    if (yRanges.length && Number.isFinite(c.top)) {
      return topInSelectedYRanges(c.top);
    }

    return false;
  });

  function safeIntersects(range, node) {
    try {
      return range.intersectsNode(node);
    } catch {
      return false;
    }
  }

  function textFromIntersection(range, node) {
    if (!safeIntersects(range, node)) return "";

    const nodeRange = document.createRange();
    nodeRange.selectNodeContents(node);

    const r = range.cloneRange();

    if (range.compareBoundaryPoints(Range.START_TO_START, nodeRange) < 0) {
      r.setStart(nodeRange.startContainer, nodeRange.startOffset);
    }

    if (range.compareBoundaryPoints(Range.END_TO_END, nodeRange) > 0) {
      r.setEnd(nodeRange.endContainer, nodeRange.endOffset);
    }

    return r.toString();
  }

  function firstUsefulRect(el) {
    const rects = Array.from(el.getClientRects()).filter(
      (r) => r.width > 0 && r.height > 0
    );

    return rects[0] || el.getBoundingClientRect();
  }

  let highlights = $$(
    ".cm-content .ol-cm-change-c, .cm-content .ol-cm-change-highlight-c, .cm-content .ol-cm-change-focus-c"
  )
    .map((el, idx) => {
      let text = "";
      let selectedByDom = false;

      if (browserRanges.length) {
        text = browserRanges.map((r) => textFromIntersection(r, el)).join("");
        selectedByDom = !!clean(text);
      } else {
        const full = clean(el.textContent);
        if (full && norm(selectedText).includes(norm(full))) {
          text = full;
          selectedByDom = true;
        }
      }

      if (!clean(text)) return null;

      const rect = firstUsefulRect(el);
      const line = el.closest(".cm-line");

      return {
        idx,
        text: clean(text),
        context: clean(line?.textContent || ""),
        top: rectToContentTop(rect),
        left: contentRect ? rect.left - contentRect.left : rect.left,
        selectedByDom
      };
    })
    .filter(Boolean);

  highlights = dedupeBy(
    highlights,
    (h) =>
      `${Math.round(h.top)}|${Math.round(h.left)}|${norm(h.text)}|${norm(
        h.context
      )}`
  );

  if (selectedComments.length) {
    const commentTops = selectedComments
      .map((c) => c.top)
      .filter(Number.isFinite);

    if (commentTops.length) {
      highlights = highlights.filter(
        (h) =>
          h.selectedByDom ||
          commentTops.some((t) => Math.abs(t - h.top) <= CFG.topTolerance)
      );
    }
  }

  if (!selectedComments.length && highlights.length) {
    selectedComments = allComments.filter((c) =>
      highlights.some(
        (h) =>
          Number.isFinite(c.top) &&
          Math.abs(c.top - h.top) <= CFG.topTolerance
      )
    );
  }

  selectedComments.sort(
    (a, b) =>
      (Number.isFinite(a.pos) ? a.pos : 1e15) -
        (Number.isFinite(b.pos) ? b.pos : 1e15) ||
      (Number.isFinite(a.top) ? a.top : 1e15) -
        (Number.isFinite(b.top) ? b.top : 1e15) ||
      a.idx - b.idx
  );

  highlights.sort((a, b) => a.top - b.top || a.left - b.left || a.idx - b.idx);

  const usedHighlight = new Set();

  function pickHighlight(comment, fallbackIndex) {
    let best = null;
    let bestRealIndex = -1;

    if (Number.isFinite(comment.top)) {
      for (let i = 0; i < highlights.length; i++) {
        if (usedHighlight.has(i)) continue;

        const h = highlights[i];
        const dist = Math.abs(h.top - comment.top);

        if (dist > CFG.topTolerance) continue;

        if (
          !best ||
          dist < Math.abs(best.top - comment.top) ||
          (dist === Math.abs(best.top - comment.top) && h.left < best.left)
        ) {
          best = h;
          bestRealIndex = i;
        }
      }
    }

    if (!best && highlights[fallbackIndex] && !usedHighlight.has(fallbackIndex)) {
      best = highlights[fallbackIndex];
      bestRealIndex = fallbackIndex;
    }

    if (bestRealIndex >= 0) usedHighlight.add(bestRealIndex);

    return best;
  }

  const rows = selectedComments.map((comment, i) => {
    const h = pickHighlight(comment, i);

    return {
      comment,
      highlight: h,
      fallbackContext: h ? "" : contextAroundPos(comment.pos)
    };
  });

  const projectName = clean(
    $('meta[name="ol-projectName"]')?.content ||
      document.title ||
      "Overleaf project"
  );

  const fileName = clean(
    $$(".ol-cm-breadcrumbs div")
      .map((x) => x.textContent)
      .filter(Boolean)
      .join("/")
  );

  const md = [];
  const add = (s = "") => md.push(s);

  add("# Overleaf review comments export");
  add("");
  add("> Focus: read each item as `Fragment` → `Fragment context` → `Comment`.");
  add("> These pairs are the important content. Metadata only helps locate the source.");
  add("");
  add(`Project: ${projectName}`);
  add(`File: ${fileName || "(unknown current file)"}`);
  add(`Generated: ${new Date().toLocaleString()}`);
  add(`Matched comments: ${rows.length}`);
  add("");
  add("## Selected content");
  add("");
  add(fence(selectedText, "tex"));
  add("");
  add("## Fragment-comment pairs");

  if (!rows.length) {
    add("");
    add("_No matched review comments found in the current selection._");
  }

  rows.forEach((row, i) => {
    const c = row.comment;

    const fragment =
      row.highlight?.text ||
      "(yellow fragment not detected; use the context below to locate this comment)";

    const context =
      row.highlight?.context ||
      row.fallbackContext ||
      "(fragment context not detected)";

    add("");
    add(`### ${i + 1}`);
    add("");
    add("**Fragment**");
    add("");
    add(fence(fragment, "tex"));
    add("");
    add("**Fragment context**");
    add("");
    add(fence(context, "tex"));
    add("");
    add("**Comment**");

    c.notes.forEach((n, j) => {
      const head =
        [n.author, n.time].filter(Boolean).join(" · ") || `Comment ${j + 1}`;
      add(`- **${head}:** ${oneLine(n.body)}`);
    });
  });

  const output = md.join("\n");

  const blob = new Blob([output], { type: "text/markdown;charset=utf-8" });
  const a = document.createElement("a");
  const ts = new Date().toISOString().replace(/[:.]/g, "-");

  a.download = `${CFG.downloadPrefix}_${safeName(fileName || projectName)}_${ts}.md`;
  a.href = URL.createObjectURL(blob);

  document.body.appendChild(a);
  a.click();

  setTimeout(() => {
    URL.revokeObjectURL(a.href);
    a.remove();
  }, 1000);

  console.log("[Overleaf review comments export] done", {
    hasCodeMirrorView: !!view,
    selectedTextLength: selectedText.length,
    allCommentCards: allComments.length,
    matchedComments: rows.length,
    matchedFragments: rows.filter((r) => r.highlight).length,
    fileName,
    projectName
  });

  console.table(
    rows.map((r, i) => ({
      "#": i + 1,
      pos: r.comment.pos,
      fragment: (
        r.highlight?.text ||
        r.fallbackContext ||
        "(fragment not detected)"
      ).slice(0, 100),
      comment: r.comment.notes
        .map((n) => n.body)
        .join(" | ")
        .slice(0, 160)
    }))
  );
})();
```