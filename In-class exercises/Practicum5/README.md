# Practicum: Workflows and Provenance in KNIME

**DSCI 549 — Introduction to Computational Thinking and Data Science**
Week 5 · Fall 2026 · In-class activity

*AI use disclaimer: Claude was used in creating this Practicum!*

You will build a real data analysis without writing code, then look at what it leaves behind as a record of itself. That record is called provenance, and it is the difference between an analysis someone can trust and one they have to take your word for.

Nothing here is graded. Work at your own pace, talk to your neighbor, break things.

## Learning goals

- Treat software as a **black box** with defined inputs, outputs, and parameters.
- Compose black boxes into a **workflow** that performs a multi-step analysis.
- Explain why the same analysis in a spreadsheet leaves **no usable record**.
- Identify what a provenance record must capture beyond the list of operations — in particular, the *reasons* behind each decision.

---

## Before you start

### Install KNIME Analytics Platform

Free, open source, all three platforms. Install before class if you can — it is a 1.5 GB download and campus Wi-Fi will not enjoy thirty people doing this at 8:05am.

1. Download from <https://www.knime.com/downloads>. No account needed.
2. Run the installer. On macOS, right-click the app and choose **Open** the first time.
3. Launch it and accept the default **workspace** location.
4. Dismiss the welcome tour and any sign-in prompt.

### Get the data

Daily vehicle counts at one intersection for 2022–2024. Three years, 1,096 rows.

<https://raw.githubusercontent.com/msoley/DSCI549/master/In-class%20exercises/Practicum5/traffic_counts_practicum.csv>

Right-click → **Save Link As…**. Do not copy-paste from the browser; you will mangle the line endings.

> **If the download misbehaves.** KNIME reads from URLs. In the CSV Reader, switch the file-source dropdown to **Custom/KNIME URL** and paste the link.

### A note on versions

Menu paths below describe KNIME 5.x. Node *names* are stable across versions even when menus move, so search the node repository by name. Ask if you are stuck rather than hunting for ten minutes.

---

## The problem

A bar opened across from a residential intersection on **1 April 2023**. It is busy Friday and Saturday nights.

Residents say they can no longer park on those nights and blame the bar. The owners say neighborhood traffic has been climbing for years anyway.

The city's counter has recorded one number per day since January 2022 — over a year before the bar opened, nearly two years after. Both sides are pointing at the same file. Find out what it says, and leave a record of *how* you found out.

> **Answer this before you open anything.** What would the data have to look like for the residents to be right? For the owners to be right? Be specific about which days, and which stretches of time you would compare.

---

## Part 1 — One node is one program

Every square on the KNIME canvas is a node: a small program with inputs on the left, outputs on the right, a settings dialog, and a traffic light underneath. You never need to know how one works inside. That is the black box idea, made literal.

### Create the workflow

1. In **Space Explorer**, right-click your local space → **Create Workflow**.
2. Name it `DSCI549_traffic_yourlastname`.
3. Double-click to open the canvas.

### Add your first node

1. In **Node Repository**, search `CSV Reader`.
2. Drag it onto the canvas. The traffic light is **red**: the node exists but has no instructions.
3. Double-click it and browse to `traffic_counts_practicum.csv`. Check the preview at the bottom of the dialog, then click **OK**. The light turns **yellow**: configured, not yet run.
4. Right-click → **Execute** (or F7). The light turns **green**.
5. Right-click → open the output table (usually **File Table**). You should have **1,096 rows**.

> **Checkpoint 1.** 1,096 rows, two columns. If you have 1,097, the header row was read as data — reopen the dialog and enable the header option.

### Fix the column names

The header is sloppy: the date column is unnamed and the count column is called `0`, so KNIME invented something like `Column0`. Normal for real data.

1. Add a **Column Renamer** node (`Column Rename` in older versions).
2. Connect it: drag from the black triangle on the right edge of **CSV Reader** to the left edge of **Column Renamer**. That arrow is a table flowing from one program into the next.
3. Rename the date column to `Date` and the count column to `Vehicles`. Execute.

> **Why this matters.** The file on disk is untouched. KNIME never edits your source data — it reads it and passes copies downstream. Compare this to renaming headers in Excel and hitting Save.

---

## Part 2 — A workflow is a multi-step program

### Look before you clean

1. Add a **Line Plot** connected to **Column Renamer**.
2. Plot `Vehicles`. Execute, open the view.
3. Say out loud what you see before reading on.

Two features. A slow wave rising and falling three times — one cycle per year, and real. And scattered days dropping straight to the floor while everything else sits between 17 and 105.

Those floor values are of exactly two kinds, and they are **not** equally bad.

**−1 is impossible.** A sensor cannot count a negative car. A value that *cannot* occur is a **sentinel** — a number the equipment writes to mean "no reading," chosen because it can never be mistaken for data. 

