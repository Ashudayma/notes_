# CLAUDE.md — Lecture Notes in the "LLD Notebook" Style

The reference standard for every set of notes in this folder is **`lld-course-notebook.html`**. When in doubt about format, tone, depth or markup, open that file and copy what it does. The rules below describe it.

---

## 1. The job

The user pastes a **video transcript** (usually Hindi / Hinglish, auto-generated, full of timestamps) and says which lecture number it is, e.g. `-> 29th vdio`. Your job:

1. Understand the lecture completely.
2. Write **English study notes** for it as one new HTML section.
3. Insert that section into the right notes file (e.g. `spring-boot-notes.html`, `lld-course-notebook.html`) in lecture order.
4. Update the Contents list and the progress line.
5. Verify the HTML (see §9) and report briefly.

The transcript is source material, not text to translate. The notes must read as if a strong engineer attended the lecture and wrote clean notes afterwards: **everything technical the lecturer taught, in the order they taught it, plus corrections and the interview-relevant things they missed.**

If the user gives no transcript (only a link), ask them to paste it; do not invent a lecture's content.

---

## 2. What to keep and what to drop

**Drop:** greetings, "like/subscribe", channel promos, notes/GitHub announcements, "is it clear?", repeated re-explanations, filler, jokes, personal stories without technical content.

**Keep (all of it):**
- Every concept, definition, rule and principle taught.
- The lecture's **motivating problem** and the **naive approach** it starts with, and *why that approach breaks*. This "problem → failed fix → better design" arc is the heart of the notes.
- Every worked example, with the lecture's own names (class names, method names, sample data like "Aditya", "Rohit", roll number 1001).
- Demo runs and their output.
- The lecturer's analogies and memorable one-liners (quote them).
- Trade-offs, "when to use / when not to use", comparisons the lecturer draws.
- Interview tips the lecturer gives.

**Never shorten technical content just to be concise.** Remove noise, not knowledge. A 1h30m lecture should produce a long section.

---

## 3. Accuracy: correct, attribute, and add

- **Correct mistakes** instead of copying them: wrong API/method names, wrong status codes, misspoken terms, buggy demo code, outdated terminology, misleading oversimplifications.
  - Small slips go in a callout: `<div class="important"><strong>Transcript slips to ignore:</strong><br> …</div>`.
  - Substantive corrections go in the final **"Corrections &amp; worth knowing"** section, each bullet starting with a bold claim: `<li><strong>Real databases don't snapshot the whole database.</strong> …</li>`.
- **Attribute clearly.** Keep what the lecturer said separate from what you added:
  - "The lecture calls it a marker interface, but…"
  - "The lecturer admits this…"
  - "(Not stated in the lecture; commonly asked.)", "(added)".
- **Versions.** When a fact depends on a language / framework / library version, state the version (e.g. "Spring Boot 3+ uses `jakarta.*`", "Java 21 sealed interfaces").
- **Don't invent.** If unsure, say so explicitly or leave it out. Never make up lecture content, dates, numbers or APIs.
- Use the lecture's language for code (C++, Java, …). If you add a note about another language, label it ("In Java, …").

---

## 4. Section structure

The notes follow the **lecture's own teaching flow**, not a fixed template. Headings (`h3`) are **descriptive**, telling the reader what the part is about ("Why the inheritance-only design collapses", "Setup: simulating robots"), never generic labels like "Concept" or "Section 2".

### 4.1 Typical order for a concept / pattern lecture

1. **The problem.** What the lecture sets up and what goes wrong.
2. **Naive approach and why it breaks.** Often a step-by-step table.
3. **The idea / the pattern.** Roles, classes, how they collaborate.
4. **Worked example from the lecture.** Class names, key methods, short code.
5. **Diagram.** The lecture's UML or flow.
6. **Standard definition.** In a callout.
7. **Standard UML.** If it differs from the lecture's example.
8. **Comparison with related concepts.** Table (X vs Y).
9. **Real-world uses.**
10. **Corrections &amp; worth knowing.**

### 4.2 Typical order for a "build X" / design-project lecture

1. **Requirements.** Functional (and non-functional if discussed).
2. **Entities / core objects.**
3. **Design approach.** Top-down or bottom-up, and why.
4. **Patterns used and why each fits.** "Why X is NOT a Singleton" style reasoning.
5. **Class-by-class walkthrough.** Fields and methods, key logic.
6. **Final UML.**
7. **Demo run.** Numbered steps with output.
8. **Homework / suggested extensions.** If given.
9. **Gaps worth raising in an interview.** Concurrency, scaling, validation, money as `double`, memory ownership, missing features.

### 4.3 Recurring heading names (reuse these exactly)

- `The problem`
- `The pattern`
- `Standard definition`
- `Standard UML diagram`
- `Worked example: …`
- `Demo run`
- `Requirements` / `Functional requirements`
- `Final UML`
- `Real-world uses`
- `Anticipated question: "…"`
- `X vs Y`
- `Homework / suggested extensions`
- `Gaps worth raising in an interview`
- `Corrections &amp; worth knowing` (or `Worth knowing` when nothing needed correcting)

