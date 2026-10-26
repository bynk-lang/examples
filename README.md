# examples

The bynk-lang org's Bynk projects, and which Bynk releases each passes with. A
Bynk app, deployed to Cloudflare Workers.

The index reads two documents and merges them:

- **the [canary registry](https://github.com/bynk-lang/.github/blob/main/canary/repos.json):**
  every repository the org canary tests, with its kind and description;
- **the [compat board](https://github.com/bynk-lang/compat-board)'s results:**
  which Bynk release each repository last passed or failed with.

It shows every repository grouped by kind (examples first, then GitHub
Actions, then apps, then books), each with its newest result.

## Endpoints

| Route | What |
| --- | --- |
| `GET /` | The index as a page: a card per repository, with its description and a badge linked to its newest run. |
| `GET /examples.json` | `{"repos": [...], "refreshed_at_ms": …, "stale": …}`. Each entry has `repo`, `kind`, `description`, `status` (`pass`, `fail`, or `none` before the board has a result), `bynk_version` and `run_url`. The last two are empty when `status` is `none`. |

The index is refreshed at most every five minutes. If a refresh fails, because
GitHub or the board is unreachable or returns something that doesn't decode,
it keeps serving the last good copy with `stale: true` (and a note on the
page). Only before the first successful refresh does it answer
`503 Service Unavailable`.

## Layout

```text
src/examples/model.bynk  commons examples.model: the registry and board types, merging, version ordering
src/examples/page.bynk   commons examples.page: the HTML page
src/examples/index.bynk  context examples.index: the Sources capability over Fetch, the Index agent, the HTTP service
tests/examples/*.bynk    one suite per unit
```

- **`Sources`** is a one-operation capability (`download(url)`) provided over the platform's `Fetch`. It's the only thing that touches the network, so tests stub it.
- **The `Index` agent** keeps the newest snapshot in a `Cache` with `@ttl(5.minutes)`, and the last good one in a `Cell`.

Each test case checks one of these:
- the real registry and board decode;
- merging and ordering;
- one fetch per five minutes;
- a refresh after the TTL;
- the stale fallback;
- the unavailable state;
- the page.

The tests stub `Sources` and `Clock`, so they run offline.

## Run it locally

```sh
bynkc test .
bynk dev
```

Then open <http://localhost:8787/>. It fetches the real registry and board.

## Deploying

[`deploy.yml`](.github/workflows/deploy.yml) does an offline dry run on every
pull request, and deploys on every push to `main` once these repository
secrets exist (until then it warns and skips):

| Secret | Value |
| --- | --- |
| `CLOUDFLARE_API_TOKEN` | A Cloudflare API token with Workers deploy scope. |
| `CLOUDFLARE_ACCOUNT_ID` | The Cloudflare account ID. |

The Worker is called `examples-index`. When a deploy changes
`bynk.deploy.lock`, bynk-deploy opens a pull request that commits it, through the org's bump App (the workflow mints its token). Merge it: later deploys rely on what it records. Without the App, the workflow uploads the new lock as an artifact named `bynk.deploy.lock` to commit instead.

## Limits from Bynk today

- **No GitHub metadata.** `Fetch.send` can't set a `User-Agent`, which
  `api.github.com` requires (bynk-lang/bynk#1886). So the index reads only
  `raw.githubusercontent.com` and the board, and can't show stars or topics.
- **No optional JSON fields.** `Json.decode` reads an `Option` field only in
  Bynk's tagged form (bynk-lang/bynk#1887). That's why the registry's
  `description` is required, and why `examples.json` uses `status: "none"` and
  empty strings rather than an absent field.

## The canary

This repository is enrolled as `kind: example`:
[`bynk-canary.yml`](.github/workflows/bynk-canary.yml) runs bynk-ci and a
bynk-deploy dry run at each new Bynk release.

## License

Licensed under either of [MIT](LICENSE-MIT) or [Apache-2.0](LICENSE-APACHE) at
your option.