**0 is only implausible.** A street could have an empty day. But the quietest genuine day in three years recorded 17, and a jump from 17 to 0 on thirty-three scattered days is not how traffic behaves. The likeliest story is that the logger ran and recorded nothing.You are about to make a judgement call. Write it down.

### Remove the sentinel values

1. Add a **Row Filter** after **Column Renamer**.
2. **Exclude** rows where `Vehicles = -1`. Execute.
3. **1,074** rows remain.

### Remove the implausible values

1. Add a **second Row Filter** after the first.
2. Keep only rows where `Vehicles > 0`. Execute.
3. **1,041** rows remain.

> **A design choice worth arguing about.** A single filter on `Vehicles > 0` would remove both sets at once, with identical numbers and one node less.

### Look again

1. Add a second **Line Plot** after the second filter.
2. Add a **Statistics** node after the second filter. The mean should be near **52.9**.
3. Keep the *first* line plot. Seeing before and after side by side, months later, is much of the point.

> **Checkpoint 2.** 1,041 rows, mean ≈ 52.9, a plot that looks like plausible traffic with an annual wave. All lights green.

---

## Part 3 — Answering the actual question

The residents' claim is about *days of the week*; the owners' claim is about a *stretch of time*. Both are locked inside a text column.

1. Add **String to Date&Time** after your second Row Filter. Select `Date`, set **New type** to **Date** (not Date&Time — there is no clock time here), format `M/d/yy`. Execute.
2. Add the date-part extractor. Your version calls it **Date&Time Part Extractor** or **Extract Date&Time Fields** — same node. Search for `extract` or `part` alone; the full name fails in some versions because of the ampersand. Tick **Year**, **Month (number)**, **Day of week (name)**. Execute.

### 3.1 — The monthly view

Start with the owners' claim: has traffic been climbing?

1. Add **Date&Time to String** connected to the date-part extractor. Pattern `yyyy-MM`, new column `YearMonth`. Execute — you should see `2022-01` through `2024-12`.
2. Add **GroupBy** after it. Group by `YearMonth` only. Aggregate `Vehicles` with **Mean**, and again with **Count**. Execute; 36 rows.
3. Add **Sorter** on `YearMonth` ascending. Do not skip this.
4. Add **Bar Chart**: category `YearMonth`, value the mean.
5. Compare to the figure, and decide what it shows before reading on.

