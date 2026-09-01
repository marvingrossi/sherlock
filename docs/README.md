<p align="center">
  <br>
  <a href="https://sherlock-project.github.io/" target="_blank"><img src="images/sherlock-logo.png" alt="sherlock"/></a>
  <br>
  <span>Hunt down social media accounts by username across <a href="https://sherlockproject.xyz/sites">400+ social networks</a></span>
  <br>
</p>

<p align="center">
  <a href="https://sherlockproject.xyz/installation">Installation</a>
  &nbsp;&nbsp;&nbsp;•&nbsp;&nbsp;&nbsp;
  <a href="https://sherlockproject.xyz/usage">Usage</a>
  &nbsp;&nbsp;&nbsp;•&nbsp;&nbsp;&nbsp;
  <a href="https://sherlockproject.xyz/contribute">Contributing</a>
</p>

<p align="center">
<img width="70%" height="70%" src="images/demo.png" alt="demo"/>
</p>


## Installation

<p align="center">
  <a href="https://usersearch.com/?utm_source=github&utm_medium=referral&utm_campaign=sherlock&utm_content=banner_install" target="_blank"><img src="images/usersearch.png" alt="User Search"/></a>
  <a href="https://www.osint.industries/" target="_blank"><img src="images/osint-industries.jpg" alt="OSINT Industries"/></a>
</p>



