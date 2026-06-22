# pbir CLI learnings (Chess Outcomes)

- **`pbir add filter <Table> <Field> -r "<...>/Some.Visual"` adds a REPORT-level filter, NOT a
  visual-level one** (the `-r` target is the report regardless of the visual path). A stray TopN top-1
  added this way silently filtered EVERY visual to one opening (the bump showed a single line). It does
  not appear under the visual's own filters; find it in `report.json` `filterConfig` and remove with
  `pbir rm "<Report>.Report/filter:<name>"`. To scope a filter to one visual, set it on the visual's
  own `filterConfig` instead.
- **Merging the Deneb tooltip card into the bump = one Vega visual** (`provider:'vega'`, not vegaLite).
  Bump = facet-by-`Opening Family` group with a `line` mark + `symbol` points + end-label `text`; the
  hover card is a `group` mark whose children draw `from` a `cardrow`/`outcome` dataset that is empty
  unless a `sel` signal is set (so the card auto-hides). `sel`/`hx`/`hy` signals update on
  `@pts`/`@hit:mouseover|mousemove` (a transparent strokeWidth-16 `line` named `hit` gives whole-line
  hover; `mousemove` makes the card follow the cursor). Card games must be **summed** across years
  (Total Games is per-Year in the dataset) and White/Black/Draw % **weighted** by games
  (`sum(%·games)/sum(games)`) — plain `max` gives one year's value. The trajectory note derives from
  `RM First Rank`/`RM Last Rank` and is only correct inside the top-5 filter context.

- **Binding a Deneb/custom visual to a report-page tooltip:** `pbir set ...visualTooltip.*` fails with
  `Unknown component: visualTooltip` on a custom visual (not in the core catalog). Fallback: edit the
  visual.json directly and add to `visualContainerObjects` a `visualTooltip` array with `show`=true,
  `type`=`'ReportPage'`, `section`=`'<tooltip page internal name>'`. `section` is the page's `name`
  (the on-disk hash, e.g. `f0786ddaebe0b630`), NOT the displayName. Create+mark the page first with
  `pbir add page` then `pbir pages set-tooltip "<Report>/<Display>.Page" -w 320 -h 240` (sets
  `type:Tooltip`, `displayOption:ActualSize`); set `visibility=HiddenInViewMode` separately. Whether
  Deneb actually forwards the hover identity to the canvas tooltip must be verified in Desktop.
- **pbir references pages/visuals by DISPLAY name, not the on-disk hash folder** (`"<Report>/Opening
  Tooltip.Page"`). Globbing a page path that contains spaces (`".../Opening Tooltip.Page/*.Visual"`)
  matched no visuals — operate per-visual instead. `pbir visuals title` takes `--show/--no-show` (no
  `--hide`, no `-f`).

- **Folding separate textbox "eyebrow" headings into a slicer's own header:** set the slicer
  `header.show=true`, `header.text="LABEL"`, `header.textSize/fontFamily/fontColor` to match the old
  eyebrow, and `header.showRestatement=false` (otherwise the header also prints the current selection
  summary). The slicer `title` object is a *different* thing — it's the selection-pane display name; keep
  `title.show=false`. Then `rm` the textboxes. At 11pt items+header, ~4 vertical-list slicers nearly
  fill the side panel above the bottom chessboard image; freeing room meant nudging the decorative
  chessboard (`scaling: 'Normal'`, z=25000, on top) down a few px. Lay slicers out with equal y-gaps;
  give each ~26px/item + ~26px header or the 5-item Victory slicer gets a scrollbar.
- **Deneb renders blank right after `pbir desktop refresh`.** The custom visual re-initializes on
  every report reload; screenshot ~8s later or it captures an empty chart card. Native visuals are fine immediately.
- **Apostrophes in a Deneb `jsonSpec` literal** (e.g. color-scale domains "Queen's Pawn Game",
  "King's Pawn Game") must be doubled (`''`) inside the single-quote-wrapped expr `Literal.Value`.
  When generating with Python: `"'" + json.dumps(spec).replace("'", "''") + "'"`. Power BI un-doubles on load.
- **`pbir add page "<Report>.Report/<Folder>.Page" -n "<Display>"`** creates the page + a default
  title textbox (folder `Title`); remove it if authoring a custom header. Folder name on disk is a hash, not the path segment.
- **Field validation in `pbir validate` is skipped when Desktop is closed** (thick/`byPath` report
  has no live engine). Verify column/measure names against the TMDL instead. Schema-version warnings
  ("'2.10.0' not available locally") are harmless fallbacks.
