# yt-dlp

Command-line media downloader, a youtube-dl fork. One pure-Python package `yt_dlp/`, no build step, 940 site
extractors, CPython and PyPy >=3.11. Tooling is `hatch` + `uv`, not pip. All dev config lives in `pyproject.toml`.

This is a personal fork. yt-dlp forbids AI for any contribution (`CONTRIBUTING.md:33-54`). Never push to `upstream`,
never open or comment on a PR or issue there. Work stays on this fork.

## Commands

```bash
hatch run setup                 # once: .venv via uv + pre-commit hook that runs `hatch fmt`
hatch test                      # core suite: 1292 offline tests, ~2 min. The 5840 network tests are deselected
hatch test test/test_utils.py   # one file (paths need a "/")
hatch test MagentaMusik         # one extractor's _TESTS, hits the network. With or without the IE suffix
hatch test -k "netrc"           # pytest -k
hatch fmt --check               # ruff check + autopep8 --diff, what CI runs. `hatch fmt` autofixes
hatch run yt-dlp -v URL         # run from source with -Werror -Xdev
make lazy-extractors            # after adding or renaming an extractor
make README.md                  # after changing help text in yt_dlp/options.py. Skips if README.md is newer, so
                                # `touch yt_dlp/options.py` first. Always emits a stray trailing-space hunk on the --date line: drop it
```

Tests run with `-Werror`: any DeprecationWarning fails. Two tests fail on macOS and are not bugs:
`test_get_netrc_login_info` (space in the hatch venv path) and `test_cookie_behavior[CurlFD]` (no `127.0.0.64` alias).

## Architecture

- `yt_dlp/__init__.py` `_real_main`: parse options (`options.py`), build `YoutubeDL`, call `ydl.download(urls)`.
- `yt_dlp/YoutubeDL.py` orchestrates: `extract_info` takes the first extractor whose `suitable(url)` matches,
  `process_video_result` picks formats, `dl` picks a downloader, `post_process` runs postprocessors.
- `yt_dlp/extractor/`: one module per site, base class `InfoExtractor` in `common.py`. Its docstring (`:105-440`) is
  the info-dict contract. `youtube/` is a subpackage.
- `yt_dlp/downloader/`, `postprocessor/`, `networking/` (request handlers: urllib, requests, curl_cffi, websockets),
  `jsinterp.py` (JS interpreter). `devscripts/` holds the generators and `run_tests.py`, which `hatch test` calls.
- `yt_dlp/utils/` and `compat/`: `_legacy.py` and `_deprecated.py` are API shims. Importing from them trips
  `-Werror` and ruff TID251. Use `utils/_utils.py` and `utils/traversal.py`.
- Extractor registry is 3 files. `extractor/_extractors.py` is the hand-edited import list (alphabetical, trailing
  comma on the last import of a group). `extractors.py` builds the lookup from it, YouTube first and `GenericIE` last,
  unless a `lazy_extractors.py` exists, which then wins. A stale one silently hides a new or renamed extractor.

## Conventions

- Extractor: `class FooIE(InfoExtractor)`, `_VALID_URL` with `(?P<id>...)`, `_real_extract` starts with
  `self._match_id`. At least one `_TESTS` entry, `'skip': 'reason'` or `'only_matching': True` if it can't run.
  Model file: `extractor/magentamusik.py`.
- Nested JSON goes through `traverse_obj(obj, ('a', 'b', {int_or_none}))`, never `.get` chains. Numbers through
  `int_or_none` / `float_or_none`, URLs through `url_or_none`, optional data with `fatal=False`.
- Single quotes, double only for docstrings. Trailing commas. 120 columns. `import re`, not `from re import`
  (ICN003). Imports: stdlib, `.common`, `..utils`. These import rules are enforced only under `yt_dlp/extractor/`.
- No `# noqa` unless a maintainer asked. No 3.12+ syntax, CI tests 3.11. New pytest markers must be registered in
  `pyproject.toml` (`strict_markers`). Network tests take `@is_download_test` from `test/helper.py`.
- Commits: `[area] Message`, `[ie/sitename]` for extractors, `[utils]`, `[rh:urllib]`, `[build]`, `[docs]`. Body ends
  with `Authored by: handle`. `devscripts/make_changelog.py` parses these.

## Guardrails

Generated files. Edit the source, then regenerate:

| Path | Source and command |
|---|---|
| `README.md` between `<!-- Auto generated -->` and `# CONFIGURATION` | help text in `yt_dlp/options.py`, `make README.md` |
| `supportedsites.md` | `make supportedsites` |
| `yt_dlp/extractor/lazy_extractors.py` (gitignored) | `make lazy-extractors` |
| `.github/ISSUE_TEMPLATE/*.yml` | `.github/ISSUE_TEMPLATE_tmpl/`, `make issuetemplates` |
| `yt_dlp/version.py`, `Changelog.md`, `CONTRIBUTORS`, `bundle/requirements/*.txt` | release scripts, leave alone |
| `yt_dlp/extractor/youtube/jsc/_builtin/vendor/*.js` | vendored from yt-dlp/ejs |

Wrong but plausible: bare `pytest` or `make test` (runs the 5840 network tests), `pip install -e .` (use hatch),
`.venv/bin/pytest` (pytest lives in the hatch-test env, not `.venv`), editing README option text directly.
