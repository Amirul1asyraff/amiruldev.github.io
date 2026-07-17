# Portfolio Revamp — Planning Doc

**Status:** DECIDED 2026-07-17 — Direction **B (Swiss Modernism 2.0)**. S0 (bug fixes) authorized and in progress.
**Date:** 2026-07-17
**Brief:** "revamp this website, make it original design, I want as my portfolio, use the max design"
**Live at:** `skycore.my` (per `CNAME`) — served from the `amiruldev.github.io` repo.

---

## 1. Scope — what a revamp actually touches

Every project page loads the same stylesheet. This is not a one-page job.

```
                        index.css  (927 lines — ONE stylesheet)
                             ▲
        ┌────────┬───────────┼───────────┬──────────┬──────────┐
        │        │           │           │          │          │
   index.html  student-   payment-    home-lab-  sports-   library-
    (434 ln)   management  gateway-   deployment booking-   system
               -api        flow                  management
        │        │           │           │          │          │
        └────────┴───────────┴───────────┴──────────┴──────────┘
                     6 pages, 1 shared CSS file
```

**Invariant: `index.css` is shared by all six pages. Restyling it restyles the project pages
whether we intend to or not — so they are in scope by definition, not by choice.**

Stack: static HTML/CSS/vanilla JS on GitHub Pages. No build step, no framework, no Tailwind.
*(Note: your global standing rule "always use the latest Laravel/Livewire/Tailwind" doesn't apply here —
this repo is deliberately plain static. Flagging it so the deviation is explicit, not accidental.)*

---

## 2. Bugs found in the current site — ranked

These are real, verified in the files, and independent of any redesign.

| # | Severity | Bug | Evidence |
|---|----------|-----|----------|
| 1 | **CRITICAL** | **The "Email Me" button goes to a placeholder.** The single most important conversion point on the site links to `mailto:your@email.com`. | `index.html:387` |
| 2 | **HIGH** | **No mobile navigation at all.** `.nav-links { display: none }` under 768px with no hamburger, no drawer. Phone visitors can only scroll. | `index.css:768` |
| 3 | **HIGH** | **Broken favicon on the homepage.** Points at `assets/profile.jpeg`; the file on disk is `profile.png`. The five project pages get this right — only the homepage is wrong. | `index.html:17` |
| 4 | **HIGH** | **No Open Graph / Twitter Card tags.** Sharing `skycore.my` to LinkedIn, WhatsApp, or a recruiter's inbox renders a naked link with no title, image, or description. | `index.html:9-14` |
| 5 | MEDIUM | **Dead CSS.** `.nav-socials` is styled in four places; no element with that class exists in any page. The socials live inline in the profile card as `.main-socials`. | `index.css:152,160,166,769` vs `index.html:97` |
| 6 | MEDIUM | **Inline `onmouseover`/`onmouseout` handlers** for a hover effect CSS already does. No keyboard equivalent, and hostile to any future CSP. | `index.html:100,105` |
| 7 | MEDIUM | **Anchor targets hide under the fixed nav on mobile.** Nav occupies 72px (12px top + 60px tall); mobile sections have 60px padding. No `scroll-margin-top` anywhere. | `index.css:757-783` |
| 8 | LOW | **Scroll smoothing declared twice** — CSS `scroll-behavior: smooth` *and* a JS `scrollIntoView` handler that `preventDefault()`s every anchor. The JS is redundant. | `index.css:23`, `index.html:403-410` |
| 9 | LOW | **Render-blocking CDNs.** Google Fonts + the entire Font Awesome library (~75KB CSS) for ~8 icons. | all 6 pages |
| 10 | MEDIUM | **~1.9 MB of PNGs.** `profile.png` is 800×800 / 398 KB but renders at 120×120. The three project images are 640×640 PNGs (417/633/446 KB) displayed at `aspect-ratio: 16/10`, so they're heavily cropped *and* oversized. Found during S0; deferred to S6 (WebP + correct dimensions + `loading="lazy"`). | `assets/` |

### S0 status — DONE, verified in-browser, awaiting commit

