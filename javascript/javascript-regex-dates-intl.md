# JavaScript — Regex, Dates & Intl (Parts 11–12)

Regular expressions end to end, then the `Date` object's genuine design flaws, the `Intl` formatting APIs that fix the presentation half, and `Temporal`, which is the replacement for the rest.

> **About this document.** Part of the [JavaScript Mastery Guide](./javascript-mastery-guide.md) set — see that document's "About This Document" section for the framing (curiosity-driven, hands-on), scope, tier conventions, sourcing rule and update instructions. Written against ECMAScript 2026; Baseline data verified September 6, 2026.

---

## Table of Contents

1. [Part 11 — Regular Expressions](#1-part-11--regular-expressions)
   - [Beginner](#part-11-beginner) · [Working Knowledge](#part-11-working) · [Advanced](#part-11-advanced) · [Mastery](#part-11-mastery) · [Cheat sheet](#part-11-cheatsheet)
2. [Part 12 — Dates, Times & Intl](#2-part-12--dates-times--intl)
   - [Beginner](#part-12-beginner) · [Working Knowledge](#part-12-working) · [Advanced](#part-12-advanced) · [Mastery](#part-12-mastery) · [Cheat sheet](#part-12-cheatsheet)

---

## 1. Part 11 — Regular Expressions

<a id="part-11"></a>

### Beginner — patterns, flags, and the four methods

<a id="part-11-beginner"></a>

```js
const re = /\d+/g;                    // literal — compiled once, at parse time
const re2 = new RegExp("\\d+", "g");  // constructor — for dynamic patterns

"abc123".match(/\d+/);                // ["123", index: 3, ...]
/\d+/.test("abc123");                 // true
"a1b2".replace(/\d/g, "#");           // "a#b#"
"a1b2".split(/\d/);                   // ["a", "b", ""]
```

Note the doubled backslash in the constructor form: `"\\d"` in a string literal is the two characters `\d`. This is the most common source of "my dynamic regex does not work".

#### The character vocabulary

```text
Literals & classes
  abc      the literal characters
  .        any character except newline (unless the s flag)
  \d \D    digit / non-digit
  \w \W    word char [A-Za-z0-9_] / non-word
  \s \S    whitespace / non-whitespace
  [abc]    any one of a, b, c
  [^abc]   any character except a, b, c
  [a-z]    a range

Quantifiers                          Anchors
  *        0 or more                   ^      start of string (or line, with m)
  +        1 or more                   $      end of string (or line, with m)
  ?        0 or 1                      \b     word boundary
  {n}      exactly n                   \B     non-boundary
  {n,}     n or more
  {n,m}    between n and m           Groups
  *? +? ?? lazy versions               (…)    capturing group
                                       (?:…)  non-capturing group
Alternation                            (?<n>…) named group
  a|b      a or b                       \1     backreference
```

#### The flags

| Flag | Name | Effect |
|---|---|---|
| `g` | global | Find all matches, not just the first |
| `i` | ignore case | Case-insensitive |
| `m` | multiline | `^`/`$` match at line boundaries |
| `s` | dotAll | `.` also matches newline |
| `u` | unicode | Correct handling of code points; enables `\p{…}` |
| `v` | unicodeSets | `u` plus set operations and string properties (ES2024) |
| `y` | sticky | Match only at exactly `lastIndex` |
| `d` | hasIndices | Add `.indices` with start/end offsets per group |

Use `u` (or `v`) by default. Without it, `.` and character classes operate on UTF-16 code units, so a single emoji is two "characters" and ranges break at the BMP boundary.

```js
// ❌ No u flag: the emoji is two code units, so this matches half of it.
"👋".match(/./)[0].length;      // 1 — a lone surrogate, mojibake

// ✅ With u, . matches a whole code point.
"👋".match(/./u)[0].length;     // 2 — the full character
```

> **Try It.** Run `console.log("café".match(/\w+/u)[0], "café".match(/[\p{L}]+/u)[0])`. Expected: `caf` then `café`. `\w` is ASCII-only by definition; Unicode property escapes are how you match real letters.

### Working Knowledge — capture groups and the extraction methods

<a id="part-11-working"></a>

#### Groups

```js
const m = "2026-09-06".match(/(\d{4})-(\d{2})-(\d{2})/);
m[0];       // "2026-09-06"  — the whole match
m[1];       // "2026"        — first group
m.index;    // 0

// Named groups are far more maintainable.
const { groups } = "2026-09-06".match(/(?<y>\d{4})-(?<m>\d{2})-(?<d>\d{2})/);
groups.y;   // "2026"
groups.m;   // "09"

// Destructure directly.
const [, year, month, day] = "2026-09-06".match(/(\d{4})-(\d{2})-(\d{2})/);
```

```js
// ❌ Positional groups: adding a group at the front renumbers everything.
const [, name, version] = line.match(/^(\w+)@([\d.]+)$/);

// ✅ Named groups survive edits.
const { name, version } = line.match(/^(?<name>\w+)@(?<version>[\d.]+)$/).groups;
```

Use non-capturing groups `(?:…)` when you only need grouping for alternation or quantification — capturing has a cost and clutters the result:

```js
/(?:https?):\/\/(?<host>[^/]+)/.exec("https://example.com/x").groups.host;
// "example.com"
```

#### `match` vs. `matchAll` vs. `exec`

The behaviour of `match` **changes completely** with the `g` flag, which surprises everyone once:

```js
const s = "a1b2c3";

s.match(/\w(\d)/);        // ["a1", "1", index: 0, …]   ← first match WITH groups
s.match(/\w(\d)/g);       // ["a1", "b2", "c3"]         ← all matches, NO groups, NO index
```

When you need all matches *and* their groups, use `matchAll` (ES2020):

```js
for (const m of s.matchAll(/(?<letter>\w)(?<digit>\d)/g)) {
  console.log(m.groups.letter, m.groups.digit, m.index);
}
// a 1 0 / b 2 2 / c 3 4

const all = [...s.matchAll(/(\w)(\d)/g)];    // materialise if you need an array
```

`matchAll` **requires** the `g` flag and throws without it — a deliberate guard against an infinite loop.

#### `replace` and `replaceAll`

```js
"a-b-c".replace("-", "+");        // "a+b+c"?  NO → "a+b-c"  (string arg = first only)
"a-b-c".replaceAll("-", "+");     // "a+b+c"
"a-b-c".replace(/-/g, "+");       // "a+b+c"

// $-substitutions in the replacement string.
"2026-09-06".replace(/(?<y>\d+)-(?<m>\d+)-(?<d>\d+)/, "$<d>/$<m>/$<y>");
// "06/09/2026"
// $&  whole match   $1 $2  numbered groups   $<name>  named   $$  a literal $

// A function replacement gets the match, groups, offset, and full string.
"a1b2".replace(/(\w)(\d)/g, (match, letter, digit, offset) =>
  `${letter.toUpperCase()}${Number(digit) * 2}`);
// "A2B4"

// With named groups, the groups object is the LAST argument.
"a1".replace(/(?<l>\w)(?<d>\d)/, (...args) => {
  const groups = args.at(-1);
  return `${groups.l}${groups.d}`;
});
```

`replaceAll` throws if given a non-global regex — again a deliberate guard, since "replace all with a non-global pattern" is always a mistake.

### Advanced — `lastIndex`, lookaround, and building regexes safely

<a id="part-11-advanced"></a>

#### The `lastIndex` trap

A regex object with `g` or `y` carries **mutable state**: `lastIndex`, the position where the next search begins. Reusing such a regex across calls produces alternating results.

```js
// ❌ A shared global regex is stateful. This is a genuine production bug class.
const re = /\d+/g;
re.test("123");     // true   — lastIndex is now 3
re.test("123");     // false  — starts at index 3, finds nothing
re.test("123");     // true   — lastIndex reset to 0 after the failure
```

```js
// ✅ Option A: no g flag when you only need a boolean.
const re = /\d+/;
re.test("123");     // true, always

// ✅ Option B: create the regex at the point of use.
const check = (s) => /\d+/g.test(s);

// ✅ Option C: reset explicitly if you must reuse it.
re.lastIndex = 0;
```

This bites hardest with module-level regex constants, which look like the right refactor and quietly introduce state. [It is one of the most-reported JavaScript regex surprises](https://stackoverflow.com/questions/1520800/why-does-a-regexp-with-global-flag-give-wrong-results). The safe rule: **a module-level regex constant must not have the `g` flag** unless it is only ever used with `matchAll`/`replace`, which reset `lastIndex` themselves.

The same state is what makes the `exec` loop work:

```js
const re = /(\w)(\d)/g;
let m;
while ((m = re.exec("a1b2")) !== null) {
  console.log(m[0], m.index);      // a1 0 / b2 2
}
// Modern equivalent, without the mutable state:
for (const m of "a1b2".matchAll(/(\w)(\d)/g)) { }
```

#### Lookaround

```js
// Lookahead
/\d+(?=px)/.exec("100px")[0];        // "100"  — followed by "px", not included
/\d+(?!px)/.exec("100em")[0];        // "100"  — not followed by "px"

// Lookbehind (ES2018) — variable-length is allowed in JS, unlike many languages
/(?<=\$)\d+/.exec("$100")[0];        // "100"  — preceded by "$"
/(?<!\$)\d+/.exec("€100")[0];        // "100"  (actually "00" — see below)

// Thousands separators, the classic lookahead application.
"1234567".replace(/\B(?=(\d{3})+(?!\d))/g, ",");     // "1,234,567"
```

That `(?<!\$)\d+` example is a good illustration of why lookbehind needs care: at index 1 the engine tries `100`, sees `$` behind it, fails, advances, and matches `00` from index 2. Negative lookbehind means "no match at *this* position", not "the whole match is not preceded by".

#### Building regexes from user input

```js
// ❌ Injection: the user's input is interpreted as a pattern.
const re = new RegExp(userInput);
// userInput = "a.*b"  → matches far more than intended
// userInput = "("     → SyntaxError, crashing the request
```

```js
// ✅ RegExp.escape — ES2025, Baseline 2025 (newly available since May 2025).
const re = new RegExp(RegExp.escape(userInput), "g");
```

[`RegExp.escape`](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/RegExp/escape) escapes every regex syntax character and punctuator, so the result is a safe literal pattern. Before it, everyone hand-rolled `str.replace(/[.*+?^${}()|[\]\\]/g, "\\$&")` — which is what you will find in older code and which misses several cases the standard method handles.

#### Catastrophic backtracking (ReDoS)

Nested quantifiers over overlapping character sets can make matching exponential in input length.

```js
// ❌ Exponential. On a 30-character non-matching string this hangs the thread.
const bad = /^(a+)+$/;
bad.test("a".repeat(30) + "!");     // effectively never returns

// ❌ A realistic version — this shape appears in real validators.
const badEmail = /^([a-zA-Z0-9_.-])+@(([a-zA-Z0-9-])+.)+([a-zA-Z0-9]{2,4})+$/;
```

```js
// ✅ Avoid nested quantifiers over the same character class.
const good = /^a+$/;

// ✅ Make alternatives mutually exclusive so there is nothing to backtrack into.
const goodQuoted = /^"[^"\\]*(?:\\.[^"\\]*)*"$/;
```

Because JavaScript is single-threaded, a ReDoS is a **complete denial of service** — the event loop stops, the server answers nothing. It is the one regex bug with security-incident consequences. The practical defences: avoid `(x+)+` and `(x*)*` shapes, prefer explicit character classes to `.*`, cap input length before matching, and do not build validators from patterns you found online without reading them.

> **Try It (carefully).** In a throwaway Node process:
> ```js
> const re = /^(a+)+$/;
> console.time("t"); re.test("a".repeat(24) + "!"); console.timeEnd("t");
> ```
> Expected: a few seconds. Increase `24` to `26` and it roughly quadruples. That doubling-per-character is the exponential curve; do not run it above ~28.

### Mastery — Unicode property escapes, the `v` flag, sticky matching

<a id="part-11-mastery"></a>

#### Unicode property escapes

With `u` or `v`, `\p{…}` matches by Unicode character property — the correct tool for anything beyond ASCII.

```js
/\p{L}/u.test("é");            // true  — any letter, any script
/\p{Lu}/u.test("A");           // true  — uppercase letter
/\p{Nd}/u.test("٣");           // true  — Arabic-Indic digit three
/\p{Script=Greek}/u.test("π"); // true
/\p{Emoji_Presentation}/u.test("👋");  // true
/\P{L}/u.test("1");            // true  — \P is the negation

// Strip everything that is not a letter, number, or space — Unicode-aware.
"Héllo, Wörld! 123".replace(/[^\p{L}\p{N}\s]/gu, "");   // "Héllo Wörld 123"
```

#### The `v` flag

`v` (unicodeSets, ES2024, [Baseline](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/RegExp/unicodeSets)) supersedes `u` and adds set operations plus properties that match *strings*, not just single code points:

```js
// Set difference and intersection inside a class.
/[\p{Script=Greek}--[πλ]]/v.test("π");       // false — Greek except π and λ
/[\p{Letter}&&\p{ASCII}]/v.test("é");        // false — ASCII letters only

// Properties of strings — multi-code-point graphemes.
/^\p{RGI_Emoji}$/v.test("👨‍👩‍👧‍👦");            // true — the whole family emoji
```

Under `u`, that family emoji is seven code points and no single-character class can match it. `v` is the first regex feature that handles real-world emoji correctly.

#### Sticky matching and `d`

```js
// y anchors the match at exactly lastIndex — nothing is skipped.
const re = /\d+/y;
re.lastIndex = 0;
re.exec("12ab34");     // ["12"] — lastIndex now 2
re.exec("12ab34");     // null   — index 2 is "a", and y will not scan forward
```

Sticky is how you write a tokenizer: each `exec` must match exactly where the previous one ended, so a gap is a syntax error rather than a silently skipped region.

```js
// A minimal tokenizer using sticky flags.
const TOKENS = [
  ["ws",     /\s+/y],
  ["number", /\d+/y],
  ["ident",  /[a-z]\w*/iy],
  ["op",     /[+\-*/=()]/y],
];

function* tokenize(src) {
  let pos = 0;
  outer: while (pos < src.length) {
    for (const [type, re] of TOKENS) {
      re.lastIndex = pos;
      const m = re.exec(src);
      if (m) {
        if (type !== "ws") yield { type, value: m[0], pos };
        pos = re.lastIndex;
        continue outer;
      }
    }
    throw new SyntaxError(`Unexpected character at ${pos}: ${src[pos]}`);
  }
}

[...tokenize("x = 1 + 22")].map(t => `${t.type}:${t.value}`);
// ["ident:x", "op:=", "number:1", "op:+", "number:22"]
```

The `d` flag adds precise offsets, which is what you need for editor tooling and error underlining:

```js
const m = /(?<year>\d{4})-(?<month>\d{2})/d.exec("on 2026-09");
m.indices[0];                 // [3, 10]  — whole match
m.indices.groups.year;        // [3, 7]
```

#### RegExp modifiers

ES2025 allows flags to be scoped to part of a pattern:

```js
/^(?i:hello) world$/.test("HELLO world");    // true
/^(?i:hello) world$/.test("hello WORLD");    // false — only the group is case-insensitive
```

This is newer than most of the features above; confirm support on MDN before relying on it in browser code.

### Part 11 quick reference

<a id="part-11-cheatsheet"></a>

| Method | Returns | Needs `g`? |
|---|---|---|
| `re.test(s)` | boolean | No — and `g` makes it stateful |
| `s.match(re)` | First match with groups, or all matches without them | Changes behaviour |
| `s.matchAll(re)` | Iterator of full match objects | **Required** |
| `re.exec(s)` | One match; advances `lastIndex` with `g` | Optional |
| `s.replace(re, x)` | New string | First only, unless `g` |
| `s.replaceAll(str\|re, x)` | New string | **Required** for regex |
| `s.split(re)` | Array | No |
| `s.search(re)` | Index or `-1` | No |

| Flag | Use it for |
|---|---|
| `g` | All matches — beware `lastIndex` state |
| `i` | Case-insensitive |
| `m` | `^`/`$` per line |
| `s` | `.` matches newline |
| `u` | Correct code-point handling; `\p{…}` |
| `v` | `u` plus set operations and string properties |
| `y` | Tokenizers; match exactly at `lastIndex` |
| `d` | Match offsets in `.indices` |

| Pitfall | Fix |
|---|---|
| Shared `/…/g` gives alternating results | Drop `g`, or create the regex per call |
| `match` with `g` loses capture groups | `matchAll` |
| User input interpolated into a pattern | `RegExp.escape(input)` |
| Nested quantifiers hang the thread | Avoid `(x+)+`; cap input length |
| `\w`/`\d` miss non-ASCII | `\p{L}` / `\p{Nd}` with `u` |
| Emoji split in half | `u` flag; `\p{RGI_Emoji}` with `v` |
| Positional groups renumber on edit | Named groups |

[↑ Back to top](#table-of-contents)

---

## 2. Part 12 — Dates, Times & Intl

<a id="part-12"></a>

### Beginner — `Date`, and the four traps you will hit immediately

<a id="part-12-beginner"></a>

```js
const now = new Date();
const epoch = new Date(0);                       // 1970-01-01T00:00:00Z
const fromMs = new Date(1757116800000);
const parsed = new Date("2026-09-06T12:00:00Z"); // ISO 8601 — the only safe string form
const parts = new Date(2026, 8, 6);              // 6 September 2026, LOCAL time
```

A `Date` is a single number: milliseconds since the Unix epoch, UTC. Everything else — the year, the month, the "timezone" — is interpretation applied on read.

#### Trap 1 — months are zero-indexed

```js
new Date(2026, 8, 6);      // 6 September 2026 — 8 means September
new Date(2026, 9, 6);      // 6 October 2026
```

Months are 0–11. Days of the month are 1–31. Days of the week are 0–6 (Sunday = 0). There is no rationale beyond a 1995 copy of Java's API; it is simply something to remember.

#### Trap 2 — string parsing depends on the format

```js
new Date("2026-09-06");            // parsed as UTC midnight
new Date("2026-09-06T00:00:00");   // parsed as LOCAL midnight
new Date("2026/09/06");            // parsed as LOCAL midnight
new Date("09/06/2026");            // US format, local — or NaN in some locales
new Date("6 Sept 2026");           // implementation-defined; may be Invalid Date
```

A date-only ISO string is UTC; a date-time ISO string without an offset is local. That inconsistency is specified, not a bug, and it is the direct cause of the "off by one day" class of bug:

```js
// ❌ In UTC-5, this prints the 5th.
new Date("2026-09-06").getDate();      // 5 — UTC midnight is 7pm the previous day locally

// ✅ Construct from parts for a local calendar date.
new Date(2026, 8, 6).getDate();        // 6

// ✅ Or read the UTC components you actually stored.
new Date("2026-09-06").getUTCDate();   // 6
```

**Never parse a non-ISO string with `new Date()`.** Format support beyond ISO 8601 is implementation-defined ([MDN: Date.parse](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Date/parse)), so the same string can yield different results in different engines.

#### Trap 3 — `Date` is mutable

```js
// ❌ Setters mutate in place. The "copy" is the same object.
function addDay(d) { d.setDate(d.getDate() + 1); return d; }
const original = new Date(2026, 8, 6);
const later = addDay(original);
original.getDate();       // 7 — the original changed

// ✅ Copy first.
function addDay(d) {
  const copy = new Date(d);
  copy.setDate(copy.getDate() + 1);
  return copy;
}
```

The setters do handle rollover correctly — `setDate(32)` in a 31-day month lands on the 1st of the next month — which is the one genuinely good thing about `Date` arithmetic.

#### Trap 4 — invalid dates fail silently

```js
const d = new Date("nonsense");
d;                          // Invalid Date
d.getTime();                // NaN
d === d;                    // true — it is a real object
isNaN(d);                   // true

// ✅ The check.
const isValid = (d) => d instanceof Date && !Number.isNaN(d.getTime());
```

> **Try It.** Run `const a = new Date("2026-09-06"), b = new Date("2026-09-06T00:00:00"); console.log(a.getDate(), b.getDate(), a.getTime() === b.getTime())`. Unless your machine is on UTC, expected: two different dates and `false`. One `T` changed the meaning of the string.

### Working Knowledge — reading, writing, and comparing

<a id="part-12-working"></a>

```js
const d = new Date(2026, 8, 6, 14, 30, 0);

// Local getters
d.getFullYear();   // 2026     ← never getYear(), which is deprecated and returns 126
d.getMonth();      // 8        ← zero-indexed
d.getDate();       // 6        ← day of month
d.getDay();        // 0        ← day of week, Sunday = 0
d.getHours();      // 14
d.getTime();       // ms since epoch

// UTC equivalents
d.getUTCFullYear(); d.getUTCMonth(); d.getUTCDate(); d.getUTCHours();

// Output
d.toISOString();       // "2026-09-06T13:30:00.000Z"  — always UTC, always this format
d.toJSON();            // same as toISOString(); used by JSON.stringify
d.toString();          // implementation- and locale-dependent. Do not parse it.
Date.now();            // current ms, without allocating a Date
```

**Store and transmit `toISOString()`.** It is unambiguous, sorts lexicographically in chronological order, and every language parses it.

#### Comparison and arithmetic

```js
const a = new Date(2026, 0, 1);
const b = new Date(2026, 6, 1);

a < b;                    // true  — relational operators use the numeric value
a.getTime() === b.getTime();   // the correct equality test
a === b;                  // ❌ false even for identical instants — object identity

const diffMs = b - a;                              // coerces to numbers
const diffDays = Math.round(diffMs / 86_400_000);  // 181
```

`a === b` being false for two equal dates is a routine bug. `<` and `>` work because relational operators coerce via `valueOf()`, but `===` compares references — see [Part 3](./javascript-foundations.md#part-3-mastery) on `Symbol.toPrimitive`.

That `86_400_000` figure assumes every day is 24 hours, which is false across daylight-saving transitions:

```js
// ❌ "Days between" via milliseconds is wrong across a DST boundary.
const days = (b - a) / 86_400_000;    // can be 180.958333…

// ✅ Compare calendar dates at local midnight.
const atMidnight = (d) => new Date(d.getFullYear(), d.getMonth(), d.getDate());
const days = Math.round((atMidnight(b) - atMidnight(a)) / 86_400_000);
```

#### Formatting with `Intl`

Do not build display strings by hand. [`Intl.DateTimeFormat`](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Intl/DateTimeFormat) handles locale, calendar, and time zone correctly.

```js
const d = new Date("2026-09-06T14:30:00Z");

new Intl.DateTimeFormat("en-GB", { dateStyle: "full", timeStyle: "short" }).format(d);
// "Sunday 6 September 2026 at 15:30"

new Intl.DateTimeFormat("en-US", { dateStyle: "medium" }).format(d);
// "Sep 6, 2026"

new Intl.DateTimeFormat("ja-JP", { dateStyle: "long" }).format(d);
// "2026年9月6日"

// Explicit time zone — the only reliable way to render "in Tokyo".
new Intl.DateTimeFormat("en-GB", {
  timeZone: "Asia/Tokyo", dateStyle: "short", timeStyle: "short",
}).format(d);
// "06/09/2026, 23:30"

d.toLocaleDateString("en-GB");     // shorthand; creates a formatter each call
```

**Reuse formatters.** Constructing one is expensive; the same instance is cheap to call repeatedly:

```js
// ❌ In a loop over 10,000 rows, this constructs 10,000 formatters.
rows.map(r => r.date.toLocaleDateString("en-GB"));

// ✅ Build once.
const fmt = new Intl.DateTimeFormat("en-GB");
rows.map(r => fmt.format(r.date));
```

The other `Intl` formatters are equally worth defaulting to:

```js
new Intl.NumberFormat("en-GB", { style: "currency", currency: "GBP" }).format(1234.5);
// "£1,234.50"

new Intl.NumberFormat("en", { notation: "compact" }).format(1_234_567);
// "1.2M"

new Intl.RelativeTimeFormat("en", { numeric: "auto" }).format(-1, "day");
// "yesterday"

new Intl.ListFormat("en", { type: "conjunction" }).format(["a", "b", "c"]);
// "a, b, and c"

new Intl.PluralRules("en").select(1);       // "one"
new Intl.PluralRules("en").select(2);       // "other"

// Segmenter — correct grapheme, word and sentence boundaries.
[...new Intl.Segmenter("en", { granularity: "grapheme" }).segment("👨‍👩‍👧‍👦a")].length;
// 2 — the family emoji is ONE grapheme
```

`Intl.Segmenter` is the correct answer to "count the characters" for user-facing purposes — better than `.length` (code units) and better than `[...str]` (code points), both of which split emoji families.

### Advanced — time zones, DST, and what `Date` cannot do

<a id="part-12-advanced"></a>

A `Date` has exactly one time zone concept: UTC internally, and the **host's** local zone for the non-UTC getters. It cannot represent a date in an arbitrary zone.

```js
// ❌ There is no setTimeZone. This is a formatted string, not a Date.
const tokyo = d.toLocaleString("en-US", { timeZone: "Asia/Tokyo" });
new Date(tokyo);        // reparsing a localised string — fragile and often wrong

// ✅ To READ components in a zone, use formatToParts.
function partsInZone(date, timeZone) {
  const fmt = new Intl.DateTimeFormat("en-CA", {
    timeZone, year: "numeric", month: "2-digit", day: "2-digit",
    hour: "2-digit", minute: "2-digit", second: "2-digit", hour12: false,
  });
  return Object.fromEntries(
    fmt.formatToParts(date).filter(p => p.type !== "literal")
       .map(p => [p.type, p.value]),
  );
}
partsInZone(new Date(), "Asia/Tokyo");
// { year: "2026", month: "09", day: "07", hour: "00", minute: "12", second: "34" }
```

`formatToParts` is the escape hatch whenever you need the pieces rather than a formatted string — it is also how you build a custom format without hardcoding locale assumptions.

#### DST discontinuities

```js
// In Europe/London, clocks go forward at 01:00 on 29 March 2026.
const before = new Date("2026-03-29T00:30:00Z");
const plus1h = new Date(before.getTime() + 3_600_000);
// The wall-clock time jumped from 00:30 to 02:30 local — one hour of ms,
// two hours of displayed time.
```

Two rules that follow:

1. **Adding 24 hours is not adding a day.** Two days a year, a "day" is 23 or 25 hours long.
2. **Some local times do not exist, and some occur twice.** 01:30 on a spring-forward date is not a real instant; 01:30 on an autumn-back date happens twice.

`Date`'s constructor silently picks an interpretation for these. There is no API to ask which. That limitation, more than the zero-indexed months, is why `Temporal` exists.

#### What to store

| Use case | Store |
|---|---|
| An instant (created-at, log timestamp) | UTC ISO string, or epoch ms |
| A future appointment | Local date-time **plus the IANA zone id** — not UTC |
| A birthday or a holiday | A plain date string, `"2026-09-06"`, with no time or zone |
| A duration | A number of seconds/ms, or an ISO 8601 duration |

The appointment row is the one people get wrong. If you convert "9am on 3 November in Berlin" to UTC and store that, and Germany later changes its DST rules, the meeting silently moves. Store the intent — wall-clock time plus zone — and resolve to an instant at read time.

> **Try It.** Find the next DST transition in your zone and run:
> ```js
> const d = new Date("2026-03-29T00:30:00Z");
> const later = new Date(d.getTime() + 3600e3);
> console.log(d.toLocaleString("en-GB", { timeZone: "Europe/London" }));
> console.log(later.toLocaleString("en-GB", { timeZone: "Europe/London" }));
> ```
> Expected: `29/03/2026, 00:30:00` then `29/03/2026, 02:30:00`. One hour added, two hours displayed.

### Mastery — `Temporal`

<a id="part-12-mastery"></a>

[`Temporal`](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Temporal) is the designed replacement for `Date`. It is on the **ES2027** track — commonly and wrongly labelled ES2026 in blog posts — and per the [TC39 finished-proposals list](https://github.com/tc39/proposals/blob/main/finished-proposals.md) it reached Stage 4 for that edition.

**Availability, stated plainly:** MDN currently shows Temporal as **limited availability — not Baseline**, because it does not yet work in some of the most widely used browsers. Chrome shipped it in version 144 (January 2026). **Do not ship it to browsers without a polyfill or a check**, and re-verify this status before acting on it — this is the single fastest-moving claim in this doc set.

The design fixes every trap above by splitting one overloaded type into several precise ones:

```js
Temporal.Instant          // a fixed point in time, UTC. No calendar.
Temporal.ZonedDateTime    // an instant + an IANA time zone + a calendar
Temporal.PlainDate        // 2026-09-06, no time, no zone  (a birthday)
Temporal.PlainTime        // 14:30, no date, no zone       (a daily alarm)
Temporal.PlainDateTime    // both, still no zone           (a wall-clock intent)
Temporal.PlainYearMonth   // 2026-09                       (a card expiry)
Temporal.PlainMonthDay    // 09-06                         (a recurring date)
Temporal.Duration         // a length of time
```

```js
// Months are ONE-indexed. Everything is immutable.
const date = Temporal.PlainDate.from({ year: 2026, month: 9, day: 6 });
date.month;                               // 9 — September, as written

const tomorrow = date.add({ days: 1 });   // returns a new object
date.toString();                          // "2026-09-06" — unchanged

// Calendar-correct arithmetic, including DST.
const meeting = Temporal.ZonedDateTime.from({
  timeZone: "Europe/London", year: 2026, month: 3, day: 28, hour: 9,
});
meeting.add({ days: 1 }).toString();
// 2026-03-29T09:00 local — still 9am, even though that day is 23 hours long

// Explicit, checked comparison and difference.
Temporal.PlainDate.compare(a, b);         // -1 | 0 | 1
a.until(b, { largestUnit: "day" });       // a Duration, not a millisecond count

// Parsing is strict — no implementation-defined guessing.
Temporal.PlainDate.from("2026-09-06");    // exact
Temporal.PlainDate.from("nonsense");      // throws RangeError, not Invalid Date
```

| Problem in `Date` | `Temporal`'s answer |
|---|---|
| Mutable | Every object is immutable; operations return new ones |
| Months 0-indexed | 1-indexed |
| One type for instants and calendar dates | Separate types per concept |
| No time-zone support beyond local/UTC | `ZonedDateTime` with IANA zones |
| DST arithmetic is wrong | Calendar-aware `add`/`subtract` |
| `Invalid Date` fails silently | Throws `RangeError` |
| Implementation-defined parsing | Strict ISO 8601 |
| Gregorian only | Pluggable calendars (`islamic`, `japanese`, …) |

Until it is Baseline, the practical migration path is: keep storing UTC ISO strings; use `Intl` for all formatting; use the [official `temporal-polyfill`](https://github.com/fullcalendar/temporal-polyfill) where you need real calendar arithmetic; and avoid adding new date-library dependencies for problems `Intl` already solves. Note also that Moment.js has been in maintenance mode for years and its own documentation recommends alternatives — if you meet it in a codebase, that is a migration signal, not a pattern to copy.

### Part 12 quick reference

<a id="part-12-cheatsheet"></a>

| Task | Do this | Not this |
|---|---|---|
| Current time | `Date.now()` | `new Date().getTime()` |
| Parse | `new Date(isoString)` | `new Date("09/06/2026")` |
| Build a local date | `new Date(y, m - 1, d)` | Parsing a formatted string |
| Store / transmit | `d.toISOString()` | `d.toString()` |
| Validity check | `!Number.isNaN(d.getTime())` | `d !== "Invalid Date"` |
| Compare | `a.getTime() === b.getTime()` | `a === b` |
| Copy | `new Date(d)` | Passing and mutating |
| Display | `Intl.DateTimeFormat` (reused) | Manual string building |
| Components in another zone | `formatToParts` with `timeZone` | Reparsing a localised string |
| Count user-visible characters | `Intl.Segmenter` | `.length` |
| Calendar arithmetic | `Temporal` (limited availability) or a polyfill | Millisecond maths across DST |

| `Date` gotcha | Remember |
|---|---|
| `getMonth()` | 0–11 |
| `getDay()` | Day of week, not month |
| `getYear()` | Deprecated — use `getFullYear()` |
| `"2026-09-06"` | Parsed as **UTC** |
| `"2026-09-06T00:00:00"` | Parsed as **local** |
| Setters | Mutate in place |
| `+1 day` | Not always `+86400000` ms |

[↑ Back to top](#table-of-contents)

---

**Next:** [Parts 13–14 — Metaprogramming & Modern JS](./javascript-metaprogramming-modern.md) · **Up:** [Mastery Guide overview](./javascript-mastery-guide.md)

*Written against ECMAScript 2026. Baseline data and Temporal availability verified September 6, 2026 — re-check Temporal's Baseline status before relying on it.*