Use `h4` only for sub-parts inside a long `h3` part.

---

## 5. Writing style

- **Tight, concrete prose.** Short paragraphs of 1–4 sentences. Bullets for facts and lists, numbered lists for processes and demo steps, tables for comparisons and step-by-step "fix → new problem" evolutions.
- Explain **why** and **when**, not just **what**. Almost every design choice gets a "why" callout.
- **Bold** the key term or the key claim in a sentence, not whole sentences.
- `<code>` for every class, method, field, keyword, file and command.
- Use the lecture's names and data so the notes match what the learner saw.
- Answer the questions a viewer would ask with `Anticipated question: "…"` headings, e.g. "aren't CompanionRobot/WorkerRobot still inheritance?".
- Cross-reference earlier lectures by number: "the Builder pattern (Lecture 28)".
- End callouts that summarise a philosophy with the lecture's quote: `<strong>One-liner from the lecture:</strong><br>"The solution to inheritance is not more inheritance."`
- English only. Typographic entities: `&mdash;`, `&rarr;`, `&hellip;`, `&ne;`, `&amp;`, `&lt;`, `&gt;`.

---

## 6. HTML building blocks

Copy these exactly. Four-space indentation inside a section.

### 6.1 Section wrapper

```html
<section class="video-section" id="video-N">
    <h2>Lecture N: Title</h2>
    <p class="subtitle">Source: "Exact video title" &middot; 32 min &middot; Code Army</p>

    <h3>The problem</h3>
    <p>…</p>

    …

  </section>
```

Sections are separated by `<hr>`:

```html
  </section>

<hr>

<section class="video-section" id="video-N+1">
```

Use "Lecture N" for course playlists. The `id` is always `video-N`.

### 6.2 Callouts (`.important`): always with a bold label lead-in ending in a colon

```html
    <div class="important"><strong>Why this actually fixes the explosion:</strong><br>
A new robot is just: pick any existing combination and inject it &mdash; no new subclass needed.
    </div>
```

Common labels:

| Kind | Labels |
|---|---|
| Definitions | `Definition:`, `Standard definition:` |
| Reasoning | `Why …:`, `Why X, not Y:`, `The real failure mode:`, `The key assumption: what varies?` |
| Lecture quotes | `One-liner from the lecture:` |
| Extras | `Worth knowing:`, `Worth adding:`, `Worth flagging:`, `Precision note (added):` |
| Errors | `Transcript slips to ignore:` |
| Code / process | `Client code:`, `Step by step:` |

If the callout contains a list, table or paragraph, drop the `<br>` and put the block element directly after `</strong>`. A definition with no label may be a bare `<div class="important">`.

### 6.3 Tables

```html
    <table>
      <caption>Which axis changes?</caption>
      <thead><tr><th>Step</th><th>New requirement</th><th>"Fix" attempted</th><th>New problem created</th></tr></thead>
      <tbody>
        <tr><td>1</td><td>…</td><td>…</td><td>…</td></tr>
      </tbody>
    </table>
```

`<caption>` is optional; use it when a table needs a one-line framing.

### 6.4 Diagrams: Mermaid inside `figure.diagram`, caption **after** the diagram

```html
    <figure class="diagram">
      <div class="mermaid">
classDiagram
  class Strategy { &lt;&lt;abstract&gt;&gt; +run()* void }
  class ConcreteStrategyA { +run() void }
  Strategy <|-- ConcreteStrategyA
  Client --> Strategy
      </div>
      <figcaption>Canonical Strategy Pattern structure</figcaption>
    </figure>
```

- **When to draw.**
  - **UML class diagrams** for every pattern and every "build X" design: the lecture's version, and the standard one if different.
  - **Sequence diagrams** for request/call flows.
  - **Flowcharts** for processes, state machines and architectures.
- **Don't** draw diagrams that add nothing over a bullet list.
- Mermaid code starts at column 0 inside the div.
- **Keep Mermaid text safe:**
  - Escape `<<abstract>>` as `&lt;&lt;abstract&gt;&gt;`.
  - Avoid `[]`, `()`, `{}`, `:` and quotes inside labels and relationship text. Write "2D grid of Symbol", not `Symbol[][]`, and "map key to value", not `map<K,V>`.
  - Keep relationship labels short.
- Captions say what the diagram **shows**, e.g. "Where the 'more inheritance' fix leads — and it only shows ONE of the three behavior axes".
- Arrows follow UML meaning:
  - `<|--` is inheritance.
  - `-->` is association / has-a.
  - `*--` is composition.
  - `o--` is aggregation.
  - `..>` is dependency.

### 6.5 Code

- Short snippets go inline, or several `<code>` lines in a callout joined with `<br>`.
- Longer code (more than ~6 lines) goes in `<pre><code>…</code></pre>`, HTML-escaped (`&lt;`, `&gt;`, `&amp;`).
- Code must be correct and compile. Fix the lecture's bugs and say so.
- Show demo output when the lecture runs a demo.
- Don't paste entire demo files. Show the lines that carry the idea, and describe the rest in bullets (class → fields → methods).