- The bump chart's `[Opening Rank by Year]` ranks **within the visual's filter context** (top-5
  families via the TopN filter, ALLSELECTED), so lines read 1–5, not the global per-year rank.
- **A `basicShape` rectangle has a ~12px minimum render height**, so a "2px" divider renders as a
  thick ~12px bar (its container background fills the clamped height). For a true hairline rule use a
  thin **image** visual (a tiny solid PNG, `fit: Fill`) — images honour the exact box height. The
  legacy `shapeType: line` did not render at all.
- **Selection-pane display name = `visual.visualContainerObjects.title[0].properties.text`** (keep
  `title.show=false` so it stays hidden). Setting it names the visual without showing a title bar.
- **Deneb color scales must not hard-code a category domain** (e.g. the 5 opening-family names): a
  slicer that changes the top-N set leaves new members uncoloured (invisible lines / blank swatches).
  Color by a **tie-free rank measure** (`[Opening Overall Rank]` 1..N, domain `[1..5]` → 5 colours)
  so any band's top-5 get consistent colours, and the bump and the list match. The same rank measure
  fixes the Top-5 list overlap (window `rank` produced ties on equal game counts).
- **Dynamic "argmax/argmin" callouts in Deneb:** expose simple per-family measures (first-year rank,
  last-year rank, rank range) + report `row_number` windows in the Vega-Lite spec to pick
  climber/steady/faller. Keeps DAX verifiable (`pbir model -q`) and the selection logic in Vega.
- **Multiple calculation groups must each have a distinct `precedence:`** (the lone Ranking Band
  group had none = default 0; adding more at 0 conflicts). They compose — selecting an item in each
  wraps `SELECTEDMEASURE()` so e.g. Winner=White AND Mode=Rated AND band filters all apply at once.
  A "single-select with All on top" filter = a calc group whose first item is `SELECTEDMEASURE()`
  ("All") + a single-select `strictSingleSelect` slicer bound to its `Name` column (sorted by `Ordinal`).
- **Deneb "table"/bar layout (all marks positioned by `value`/`expr`, no x-scale): use a FIXED
  canvas, not autosize-fit.** With `"autosize":{"type":"fit"}` and a spec whose marks reference the
  `width` signal (e.g. `"x":{"expr":"width-N"}`) but no encoding establishes an x-scale extent, the
  layout is circular (fit sizes the plot *from* the marks, the marks size *from* `width`) → the
  `width` signal collapses to ~200/near-0 and every right-aligned column piles up at one x ("columns
  merged"; a horizontal rule even degenerates into a stray vertical line). **Fix:** set
  `"autosize":{"type":"none"}`, an explicit numeric `"width"/"height"` taken from the Deneb-recorded
  `objects.stateManagement[0].properties.viewportWidth/Height` (e.g. 496×192, 330×180), `"padding":0`,
  and position every column with **fixed pixels** `"x":{"value":N}` — never `width-N`. Size the canvas
  a few px **under** the recorded viewport (e.g. 192→182): an SVG taller/wider than Deneb's drawing
  area triggers a scrollbar; smaller just leaves harmless cream margin. (Two other traps: a header
  `layer` with no own `data` inherits the bound dataset and draws once **per row** — give it
  `"data":{"values":[{}]}`; and mark-level position constants like `"mark":{"y":24}` can blank the
  visual — position via `encoding` `"y":{"value":24}` or a per-datum `"scale":null` pixel field.)
- **A PBIR textbox shorter than its (single-line) text shows a vertical scrollbar (two tiny grey
  square arrows).** An 8pt eyebrow needs ~24-26px box height, not 16 — `pbir visuals resize … --height 26`.
- **A measure that should ignore its own slicer** (e.g. a win%/victory-mix breakdown while a Winner or
  Victory-Status **calc-group** slicer is active) needs `CALCULATE(..., REMOVEFILTERS(DimX), DimX[col]="...")`.
  Bind such breakdowns as **scalar measures** (not the dim as a category) so the calc group can't also
  drop the category rows.
- **After adding calc groups, Desktop shows "calculation groups need to be manually refreshed."**
  Visuals still compute correct values; let the **user** click *Refresh now* + Save (don't run a TOM
  refresh — see [[pbi-desktop-refresh-race]]). Model structural changes need a fresh Desktop load
  (close → edit TMDL → reopen), not `pbir desktop refresh`.
