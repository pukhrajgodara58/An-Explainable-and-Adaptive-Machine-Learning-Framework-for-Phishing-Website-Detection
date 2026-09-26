# Upgrade notes

This project originally targeted Python 3.6.10 with library versions from
late 2021. It has been updated to run on current Python (tested on 3.12)
and current versions of every dependency. Summary of what changed and why.

## 1. `requirements.txt` — all packages bumped to current releases

| Package | Old | New |
|---|---|---|
| Flask | 2.0.2 | 3.1.3 |
| numpy | 1.21.4 | 2.4.4 |
| pandas | 1.3.4 | 3.0.2 |
| scikit-learn | 1.0.1 | 1.8.0 |
| beautifulsoup4 | 4.9.3 | 4.14.3 |
| requests | 2.25.1 | 2.33.1 |
| python-dateutil | 2.8.2 | 2.9.0 |
| gunicorn | 20.1.0 | 26.2.0 |
| `whois` (unmaintained, last released 2021) | 0.9.13 | replaced with **`python-whois`** 0.9.6 (actively maintained fork, same `import whois` API) |

`googlesearch_python` was already the right package name for `from googlesearch
import search`; it's now pinned to `googlesearch-python==1.3.0`.

## 2. `pickle/model.pkl` was regenerated

This was the actual breaking issue, not just outdated packages: the shipped
`model.pkl` was trained with scikit-learn 1.0.1, and loading it under a
current scikit-learn fails outright —

```
ModuleNotFoundError: No module named 'sklearn.ensemble._gb_losses'
```

scikit-learn reorganized its internal modules since then, and pickle stores
a reference to those internal paths, so an old model file simply cannot be
unpickled with a new scikit-learn install (there's no library version that
gets you both "new sklearn" and "old pickle" working together).

Fix: re-ran the exact training steps from `Phishing URL Detection.ipynb`
(same 80/20 split, `random_state=42`, `GradientBoostingClassifier(max_depth=4,
learning_rate=0.7)`) against `phishing.csv` with the current library
versions, and re-saved `pickle/model.pkl`. Accuracy on the held-out test set
came out to 97.4%, matching the notebook's original reported result — so
the model quality is unchanged, only its serialization format is current.

**If you ever change scikit-learn's version again, re-run this training
step and re-save the pickle** — this is a general rule for any sklearn
project, not specific to this codebase.

## 3. Real bugs fixed in `feature.py`

Several methods referenced undefined variables (`url`, `domain`, `self.soap`)
or never initialized loop counters. Because every method is wrapped in a bare
`try/except: return -1`, these typos never crashed the app — they just made
those specific features silently always return the same fallback value
instead of ever reading the actual page. Fixed:

- `__init__`: `BeautifulSoup(response.text, ...)` referenced an undefined
  `response` instead of `self.response`.
- `RequestURL`: `success`/`i` were used before being initialized.
- `AnchorURL`: compared against an undefined `url` instead of `self.url`.
- `Favicon`: compared against an undefined `domain` instead of `self.domain`.
- `InfoEmail`: referenced `self.soap`, which was never set anywhere (should
  be `self.response.text`).
- `StatsReport`: same undefined-`url` issue as `AnchorURL`.
- `PageRank`: built `prank_checker_response` but read from
  `rank_checker_response` (different variable name).

`DNSRecording` is intentionally **left** as a duplicate of `AgeofDomain`
(it doesn't actually query DNS) — that's how it was originally written, and
`phishing.csv`'s `DNSRecording` column was generated against that same
definition. "Correcting" it to a real DNS lookup would make live predictions
inconsistent with what the model was trained on.

## 4. Dead/fragile third-party services

- **Alexa (`data.alexa.com`)** — Amazon shut this down permanently on
  1 May 2022. `WebsiteTraffic` always returned -1 regardless of the URL.
  Replaced with a call to [Tranco](https://tranco-list.eu), the standard
  free academic replacement for Alexa-style rankings. If Tranco's endpoint
  ever changes, the method fails safe back to -1, same as before — nothing
  will crash, it just stops being informative again.
- **`checkpagerank.net`** — an unofficial third-party site, not a stable
  API. Fixed the variable-name bug so it at least tries correctly, but it
  may still be slow/unavailable; also fails safe to -1.
- **`googlesearch-python`** — the current version's `search()` returns a
  generator and takes `num_results` as a keyword argument (the old
  positional-`5` call no longer means the same thing). Also, Google
  aggressively rate-limits scraped queries, so expect frequent fallbacks
  here regardless.
- Added a real browser `User-Agent` header and an 8s timeout to outgoing
  `requests` calls — several sites otherwise silently block or hang on
  the default `python-requests` UA.

## 5. Testing done in this environment

- Fresh `venv`, installed strictly from the new `requirements.txt`.
- Loaded the regenerated `pickle/model.pkl` under scikit-learn 1.8.0 — works.
- Ran `FeatureExtraction` end-to-end against a live URL and fed the 30
  features into the model — produced a real prediction with no exceptions.
- Booted the Flask app with Flask 3.1.3 and confirmed `GET /` renders.

Network access in this sandbox is restricted to a small domain allowlist,
so `whois`, Google search, Tranco, and checkpagerank calls couldn't be
verified against arbitrary live domains from here — those will hit the
real internet once you run this on your own machine or deploy it. Please
smoke-test with a few real URLs after deploying.
