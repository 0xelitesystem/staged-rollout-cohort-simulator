# Staged Rollout Cohort Simulator

Runs the electron-updater staged rollout formula against freshly generated install ids and shows that a staged rollout is a fixed cohort, not a rotating sample.

**Live demo:** https://0xelitesystem.github.io/staged-rollout-cohort-simulator/

This is a disproof, not a visualiser. A retry calculator draws the policy you configured. This page runs the shipped bucketing formula against ids it generates in your tab and measures a property of that formula you did not configure and probably did not intend.

The formula under test, as shipped in electron-updater's `AppUpdater.ts`:

    id  = UUID.v5(randomBytes(4096), UUID.OID)   // persisted once to .updaterId
    val = UUID.parse(id).readUInt32BE(12)
    pct = val / 0xffffffff
    return pct < stagingPercentage

The version string never enters that expression. Neither does the channel, the release date, or any per-release salt. The bucket is a pure function of an id the install persists once and keeps forever, so it is fixed for the life of the machine. Every consequence below follows from that one omission.

## Use

1. Open the page and press **Run simulation**. The defaults are 200,000 installs, gates at 10, 25 and 50 percent, and 10 consecutive releases at a 10 percent canary.
2. Read the three results:
   - **Nesting.** Each smaller cohort is a strict subset of the next. Raising the gate adds installs and drops zero. Widening a rollout is never a fresh sample, only a superset of the one you already shipped to.
   - **Repetition.** Ten consecutive releases select the identical set at a release-to-release overlap of exactly 1.0000. The share of your base that has ever received a staged build stays equal to one cohort no matter how many releases you ship.
   - **The counterfactual.** The same install ids, run again with the release id hashed in, the way percentage-rollout systems such as LaunchDarkly, Unleash and Statsig do it. Overlap falls to roughly 0.10 and the share ever canaried rises past 65 percent.
3. Run it a second time. The cohort sizes move, because the ids are drawn fresh from `crypto.getRandomValues` on every run. The structural results do not move. That contrast is how you tell a measurement from a hardcoded picture.
4. Use the **bucket inspector** to check the byte math on a real UUID. It computes the bucket two ways, Node's `readUInt32BE(12)` by explicit byte composition and the browser's `parseInt(hex.slice(24, 32), 16)`, and tells you whether they agree. The agreement sweep runs 1,000 random ids plus the all-ff and all-zero edge vectors.

Every field validates. Empty, non-numeric, out-of-range and duplicate inputs produce a visible error next to the control, never a silent no-op.

### The receipt

Both columns run over the same install ids in the same tab. This is the entire difference between them:

    - val = bucket(installId)                    // shipped: releaseId is not an input
    + val = bucket(hash(installId, releaseId))   // salted: one extra input
      return val / 0xffffffff < stagingPercentage / 100

### Three findings the page reports honestly

- **The nesting and repetition result** is the expensive one. One fixed slice of your install base absorbs every bad build you ever ship, and the rest of the base is a permanent control group that tests nothing.
- **The `0xffffffff` off-by-one** is real and unreachable, and the page says both halves. Dividing by `0xffffffff` instead of `0x100000000` means a maximal bucket yields exactly `1.0`, and `1.0 < 1.0` is false, so that install is excluded even from a 100 percent staged release. The probability is one in 2^32 per install. The page computes on load how many installs you would need for a 50 percent chance of a single affected user; it is close to three billion. Report it, do not lead with it.
- **Two more findings from the same update path.** The channel setter sets `allowDowngrade` to true unconditionally, so setting a channel for any reason silently arms downgrades. And pulling a bad staged release requires publishing a version *above* the broken one, because re-publishing the same version leaves everyone who already took the bad build stuck on it.

### Method

The shipped code generates 4096 random bytes, takes a UUID v5 of them, persists it, and reads exactly four bytes back out: bytes 12 to 15. UUID v5 places the version bits in byte 6 and the variant bits in byte 8, so bytes 12 to 15 carry no forced bits and the bucket is a uniform 32-bit draw. The simulator draws those 32 bits directly, four bytes per install instead of 4096. Same distribution, and it is what lets 200,000 installs times a dozen cohorts finish in well under a second. The inspector runs the full UUID path so the byte math is checkable rather than asserted.

The work runs in a worker built from a blob when the browser allows it, and falls back to time-sliced chunks on the main thread when it does not, for instance under a strict `file://` origin. Each run reports which path it took. Neither path is a blocking loop, and the tab stays responsive at the maximum of 2,000,000 installs.

## Why this exists

Staged rollout is sold as risk reduction, and the name invites you to picture a sample that moves. Reading the four-line formula tells you it does not move, but reading is not the same as seeing 19,900-odd machines selected ten times in a row on your own screen with the numbers regenerated in front of you. The gap between "I read the source" and "I measured it" is where this tool lives.

No account, no telemetry, no build step, no dependency tree. One HTML file with the CSS and JS inline. Nothing is fetched, so there is nothing to break when a CDN changes, and nothing to leak. Clone it, read all of it in one sitting, host it anywhere or nowhere. MIT, so lift the bucketing and the salted counterfactual straight into your own release tooling if that is more useful than the page.

## Privacy

Everything runs in your browser. Nothing leaves your machine. There is no network call of any kind: no analytics, no fonts, no CDN, no telemetry, no error reporting. The only thing written to storage is your light or dark theme preference in `localStorage`. Install ids are generated locally with `crypto.getRandomValues`, live only in memory for the duration of a run, and are discarded when you close the tab.

## Run locally

    git clone https://github.com/0xelitesystem/staged-rollout-cohort-simulator.git
    cd staged-rollout-cohort-simulator

Then open `index.html` in a browser. That is the whole procedure; it works straight off the filesystem.

If you would rather serve it over HTTP, which lets the simulation use the worker path instead of the main-thread fallback on stricter browsers:

    python -m http.server 8000

Then visit http://localhost:8000/.

## Build

There is no build. One file, `index.html`, with the CSS and JS inline. No bundler, no package manager, no dependencies, no lockfile, no CI step. Edit the file and reload the page.

## License

MIT. Copyright (c) 2026 0xelitesystem.

## Related

- https://github.com/0xelitesystem/electron-release-matrix-auditor
- https://github.com/0xelitesystem/yaml-type-coercion-auditor
- https://github.com/0xelitesystem/retry-backoff-calculator