Fixed: **3, 4, 5, 6, 7, 8** (+ bug 6 also fixed on all five project pages — the same
inline-styled `onmouseover` back-link was copy-pasted into each, byte-identical).
Deferred to S6: **9, 10**.
**Bug 1 (placeholder email) is BLOCKED** — needs Amirul's real address (→ §9 Q2).

Verified by driving a real browser (Playwright), not by grep:
18/18 checks at 375px + 1440px + reduced-motion, 0 console errors, all 5 project pages clean.
Both new guards **mutation-proven** (broke the fix → test went red → restored, diff-verified
byte-identical per RG-11: never revert a mutation with git when the work is uncommitted).

### Traps found while fixing S0 — save-worthy (both bit me in this session)

**T1 — `backdrop-filter` on an ancestor makes it a containing block for `position: fixed` children.**
The mobile menu was written `position: fixed; top: 84px; left: 12px`, which *looks* viewport-relative.
It isn't: `.nav` has `backdrop-filter`, so — exactly like `transform` or `filter` — it becomes the
containing block, and the panel resolved against the nav pill instead (landing at 97/25, not 84/12).
It rendered close enough to look correct, which is what makes it dangerous: the offsets are lies that
happen to agree with the truth, until the nav's height changes. The same stacking context also
**neutralises `backdrop-filter` on the child**, so the panel's own blur silently did nothing.
*Fix: be honest — `position: absolute; top: calc(100% + 12px); left: 0; right: 0` tracks the nav.*

**T2 — never animate the element that is also the scroll anchor target.**
`scroll-margin-top: 84px` was defeated by the reveal animation's `translateY(20px)`: the browser scrolls
to the target while the section is still translated down, then the transform settles to 0 and drags the
section **up by exactly the translate distance** — landing it at 64px, under a nav whose bottom is 72px.
`84 − 20 = 64`, precisely. *Fix: animate the inner `.container`, never the `<section>` itself, so the
anchor's box never moves.*