![Mean daily vehicle count by month across all three years. The dotted line marks the bar's opening.](fig1_monthly.png)

The dominant feature is a wave: peaks near 65 every July, troughs near 39 every January. The quietest month is barely half the busiest. There is also a slow climb across years — 47.6, 52.8, 58.4.

So the owners are telling the truth about the trend, and it predates them. What the chart does **not** show is any step at April 2023.

> **Why the two extra nodes were necessary.**
>
> - **A chart needs one label per bar.** Grouping by `Year` and `Month (number)` gives two identifying values and no single name — and Bar Chart will not offer a numeric column as a category at all.
> - **GroupBy makes no promise about row order.** It groups; it does not sort. Without the Sorter your months arrive scrambled.
> - **`yyyy-MM`, not `yyyy-M`.** `YearMonth` is text, and text sorts character by character, so `2022-10` would land between `2022-1` and `2022-2`. The leading zero is what makes an alphabetical sort come out chronological.
>
> Both decisions are now nodes on the canvas, which means they are written down. Typing labels by hand or dragging bars into order would have recorded nothing.

### 3.2 — Two things this chart cannot tell you

**It collapsed the days of the week.** Fri and Sat are two days in seven, so a change confined to them is spread across the other five — about **71% averaged away**. Sixteen extra cars on bar nights would appear here as a bump under five, in a chart whose bars already swing by twenty-six across a year.

**It mixed up season with time.** The bar opened 1 April. Comparing the six months before to the six months after means comparing **October–March** to **April–September**: winter to summer. Traffic would have risen over that window with no bar at all.

You cannot fix the second problem by adding more data. The before and after periods are not comparable, because the calendar moved when the bar did.

> **The general lesson.** Aggregation destroys variation *within* the groups you collapse. If your question is about a difference inside a group and you aggregate over it, the answer is gone before you start.
>
> And a before/after comparison means something only if everything else held still. Here the season did not.

### 3.3 — Keeping season and day type separate

Stop collapsing. Split by day type *and* season, keep years apart, compare like with like.

1. Add a **Rule Engine** connected to the date-part extractor — the same node feeding your monthly GroupBy. New column `DayType`:

   ```
   $Day of week (name)$ IN ("Friday","Saturday") => "Fri-Sat"
   TRUE => "Sun-Thu"
   ```

2. Add a **second Rule Engine** connected to the **first one's output** — in series, not side by side. Each adds one column and you need both. New column `Season`:

   ```
   $Month (number)$ >= 4 AND $Month (number)$ <= 9 => "Apr-Sep"
   TRUE => "Oct-Mar"
   ```

3. Add a **GroupBy** connected to the **second** Rule Engine. Group by `Year`, `Season`, and `DayType`. Aggregate `Vehicles` with **Mean** and **Count**. Twelve rows.
4. Add a **Sorter** on `Year`, `Season`, `DayType`.

> **Do not connect the Rule Engines to your 3.1 GroupBy.** It is the nearest node and the obvious place to continue. It will not work, and the reason matters more than the fix: that GroupBy already collapsed 1,041 daily rows into 36 monthly ones, and there is no day of the week for the month of March. The aggregation destroyed the information; nothing downstream can recover it. Branch from before an aggregation, never after.

#### First, measure the confound

Fill in the four **2022** cells. No bar existed in 2022, so this is the neighborhood's own behavior.

| 2022 only — no bar yet | Fri-Sat | Sun-Thu | Weekend premium |
|---|---|---|---|
| **Apr-Sep (summer)** | | | |
| **Oct-Mar (winter)** | | | |

The last column is Fri-Sat minus Sun-Thu.

> **Stop and compare those two rows.** The weekend premium is not a constant — large in summer, near zero in winter, in a year with no bar. People go out more on warm weekends.
>
> Now reread the tempting comparison from 3.2. Six months before against six months after compares a winter premium to a summer one. It would have found a large "bar effect" in 2022.

#### Now measure the bar

Same six calendar months every year, so season is held still and only the years differ.

| Apr-Sep | Fri-Sat | Sun-Thu | Weekend premium |
|---|---|---|---|
| **2022 — before** | | | |
| **2023 — after** | | | |
| **2024 — after** | | | |

Your estimate is the change in weekend premium from 2022 to 2023. The 2024 row is a check: a real effect should still be there.

> **Why this works.** Anything that moved the whole neighborhood — the summer peak, the yearly growth, the weather — lifted Fri-Sat and Sun-Thu together and cancels in the subtraction. Anything that hit only bar nights does not cancel.
>
> This has a name: **difference-in-differences**. You have just built one.

> **Checkpoint 3.** State in one sentence per claim whether it is supported and which comparison establishes it. Say roughly how wrong you would have been comparing the six months before the opening to the six months after.
>
> One loose end: the `Oct-Mar` row for 2023 is meaningless, because Jan–Mar 2023 is before the bar and Oct–Dec is after. Your grouping cut through the event you are studying. What would you change?

---

## Part 4 — Provenance: the workflow is the record

Done in a spreadsheet, this analysis would have been correct and would have left nothing behind. Fifty-five rows would simply be absent, with no record that they ever existed or why they left.

Your canvas is not a picture of the analysis. It is the analysis — every operation still there, in order, with its settings, still executable.

### 4.1 — Record the reasons, not just the operations

A node named "Row Filter" says *what* happened, never *why*. The why is the part that cannot be recovered later.

1. Click the text label under your first Row Filter and type a real explanation: *Drop −1 — sensor's offline sentinel. A count cannot be negative, so nothing real is lost.*
2. Now the harder one, which has to admit what it is doing: *Drop 0 — assumed logger error, not a genuinely empty street. Lowest real reading in three years is 17. Not confirmed with the city.*
3. Right-click empty canvas → add a **workflow annotation**. Draw one around both filters, label it **Data cleaning**.
4. Add a second annotation around Part 3, labeled **Analysis**.

> **The honest version.** The first annotation states a fact; the second states an assumption and labels it as one. Nobody can tell from the data which of your filters was safe and which was a guess — unless you wrote it down. "Not confirmed with the city" is the sentence that saves you when someone checks.

### 4.2 — Record who, when, and what this is

1. Open the workflow **Description** / **Metadata** panel (in 5.x, the side panel when the workflow is selected in Space Explorer).
2. Describe what question it answers, what data it uses, and the full source URL — not "the class file."
3. Add yourself as author and tags such as `traffic`, `data-cleaning`.

This is **metadata**: data about your data and your process. Note that KNIME provides the fields and nothing makes you fill them in. Most people don't, which is most of why most analyses are not reproducible.

### 4.3 — Reproducibility

1. Right-click the workflow → **Reset**. Every light goes yellow, every result is discarded.
2. **Execute All** (Shift+F7).
3. Confirm 1,041 rows and the same mean. You rebuilt the entire analysis in seconds from a record rather than from memory.

### 4.4 — Make it portable

1. Add a **CSV Writer** at the end of your cleaning branch, writing `traffic_clean.csv`. Execute.
2. Export the workflow: right-click in Space Explorer → **Export** (older versions: **File → Export KNIME Workflow…**). Save the `.knwf`.
3. Compare the sizes of the two files, and what each contains.

The CSV holds 1,041 numbers and no history. The `.knwf` holds every node, setting, annotation, and your metadata, plus the graph connecting them — an executable account of how those numbers came to be. One is a result. The other is a result **plus its justification**.