> [!WARNING]  
> Packages for ParrotOS and Ubuntu 24.04, maintained by a third party, appear to be __broken__.  
> Users of these systems should defer to [`uv`](https://docs.astral.sh/uv/)/`pipx`/`pip` or Docker.

| Method | Notes |
| - | - |
| `pipx install sherlock-project` | `pip` or [`uv`](https://docs.astral.sh/uv/) may be used in place of `pipx` |
| `docker run -it --rm sherlock/sherlock` |
| `dnf install sherlock-project` | |

Community-maintained packages are available for Debian (>= 13), Ubuntu (>= 22.10), Homebrew, Kali, and BlackArch. These packages are not directly supported or maintained by the Sherlock Project.

See all alternative installation methods [here](https://sherlockproject.xyz/installation).

## General usage

To search for only one user:
```bash
sherlock user123
```

To search for more than one user:
```bash
sherlock user1 user2 user3
```

Accounts found will be stored in an individual text file with the corresponding username (e.g ```user123.txt```).

```console
$ sherlock --help
usage: sherlock [-h] [--version] [--verbose] [--folderoutput FOLDEROUTPUT] [--output OUTPUT] [--csv] [--xlsx] [--site SITE_NAME] [--proxy PROXY_URL] [--dump-response]
                [--json JSON_FILE] [--timeout TIMEOUT] [--print-all] [--print-found] [--no-color] [--browse] [--local] [--nsfw] [--txt] [--ignore-exclusions]
                USERNAMES [USERNAMES ...]

Sherlock: Find Usernames Across Social Networks (Version 0.16.0)

positional arguments:
  USERNAMES             One or more usernames to check with social networks. Check similar usernames using {?} (replace to '_', '-', '.').

options:
  -h, --help            show this help message and exit
  --version             Display version information and dependencies.
  --verbose, -v, -d, --debug
                        Display extra debugging information and metrics.
  --folderoutput FOLDEROUTPUT, -fo FOLDEROUTPUT
                        If using multiple usernames, the output of the results will be saved to this folder.
  --output OUTPUT, -o OUTPUT
                        If using single username, the output of the result will be saved to this file.
  --csv                 Create Comma-Separated Values (CSV) File.
  --xlsx                Create the standard file for the modern Microsoft Excel spreadsheet (xlsx).
  --site SITE_NAME      Limit analysis to just the listed sites. Add multiple options to specify more than one site.
  --proxy PROXY_URL, -p PROXY_URL
                        Make requests over a proxy. e.g. socks5://127.0.0.1:1080
  --dump-response       Dump the HTTP response to stdout for targeted debugging.
  --json JSON_FILE, -j JSON_FILE
                        Load data from a JSON file or an online, valid, JSON file. Upstream PR numbers also accepted.
  --timeout TIMEOUT     Time (in seconds) to wait for response to requests (Default: 60)
  --print-all           Output sites where the username was not found.
  --print-found         Output sites where the username was found (also if exported as file).
  --no-color            Don't color terminal output
  --browse, -b          Browse to all results on default browser.
  --local, -l           Force the use of the local data.json file.
  --nsfw                Include checking of NSFW sites from default list.
  --txt                 Enable creation of a txt file
  --ignore-exclusions   Ignore upstream exclusions (may return more false positives)
```

## Adding a new site

Sites are data-driven: adding one only requires a single entry in
[`sherlock_project/resources/data.json`](../sherlock_project/resources/data.json), keyed by the
site's display name and kept in alphabetical order. No code changes are needed.

Every entry requires four fields:

| Field | Meaning |
| - | - |
| `url` | Public profile URL. The `{}` placeholder is replaced with the username being searched. |
| `urlMain` | Site homepage, shown in results. |
| `errorType` | How Sherlock decides that a username is *not* claimed (see below). |
| `username_claimed` | A real, stable account on the site, used by the validation suite. |

### Picking an `errorType`

Probe the live site with `curl` for both a known-existing username and a random non-existent one,
then compare the responses:

| `errorType` | Use when | Extra field |
| - | - | - |
| `status_code` | The existing profile returns 2xx and the missing one returns a non-2xx (e.g. 404). Cheapest option (uses `HEAD`), prefer it when reliable. | `errorCode` (optional) |
| `message` | Both return 200, but the missing profile's body contains a distinctive error string. | `errorMsg` (string or list of strings) |
| `response_url` | Existence is signalled by a redirect. | `errorUrl` |

Commonly used optional fields:

- `urlProbe` — alternate URL to request, typically a JSON API endpoint, while `url` stays the human-facing profile page.
- `regexCheck` — regex describing the characters the site allows in a username, so impossible usernames are skipped.
- `request_method` / `request_payload` / `headers` — for sites needing `POST` or special headers.
- `isNSFW` / `tags` — mark adult (`isNSFW: true`, `tags: "adult"`) or gaming sites.

```json
  "Example": {
    "errorType": "status_code",
    "regexCheck": "^[A-Za-z0-9_]{3,20}$",
    "url": "https://example.com/user/{}",
    "urlMain": "https://example.com/",
    "username_claimed": "blue"
  },
```

### Validating the entry

```bash
# Schema conformance
poetry run pytest tests/test_manifest.py::test_validate_manifest_against_local_schema

# Live check of just your site: username_claimed must be detected as CLAIMED (false negative
# check) and random usernames must come back AVAILABLE (false positive check)
poetry run pytest -q -rA -m validate_targets -n 1 --chunked-sites "Example"
```

Iterate on `url`, `errorType`, `errorMsg`/`errorUrl` and `regexCheck` until both pass. If a site
cannot be detected reliably — it always returns 200 with no distinguishing content, or requires
authentication — it is not a good candidate; see [removed-sites.md](removed-sites.md) for sites
that were dropped for this reason.

## Credits

Thank you to everyone who has contributed to Sherlock! ❤️

<a href="https://github.com/sherlock-project/sherlock/graphs/contributors">
  <img src="https://contrib.rocks/image?&columns=25&max=10000&&repo=sherlock-project/sherlock" alt="contributors"/>
</a>

## Star History

<picture>
  <source media="(prefers-color-scheme: dark)" srcset="https://api.star-history.com/svg?repos=sherlock-project/sherlock&type=Date&theme=dark" />
  <source media="(prefers-color-scheme: light)" srcset="https://api.star-history.com/svg?repos=sherlock-project/sherlock&type=Date" />
  <img alt="Sherlock Project Star History Chart" src="https://api.star-history.com/svg?repos=sherlock-project/sherlock&type=Date" />
</picture>

## License

MIT © Sherlock Project<br/>
Creator - [Siddharth Dushantha](https://github.com/sdushantha)

<!-- Reference Links -->

[ext_pypi]: https://pypi.org/project/sherlock-project/
[ext_brew]: https://formulae.brew.sh/formula/sherlock