**Both were invisible to a green grep and to a passing checklist.** T2 was only caught because the first
version of the test measured *mid-scroll* and passed for the wrong reason — the classic worthless
assertion (→ UI rule #8, WL-34). Waiting for scroll to settle turned it red.

---

## 3. The actual problem: it isn't bad, it's *generic*

Let me be straight with you, because this is the whole reason you asked.

The current site is **competently built** and **completely anonymous**. Dark navy `#030712`, glassmorphism
cards, a cyan→blue→purple gradient on the headline, floating pill nav, ambient blurred orbs, Outfit font.
That is the default look of every AI-generated developer portfolio from 2023 onward. A recruiter who
screens twenty portfolios a week has seen this exact page twenty times. Nothing in it is *yours*.

The design engine independently reached the same verdict — for a portfolio it lists the anti-patterns as
literally **"corporate templates"** and **"generic layouts"**, and recommends a *neutral* background so the
work carries the page instead of the chrome.

**So "original" isn't a coat of paint. It means picking a point of view and committing.**

---

## 4. Content problems that outrank the visuals

A gorgeous site you can't be contacted through is a brochure with no phone number. These cost you
interviews more than any palette will:

- **Two of five projects are vaporware.** "Sports Booking Management" and "Library System" are both
  tagged *Upcoming Project*. A recruiter reads an Upcoming card as *nothing here*. Five cards where two
  are empty is weaker than three cards that are all real.
- **Two of three "stats" aren't stats.** "3+ Projects Shipped" is a number. "API — Focused Dev" and
  "Growth — Always Learning" are adjectives wearing a metric's costume. They read as filler, and filler
  next to a real number makes the real number look like filler too.
- **"3+ Projects Shipped" contradicts the grid**, which shows five cards.
- **No CV/résumé download.** For a junior role hunt this is the single most-requested artifact.
- **No live demos, no per-project repo links.** Every project ends at an internal "View Details" page.
  Nothing links to running code or to GitHub.
- **The CPRE-FL certification is buried** in the third card of the About grid. It is genuinely your
  most differentiating credential — most junior devs do not have a formal requirements-engineering cert —
  and it's below the fold, in a box, after two cards of generic prose.

---

## 5. Three directions — pick one

### Direction A — Editorial Monochrome *(the engine's own pick)*
Off-white `#FAFAFA`, near-black `#09090B`, one blue accent `#2563EB`. Playfair Display headlines over
Source Serif 4 body. Reads like a printed essay or a design annual.

- **Feels like:** a considered personal manifesto
- **Pro:** almost no dev portfolio is a light-mode serif editorial piece — instantly non-generic. WCAG AAA, excellent performance.
- **Con:** serif body copy reads "writer/designer" more than "engineer". Risks looking like a blog.

### Direction B — Swiss Modernism 2.0 ★ *my recommendation*
Strict visible 12-column grid, mathematical 8px spacing, Inter/Helvetica, `#000` / `#FFF` / `#F5F5F5`,
**one** vibrant accent. Asymmetric balance, zero decoration, ruthless hierarchy.

- **Feels like:** an engineer who documents, specs, and traces things properly
- **Pro:** **this one means something for you specifically.** You are CPRE-FL certified in requirements
  engineering — your whole differentiator is *structure, precision, traceability*. A site built on a
  strict, visible, documented grid **is the argument.** The form proves the claim instead of asserting it.
  Also WCAG AAA, excellent performance, and trivially maintainable with no build step.
- **Con:** demands discipline. A Swiss grid with one sloppy alignment looks broken, not minimal.

### Direction C — Brutalism / Kinetic
Raw, `border-radius: 0`, 2px hard borders, oversized uppercase type, acid yellow `#DFE104` on
rich black, instant transitions, marquee rows.

- **Feels like:** a zine, a statement, someone who does not care what you think
- **Pro:** unforgettable. Nobody screens past it.
- **Con:** high risk for a junior hunting roles with conservative Malaysian employers. It's a bet on the
  reader having taste. Some will read it as broken.

**My call: B.** A is beautiful but says "I write". C is a coin flip on the reader. **B is the only one
where the design is evidence for the thing you're claiming on your CV.** That's what makes a portfolio
original rather than merely different — it's *about you* instead of *about a trend*.

---

## 6. Proposed information architecture

Ordering follows the engine's Portfolio Grid pattern (Hero → Work → About → Contact), reordered so your
credential and your real work come before your prose.

```
┌──────────────────────────────────────────────────────────┐
│ NAV  Amirul.dev        Work  Approach  About  Contact    │  ← + working mobile menu
├──────────────────────────────────────────────────────────┤
│ HERO                                                     │
│   Name · role · one sharp sentence                       │
│   [ View Work ]  [ Download CV ]      ← real CTAs        │
│   ── CPRE-FL certified · IREB ──      ← promoted, inline │
├──────────────────────────────────────────────────────────┤
│ WORK        ← the 3 REAL projects, full width, no filler │
│   ┌────────┐ ┌────────┐ ┌────────┐                      │
│   │ API    │ │ Payment│ │ HomeLab│  each: repo + demo    │
│   └────────┘ └────────┘ └────────┘                      │
│   › In progress: Sports Booking, Library  (a LINE,       │
│                                            not 2 cards)  │
├──────────────────────────────────────────────────────────┤
│ APPROACH    ← the SRS→SDD→RTM→Code pipeline, promoted    │
│   the one thing no other junior portfolio has            │
├──────────────────────────────────────────────────────────┤
│ ABOUT       ← short. journey timeline folded in.         │
├──────────────────────────────────────────────────────────┤
│ CONTACT     ← a REAL mailto + LinkedIn + GitHub          │
└──────────────────────────────────────────────────────────┘
```

**Invariant: every section either shows real work or gives the reader a way to act.
Nothing on the page exists to fill space.**

Key moves vs. today:
- **Upcoming projects demoted from cards to a one-line note.** Two empty cards cost more than they earn.
- **The CPRE-FL pipeline promoted to its own section.** It's the differentiator; stop hiding it.
- **Journey timeline folded into About.** Three items don't need a top-level section.
- **Stats row cut** unless we can find three real numbers. Better nothing than filler.

---

## 7. Design tokens (Direction B)

Written as plain CSS custom properties — no build step, matching the repo as it is.

```css
:root {
  /* Color — monochrome + exactly one accent */
  --color-bg:        #FFFFFF;
  --color-surface:   #F5F5F5;
  --color-fg:        #000000;
  --color-fg-muted:  #52525B;   /* verify ≥4.5:1 on both bg and surface */
  --color-border:    #000000;
  --color-accent:    #2563EB;   /* ONE accent. no gradients. */
  --color-on-accent: #FFFFFF;

  /* Spacing — strict 8px base */
  --space-1: 8px;   --space-2: 16px;  --space-3: 24px;
  --space-4: 32px;  --space-6: 48px;  --space-8: 64px;  --space-12: 96px;

  /* Grid */
  --grid-cols: 12;
  --grid-gap: var(--space-2);
  --max: 1200px;

  /* Type */
  --font-sans: 'Inter', system-ui, sans-serif;
  --font-mono: 'JetBrains Mono', ui-monospace, monospace;  /* labels, metadata */
  --radius: 0px;      /* Swiss: no rounding */
}
```

**Dark mode:** the current site is dark-only. Direction B is light-first. I'd add a
`prefers-color-scheme: dark` pair and design both together rather than inverting — but **that is a real
chunk of extra work.** Call it: light-only first, or both from day one? *(→ open question Q5)*

---

## 8. Build plan — sliced, each slice reviewable

| Slice | What | Why first |
|-------|------|-----------|
| **S0** | Fix the 9 bugs on the **current** design | Your live site stops being broken today, regardless of whether the revamp ships. The placeholder email alone justifies it. |
| **S1** | Tokens + grid + typography in `index.css` | The foundation everything else sits on |
| **S2** | Nav (incl. **working mobile menu**) + hero + CPRE-FL promotion | Above the fold |
| **S3** | Work section — 3 real projects, repo/demo links | The actual point of a portfolio |
| **S4** | Approach / About / Contact | |
| **S5** | Re-skin the 5 project detail pages to the new system | They share the CSS; they'll be half-broken until this lands |
| **S6** | OG tags, favicon, self-host fonts, drop Font Awesome for inline SVG | Perf + shareability |

**S0 is independently shippable and I'd do it first** even if you hate all three directions.

---

## 9. Open questions — I need your answers before writing code

1. **Which direction — A, B, or C?** (I recommend B.)
2. **What email address goes on the site?** I will not guess this one. The address I have in context
   (`pejabat.revenue@gmail.com`) looks like an office/work account, not something you'd hand a recruiter.
3. **Do you have a CV/résumé PDF to link?** If yes, drop it in `assets/` and I'll wire the button.
4. **Live demo URLs + GitHub repo links per project?** Three real projects with no runnable link is the
   biggest credibility gap on the site. Even one live demo changes the read.
5. **Light-only, or light + dark from the start?** (Affects S1 scope meaningfully.)
6. **Keep the two "Upcoming" project pages?** My proposal demotes them to a line on the homepage, but the
   detail pages can stay reachable if you want them.
7. **Any real numbers for a stats row?** (GitHub contributions? Months of experience? API endpoints
   shipped?) If not, I cut the row rather than pad it.

---

## 10. Git workflow

Per your standing rule — **issue first, then branch, then commits referencing it, then PR.**

```
gh issue create  →  #N
git checkout -b feat/N-portfolio-revamp    (off main — this repo has no dev branch)
commits: "Refs #N"
PR → main: "Closes #N"
```

Two notes:
- This repo has **no `dev` branch** — it's `main` only. So PRs target `main` unless you want a `dev` added.
- **`main` is what GitHub Pages serves.** Merging publishes to `skycore.my` immediately. There is no
  staging. Worth knowing before we merge anything half-done.

---

**Next step: answer §9, and I'll either start S0 (bug fixes, safe, independent) or wait for the full go.**
</content>
</invoke>
