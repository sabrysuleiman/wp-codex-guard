# wp-codex-guard

![License: MIT](https://img.shields.io/badge/License-MIT-blue.svg)
![PHP](https://img.shields.io/badge/PHP-8.1%2B-purple)
![Status](https://img.shields.io/badge/status-alpha-orange)
![Codex](https://img.shields.io/badge/powered%20by-OpenAI%20Codex-412991)

> AI-powered semantic security analyzer for WordPress plugins & themes — built on OpenAI Codex.

`wp-codex-guard` scans PHP and JavaScript code for **real** vulnerabilities, not just pattern matches. It uses Codex agents to reason about data flow, sanitization context, and intent — then ships remediation patches and PHPUnit/Pest tests alongside every finding.

---

## The Problem

WordPress powers roughly **43% of the web**. Plugin and theme vulnerabilities remain the single largest source of CMS compromises year after year (Wordfence, Patchstack, WPScan reports). Existing open-source tooling for plugin authors is dominated by pattern-based linters:

| Tool | Approach | What it misses |
| --- | --- | --- |
| WordPress Coding Standards (WPCS) | Sniff rules | Semantic data-flow flaws |
| PHP_CodeSniffer | Token rules | Logic-level injection paths |
| Psalm / PHPStan (default rulesets) | Static types | Untrusted-source taint |
| WPScan | CVE database | Zero-days in unreported plugins |

None of them understand **what the code is trying to do**. They flag `$_GET` near `query()` and call it a day — generating noise on safe code and silence on novel flaws.

---

## How wp-codex-guard is different

Codex reads the function in context, traces where untrusted data enters, how (or whether) it is sanitized, and where it is consumed. It then reasons about the *category* of exploit and writes:

1. A **finding** with severity, location, CWE, and a proof-of-concept.
2. A **remediation patch** as a unified diff the maintainer can apply.
3. A **PHPUnit/Pest test** that reproduces the issue and proves the fix.

This is the gap pattern-based scanners cannot close — and the reason `wp-codex-guard` exists as an open, free toolchain for the WordPress community.

---

## Features

- 🛡️ **Multi-vector detection** — SQLi, XSS, CSRF, RCE, LFI/RFI, SSRF, XXE, insecure deserialization, broken auth, IDOR, open redirects.
- 🤖 **Codex semantic analysis** — context-aware, taint-flow reasoning instead of regex matching.
- 🔧 **Auto-generated patches** — unified diff per finding, ready to `git apply`.
- ✅ **Test scaffolding** — every critical finding ships with a reproducing PHPUnit/Pest test.
- 📦 **WordPress.org SVN scanner** — bulk-scan any plugin slug from the official repository.
- 🚀 **GitHub Action** — drop-in CI for plugin/theme repos; runs on every PR.
- 📊 **Open vulnerability dataset** — anonymized findings released under CC0 for research.

---

## Architecture

```
                       ┌──────────────────────────┐
   Source (PHP/JS) ───▶│  AST + Taint Graph       │
                       │  (nikic/php-parser)      │
                       └────────────┬─────────────┘
                                    │
                                    ▼
                       ┌──────────────────────────┐
                       │  Codex Agent (security)  │
                       │  - CWE classification    │
                       │  - Proof-of-concept      │
                       │  - Severity scoring       │
                       └────────────┬─────────────┘
                                    │
                       ┌────────────▼─────────────┐
                       │  Codex Agent (patcher)   │
                       │  - Unified diff          │
                       │  - PHPUnit/Pest test     │
                       └────────────┬─────────────┘
                                    │
                                    ▼
                       ┌──────────────────────────┐
                       │  Report (SARIF + Markdown)│
                       │  + diff.patch + tests/   │
                       └──────────────────────────┘
```

---

## Installation

### Requirements

- PHP **8.1+**
- Composer 2
- An OpenAI API key with access to Codex (or a ChatGPT Pro + Codex seat)

### Composer

```bash
composer require --dev sabrysuleiman/wp-codex-guard
```

### Standalone (PHAR)

```bash
curl -L https://github.com/sabrysuleiman/wp-codex-guard/releases/latest/download/wp-codex-guard.phar \
  -o /usr/local/bin/wp-codex-guard
chmod +x /usr/local/bin/wp-codex-guard
```

---

## Quick start

```bash
# Set your OpenAI key
export OPENAI_API_KEY="sk-..."

# Scan a single plugin file
wp-codex-guard scan ./wp-content/plugins/my-plugin/my-plugin.php

# Scan a whole plugin directory
wp-codex-guard scan ./wp-content/plugins/my-plugin/ --format=markdown -o report.md

# Scan a plugin directly from wordpress.org SVN
wp-codex-guard scan:svn --slug=contact-form-7 --format=sarif -o cf7.sarif
```

---

## Usage examples

### Scan a local plugin

```bash
wp-codex-guard scan ./wp-content/plugins/woocommerce \
  --severity=critical,high \
  --format=markdown \
  -o woocommerce-audit.md
```

Sample output:

```markdown
# wp-codex-guard Report — woocommerce

## Finding 1 — Critical · SQL Injection (CWE-89)

**File:** `includes/class-wc-query.php:412`
**Sink:** `$wpdb->get_var( $raw_sql )`
**Taint source:** `$_GET['orderby']`

### Why this is exploitable
The `orderby` parameter is mapped through an allowlist, but the
allowlist check happens after the value has already been concatenated
into `$raw_sql`. An attacker can inject a crafted `orderby` value
that bypasses the allowlist via type juggling.

### Proof of concept
GET /?orderby=1,(case+when+...+then+...+end)

### Patch
```diff
- $raw_sql = "SELECT ... ORDER BY " . $orderby;
+ $allowed = [ 'date', 'title', 'price' ];
+ $orderby = in_array( $orderby, $allowed, true ) ? $orderby : 'date';
+ $raw_sql = $wpdb->prepare( "SELECT ... ORDER BY %s", $orderby );
```

### Reproducing test
`tests/WC/Query/OrderByTest.php` (auto-generated)
```

### GitHub Action for CI

```yaml
# .github/workflows/codex-guard.yml
name: wp-codex-guard

on:
  pull_request:
    paths:
      - '**.php'
      - '**.js'

jobs:
  security:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: shivammathur/setup-php@v2
        with:
          php-version: '8.2'
          coverage: none
      - run: composer install --no-progress
      - name: Run wp-codex-guard
        uses: sabrysuleiman/wp-codex-guard-action@v1
        with:
          openai_api_key: ${{ secrets.OPENAI_API_KEY }}
          path: ./wp-content/plugins/${{ github.event.repository.name }}
          severity: critical,high
          fail_on: critical
```

### Bulk scan a plugin from wordpress.org

```bash
wp-codex-guard scan:svn \
  --slug=elementor \
  --format=sarif \
  --export-patches \
  -o elementor-$(date +%Y%m%d).sarif
```

---

## Configuration

`wp-codex-guard` reads an optional `.codex-guard.yml` at the project root:

```yaml
openai:
  model: codex-gpt
  temperature: 0.0
  max_tokens_per_request: 16000

scan:
  severity: [critical, high, medium]
  exclude:
    - vendor/
    - node_modules/
    - tests/
  detectors:
    - sqli
    - xss
    - rce
    - csrf
    - lfi
    - ssrf
    - xxe
    - deserialization
    - broken-auth
    - idor
    - open-redirect

output:
  format: [markdown, sarif, json]
  emit_patches: true
  emit_tests: true

report:
  upload_dataset: true   # anonymous finding -> CC0 dataset
```

---

## Detectors

| Detector | What it catches |
| --- | --- |
| `sqli` | Untrusted input reaching `$wpdb->query` / `get_var` / `get_results` without `prepare()` or after broken allowlist. |
| `xss` | Echoed `$_GET/$_POST/$_REQUEST` without escaping context (`esc_html`, `esc_attr`, `esc_js`, `wp_kses`). |
| `rce` | `eval`, `assert`, `create_function`, `preg_replace /e`, `system`/`exec`/`shell_exec` with tainted args. |
| `csrf` | State-changing handlers missing `check_ajax_referer` / nonce verification. |
| `lfi` / `rfi` | `include`/`require` with user-controlled paths. |
| `ssrf` | `wp_remote_get`/`wp_remote_post` with attacker-influenced URLs. |
| `xxe` | `libxml` parsing without `LIBXML_NOENT` hardening. |
| `deserialization` | `unserialize` / `maybe_unserialize` on untrusted data. |
| `broken-auth` | Capability checks missing or inverted on admin hooks. |
| `idor` | Object IDs from request used without ownership verification. |
| `open-redirect` | `wp_redirect` / `wp_safe_redirect` with unvalidated target. |

---

## Output formats

| Format | Use case |
| --- | --- |
| `markdown` | Human-readable audit report. |
| `sarif` | Upload to GitHub Code Scanning. |
| `json` | Pipeline integration / dashboards. |
| `patch` | A `diff.patch` file you can `git apply`. |
| `phpunit` / `pest` | A `tests/CodexGuard/` directory with reproducing tests. |

---

## Roadmap

### v0.1 — Alpha (current)
- [x] PHP AST + taint graph (nikic/php-parser)
- [x] Codex agent integration (findings only)
- [ ] Patch generation
- [ ] PHPUnit/Pest test scaffolding

### v0.2 — Beta
- [ ] JavaScript detector pack (DOM XSS, prototype pollution)
- [ ] Block theme + FSE support
- [ ] SARIF upload to GitHub
- [ ] `.codex-guard.yml` full schema

### v0.3 — Public dataset
- [ ] Anonymized CC0 vulnerability dataset
- [ ] WordPress.org top-1000 plugin quarterly reports
- [ ] Plugin author dashboard (read-only)

### v1.0 — Stable
- [ ] GH Action marketplace listing
- [ ] WordPress.org plugin page
- [ ] Stable detector contract + SemVer guarantees

---

## Why this matters for the WordPress ecosystem

- **Plugin authors** get a free, semantic security review on every PR — closing the gap that prevents indie devs from running commercial SAST.
- **The wordpress.org repository** gets a continuously-updated open vulnerability+patch dataset, useful for the broader security community.
- **Security researchers** get an open baseline to build on instead of re-deriving taint rules per project.

---

## Contributing

Contributions are welcome. Areas where help is especially needed:

- New detectors (Guzzle SSRF, REST API auth, block editor).
- Patch-generation prompts and evaluation harness.
- Reproducing-test templates for the PHPUnit/Pest scaffolder.
- Documentation and translation.

See [`CONTRIBUTING.md`](./CONTRIBUTING.md) for the workflow. All contributions must pass `wp-codex-guard scan` on themselves.

---

## License

MIT — see [`LICENSE`](./LICENSE).

The anonymized vulnerability dataset is released under **CC0 1.0 Universal**.

---

## Acknowledgements

- Built on top of [nikic/php-parser](https://github.com/nikic/PHP-Parser) for AST.
- Detector design informed by [OWASP Top 10](https://owasp.org/Top10/) and the [CWE catalog](https://cwe.mitre.org/).
- Standing on the shoulders of [WordPress Coding Standards](https://github.com/WordPress/WordPress-Coding-Standards) and [PHP_CodeSniffer](https://github.com/squizlabs/PHP_CodeSniffer) — we complement, not replace them.
- Powered by [OpenAI Codex](https://openai.com/codex/).
