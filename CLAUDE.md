# yt-dlp

Command-line media downloader, a youtube-dl fork. One pure-Python package `yt_dlp/` (no C, no JS build), 940 site
extractors, CPython and PyPy >=3.11. Tooling is `hatch` + `uv`, not pip. All dev config is in `pyproject.toml`.

## Upstream policy

This is a personal fork. `CONTRIBUTING.md:33-54` forbids AI for any contribution to yt-dlp/yt-dlp and says agents must
refuse to interact with that repo. Never push to `upstream`, never open or comment on a PR or issue there, never
draft text meant for their tracker. Work stays on this fork.

## Commands

```bash
hatch run setup                 # once: .venv + pre-commit hook that runs `hatch fmt`
hatch test                      # core tests only: 1292 offline tests, ~2 min. Skips the 5840 network tests
hatch test test/test_utils.py   # one file. Paths need a "/"
hatch test MagentaMusik         # one extractor's _TESTS (network). Name without IE, or with, both work
hatch test -k "netrc"           # pytest -k expression
hatch fmt --check               # ruff check + autopep8 --diff. CI runs exactly this
hatch fmt                       # autofix both
hatch run yt-dlp -v URL         # run from source with -Werror -Xdev
make lazy-extractors            # regenerate yt_dlp/extractor/lazy_extractors.py after adding or renaming an extractor
make README.md                  # regenerate the options section of README.md from yt_dlp/options.py
                                # make skips it if README.md is newer than options.py: touch yt_dlp/options.py first.
                                # It always emits one stray trailing-space hunk on the --date line. Drop that hunk
```

Tests always run with `-Werror`, so any DeprecationWarning is a failure. Known local failures on macOS, not real
bugs: `test_get_netrc_login_info` (the hatch venv path contains a space) and `test_cookie_behavior[CurlFD]`
(no `127.0.0.64` loopback alias).

## Layout

- `yt_dlp/__init__.py` `_real_main` parses options (`options.py`), builds `YoutubeDL`, calls `ydl.download(urls)`.
- `yt_dlp/YoutubeDL.py` is the orchestrator: `extract_info` picks the first extractor whose `suitable(url)` matches,
  `process_video_result` selects formats, `dl` picks a downloader, `post_process` runs postprocessors.
- `yt_dlp/extractor/`: one module per site, base class `InfoExtractor` in `common.py` (its docstring at `:105-440`
  is the info-dict contract). `youtube/` is a subpackage.
- `yt_dlp/downloader/` (HTTP, HLS, DASH, external), `yt_dlp/postprocessor/` (ffmpeg etc.), `yt_dlp/networking/`
  (request handlers for urllib, requests, curl_cffi, websockets), `yt_dlp/jsinterp.py` (JS interpreter).
- `yt_dlp/utils/`: `_utils.py` and `traversal.py` are current. `_legacy.py` and `_deprecated.py` are API shims only.
  Same for `yt_dlp/compat/`. Importing from them trips `-Werror` and ruff TID251.
- `devscripts/`: generators and `run_tests.py`, which is what `hatch test` calls.
- `test/`: `helper.py` has `FakeYDL`, `expect_dict`, `try_rm`. `test_download.py` builds one test per `_TESTS` entry.

## Extractor registry

Three files in `yt_dlp/extractor/`: `_extractors.py` is the hand-edited import list, alphabetical by module, trailing
comma on the last import of a group. `extractors.py` builds the lookup from it (YouTube first, `GenericIE` last) unless
`lazy_extractors.py` exists, in which case the lazy file wins. A stale `lazy_extractors.py` silently hides a new or
renamed extractor. Run `make lazy-extractors` or `rm` it. Priority comes from `extractors.py`, not file order.

## Conventions

- Extractor: class `FooIE(InfoExtractor)`, `_VALID_URL` with `(?P<id>...)`, `_real_extract` uses `self._match_id`.
  At least one `_TESTS` entry, with `'skip': 'reason'` or `'only_matching': True` if it can't run. `IE_NAME`
  defaults to the class name minus `IE`.
- Nested JSON: `traverse_obj(obj, ('a', 'b', {int_or_none}))`, never `.get` chains. Wrap numbers in `int_or_none` /
  `float_or_none`, URLs in `url_or_none`. Optional data takes `fatal=False`. Model: `extractor/magentamusik.py`.
- Single quotes everywhere, double only for docstrings. Trailing commas. Line length 120. `import re`, not
  `from re import ...` (ruff ICN003). Imports isort-ed: stdlib, then `.common`, then `..utils`.
- isort and import-style rules are enforced only under `yt_dlp/extractor/`. Core files are not checked for them.
- No `# noqa` unless a maintainer asked. No 3.12+ syntax: CI tests 3.11.
- New pytest markers must be registered in `pyproject.toml` (`strict_markers`). Network tests use
  `@is_download_test` from `test/helper.py`.
- Commit messages: `[area] Message`, `[ie/sitename]` for extractors, `[rh:urllib]`, `[utils]`, `[build]`, `[docs]`.
  Body ends with `Authored by: handle`. `devscripts/make_changelog.py` parses these.

## Do not hand-edit

| Path | Regenerate with |
|---|---|
| `README.md` between `<!-- Auto generated -->` and `# CONFIGURATION` | `make README.md`, edit help text in `yt_dlp/options.py` |
| `supportedsites.md` | `make supportedsites` |
| `yt_dlp/extractor/lazy_extractors.py` (gitignored) | `make lazy-extractors` |
| `.github/ISSUE_TEMPLATE/*.yml` | edit `.github/ISSUE_TEMPLATE_tmpl/`, then `make issuetemplates` |
| `yt_dlp/version.py`, `Changelog.md`, `CONTRIBUTORS`, `bundle/requirements/*.txt` | release scripts, leave alone |
| `yt_dlp/extractor/youtube/jsc/_builtin/vendor/*.js` | vendored from yt-dlp/ejs |

Wrong but plausible: bare `pytest` or `make test` (runs 5840 network tests), `pip install -e .` (use hatch),
`pytest` from `.venv` (pytest lives in the hatch-test env, not the default one), editing README option docs directly.