### 6.6 Class-by-class walkthroughs: nested bullet lists

```html
    <ul>
      <li><strong><code>TransactionManager</code></strong> (caretaker) &mdash; one <code>backup</code> memento:
        <ul>
          <li><code>beginTransaction(db)</code> &mdash; delete any old backup; <code>backup = db.createMemento()</code>.</li>
          <li><code>commit()</code> &mdash; delete the backup.</li>
        </ul>
      </li>
    </ul>
```

---

## 7. Interview focus (woven in, not bolted on)

This is **not** a separate Q&amp;A dump at the end. Interview value comes through:

- **"Why" callouts** on every design decision.
- **`X vs Y` comparison tables**: Strategy vs State, Visitor vs Strategy, Mediator vs Observer, Adapter vs Facade, PUT vs PATCH, etc.
- **`Anticipated question: "…"`** headings for likely follow-ups.
- **`Gaps worth raising in an interview`** for project lectures. Name concrete weaknesses and their fixes:
  - Race conditions (and atomic check-and-set).
  - O(n) scans that need an index.
  - `double` for money.
  - Leaked raw pointers.
  - Missing validation.
  - Missing features (unmatch, expiry, per-user limits…).
- **`Corrections &amp; worth knowing`**:
  - Precise terminology.
  - Real-world implementations (how real databases, Git or compilers do it).
  - Language-specific pitfalls (shallow vs deep copy, Java vs C++).
  - Modern alternatives (e.g. Java 21 sealed types + pattern matching vs Visitor).
- **`Standard definition`** callouts giving the textbook (GoF / spec) phrasing an interviewer expects.

The **final lecture of a course** also gets a wrap-up: a cheat-sheet table of everything covered (e.g. Pattern | Lecture | Intent).

---

## 8. File conventions

### 8.1 New notes file

- Copy the whole `<head>` of `lld-course-notebook.html` **verbatim**, including:
  - the Google Fonts link (Caveat + Patrick Hand);
  - the Mermaid 10.9.1 script from cdnjs;
  - the full `<style>` block (lined paper, `.important`, `.toc`, `.dur`, `caption`, `figure.diagram`, dark mode).
- Change only the `<title>`.
- Copy the Mermaid theme `<script>` from the end of that file, just before `</body>`.

### 8.2 Top of the page

```html
<h1>The Spring Boot Notebook</h1>
<p class="subtitle">Handwritten-style study notes &mdash; &lt;course&gt; (Hindi lectures, notes in English)</p>
<p class="subtitle">3 / 40 lectures noted</p>

<div class="toc">
<strong>Contents</strong>
<ol>
  <li><a href="#video-1">Introduction to Spring Framework</a> <span class="dur">&mdash; 43m</span></li>
</ol>
</div>

<hr>
```

### 8.3 End of the page

```html
<hr>

<p class="subtitle">N lectures noted &middot; notes condensed &amp; corrected from the original Hindi lecture, written in English.</p>

<script> … mermaid theme … </script>

</body>
</html>
```

### 8.4 Adding a lecture

1. Insert the section in numeric order, before the footer `<hr>`, with an `<hr>` between sections.
2. Add the Contents entry with its duration. Take the duration from the last transcript timestamp.
3. Update the progress line (`N / total lectures noted`) and the footer.
4. Never rewrite or reformat existing lectures unless asked.

For long sections, write the section to a scratch file first, then insert it with a small script. Avoid shell heredocs with quotes, and `sed` replacements containing `&`.

---

## 9. Before you finish: checklist

- [ ] All noise removed; **all** technical content kept, in the lecture's order.
- [ ] The problem → naive approach → why it breaks → solution arc is clear.
- [ ] Lecture names, examples, demo steps and quotes preserved.
- [ ] Mistakes corrected. Lecture vs added content is clearly attributed. Versions stated where relevant.
- [ ] Descriptive `h3` headings. The recurring names from §4.3 used where they fit.
- [ ] Every callout has a bold label ending in `:` (except bare definitions).
- [ ] Diagrams:
  - [ ] Mermaid-safe text.
  - [ ] Caption after the diagram.
  - [ ] UML arrows correct.
  - [ ] Each one worth including.
- [ ] Code compiles. Run it if a JDK or compiler is available. HTML-escaped.
- [ ] `X vs Y` table, `Anticipated question`, `Gaps worth raising` / `Corrections &amp; worth knowing` where relevant.
- [ ] Contents entry, duration, progress line and footer updated.
- [ ] HTML validity: count open vs close tags for `section`, `div`, `table`, `ul`, `ol`, `li`, `figure`, `pre`, `h3`, `h4`; each pair must match.
- [ ] Report to the user in a few lines:
  - [ ] What was added.
  - [ ] Notable corrections.
  - [ ] Anything uncertain.
  - [ ] Then ask for the next transcript.
