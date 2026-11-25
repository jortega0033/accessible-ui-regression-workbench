# Accessible UI Regression Workbench

## Product requirements

**Problem.** A maintainer can recheck a component five times in Chrome and still have trouble explaining which change moved a tab stop, changed a control name, or clipped a panel. The workbench records repeatable observations for an agreed fixture contract and leaves the visual and assistive-technology judgments with a reviewer.

**Intended users.** Frontend developers maintaining forms and shared UI components, and teammates confirming a regression during review. The proposed workflow targets small, versioned examples that a team owns and can bundle with the app.

**Primary journey.**

1. Select a fixture family and the reference and candidate versions. Read the planned viewports, keyboard steps and evidence checks before starting.
2. Run the reference fixture locally. Inspect the measured focus paths, named control match and overflow reading. Confirm it as the reference for that family and measurement profile.
3. Run the candidate. Review changed observations by check and viewport. Open one evidence row to see the exact path, match, or dimensions behind it.
4. Record the manual keyboard review separately. Reopen the run after a restart, and save a local report with the two run IDs and the measurement profile.

**First durable milestone.** Capture one bounded forward focus trace in a fresh Chrome context. Save the trace, close the workbench, restart it and reopen the same result. Start the comparison work only once that workflow is reproducible with the bundled fixture.

**Release scope.** Two fixture versions in one release family, both narrow and wide viewports, forward and backward focus traces, a named control assertion, one declared clipping container, baseline selection, comparison, manual review notes, and local JSON and HTML reports. The full release exercises the fixture catalog, browser runner, durable store, reviewer workflow and repeatable verification. Future work may add more fixture families and expressly supported browser profiles.

**Measurement scope.** This is a fixture regression tool. Each pass concerns one recorded observation. Visual focus visibility, practical keyboard operability and assistive-technology behavior still need human review. The app uses neither a WCAG grade nor a percentage that implies overall accessibility. WAI describes its initial checks as a starting point and outlines the role of human judgment when using evaluation tools. [WAI Easy Checks](https://www.w3.org/WAI/test-evaluate/preliminary/) · [WAI evaluation tools guidance](https://www.w3.org/WAI/test-evaluate/tools/selecting/)

**Input boundaries.** Serve original synthetic UI pages and assets included in the project. The workbench has no URL entry, custom JavaScript upload, site crawler, browser profile import, remote storage, online scanner, credentials, personal content or continuous monitoring. Adding a fixture means reviewing and bundling files and a manifest with the project source.

## Technical design

### Runtime and repository layout

Use Python 3 with a loopback HTTP server, a bounded Python Playwright worker and SQLite. Use bundled HTML, CSS and ES modules for the workbench. Use an existing Chrome or Chromium executable in the Linux coding workspace and an already installed Python Playwright package. The setup command preflights Python, SQLite, Chrome launch, Playwright import and the required fixture APIs, including ARIA snapshots. Record the exact versions and executable selection. Use a clean temporary browser profile with normal browser isolation. If any prerequisite or the launch check fails, show a setup message and keep historical runs readable. Implementation must demonstrate the exact locally installed combination before declaring it supported. [Playwright Python browser launch](https://playwright.dev/python/docs/api/class-browsertype#browser-type-launch) · [Playwright Python ARIA snapshots](https://playwright.dev/python/docs/api/class-locator#locator-aria-snapshot)

Organize application modules under app, UI files under web, immutable original fixtures under fixtures, and unit and browser scenarios under tests. Start with a single workbench command for setup verification and a separate command to open the local app. Keep the SQLite file and locally exported reports in an explicit application-owned data directory. The only standard runtime input files are bundled fixtures and the workbench database.

### Fixture and run contracts

| Record | Required information |
|---|---|
| Fixture manifest | Family, bundle version, relative route, manifest schema version, trusted asset digest, shared contract hash, ordered viewports, step sequences, known element keys, expected role and name, and declared layout target. |
| Measurement profile | Schema version, rule-set hash, fixture family and contract hash, ordered viewport definitions in CSS pixels, locale, browser name, full version and Playwright version. Baseline comparisons require equality across these fields. |
| Run | Run ID, immutable fixture revision and asset digest, measurement profile, run status, elapsed duration, and creation time. A run result includes all required check rows or a terminal capture error. |
| Evidence row | Viewport, stable check key, expected value, measured value, optional bounded diagnostic, and outcome. Sort by declared viewport and check order. |
| Baseline | One selected completed reference run per family and profile. Include a revision number for optimistic updates. |
| Manual review | Run ID, check key, status, short note, reviewer-chosen revision and recorded time. Manual reviews remain identifiable as human observations. |

The fixture catalog fixes a maximum of four families, four versions per family, two viewports per run, twelve keyboard actions per viewport, and eight check rows per viewport in the first release. Paths, expectation values and selectors come only from the catalog. Reject duplicate family/version pairs, duplicate stable check keys, missing targets and out-of-range viewports when validating the bundle. Use hash inputs with a defined ordering and canonical JSON encoding so identical fixture inputs produce identical digests.

Asset digests identify a fixture revision. A shared contract hash identifies the steps and expectations intended for comparison. A candidate can change its assets while keeping the same contract. Changing the rule thresholds, the contract, or the browser profile requires selecting a new matching reference.

### Capture behavior

Run each capture in a fresh browser context with a fixed desktop input profile, an explicit viewport and device scale factor 1. Use two contexts in sequence so focus and storage reset between viewports. The catalog provides a ready marker and a focus-start condition. Start measurement after the marker is present with the document body initially focused. Capture actual keyboard movement through Playwright's keyboard input and observe the focused element immediately after each action. No test may set focus between steps to make the path agree. Map the active element to a catalog key using its unique fixture attribute. Unknown or absent targets remain visible in the trace. [Playwright Python keyboard](https://playwright.dev/python/docs/api/class-keyboard)

- **Forward focus path (FWD).** Record four actual Tab results. Compare the ordered key sequence against the contract, including repeated and unrecognized keys.
- **Reverse focus path (REV).** From the final forward position record three actual Shift+Tab results. Compare the complete sequence.
- **Control name and role (NAME).** Select the declared target by its stable key. Query its documented role and exact accessible name through the Playwright role locator, then check whether the declared key is in the match set. Also store the target ARIA snapshot as bounded diagnostic text. Treat the locator match as the assertion. Display the snapshot as browser-runner evidence and avoid treating its YAML representation as a public application data format. Missing or duplicate target keys invalidate capture. A present target whose role or expected name differs produces a failed observation. [Playwright Python locators](https://playwright.dev/python/docs/locators) · [Locator ARIA snapshot](https://playwright.dev/python/docs/api/class-locator#locator-aria-snapshot)
- **Declared region overflow (OVER).** Read scrollWidth and clientWidth once the fixture has settled. Calculate excess as the nonnegative difference between them in CSS pixels. Pass when excess is at most one CSS pixel. This threshold absorbs a small rounding difference and applies to the declared region. Keep the two integer readings in the evidence row. The manual checklist asks the reviewer to inspect clipped content visually.

Record evidence rows for each viewport even if a check fails. Distinguish check failures from a runner error. A navigation timeout, missing fixture marker, duplicate stable key, unavailable browser or broken fixture contract ends capture with a typed error and publishes zero result rows. Automated recording checks do not depend on the desktop app's own focus or rendering.

### Persistence, comparison and recovery

Use tables for runs and evidence, baseline selections, and review revisions. Persist a queued run before launching a worker. Statuses are queued, running, complete, failed, cancelled and interrupted. Allow one active run per data directory. Acquire a startup lock so two server instances cannot claim the same queue. A startup with unfinished queued or running rows records them as interrupted. An interrupted run can be retried as a new run with a link to its predecessor.

Use a caller-generated request ID for start retry protection. The database enforces uniqueness per application instance. Retrying the same request with identical inputs returns its existing run. A reused request ID with different inputs returns a conflict. Use monotonic deadlines and terminate only the dedicated worker on timeout, after context cleanup or a fixed grace period. Cancellation moves an active run to cancelled and releases its slot, or returns the existing terminal state when completion wins. The final write transaction changes a running row to complete and inserts all evidence rows atomically. If cancelled before this transaction obtains the write lock, discard late output. Limit individual runner diagnostics to 8 KiB and each report to 256 KiB. Show typed timeouts and truncation warnings where relevant.

Compute differences by viewport and stable check key. Report new failure, fixed observation, unchanged failure and unchanged pass. For a row whose result stays the same, still show changed measurements in its detail panel. Complete comparable runs contain all expected keys exactly once. A nonmatching profile yields an incomparable result with the mismatched dimensions shown. Never infer success or a fix from missing data.

A baseline update requires a completed run and the current baseline revision. Stale updates return a conflict with the current selection. Review updates use their own revisions and reject stale changes without replacing the existing note. Keep captured measurements immutable when review statuses change. Use manual review states pending, reviewed without issue and issue observed. Require a note for an observed issue. The two manual prompts are whether focus remained clearly visible and whether the relevant operation could be completed via keyboard.

### Local routes and UI

- List the fixture catalog, runners' availability and exact measurement contract.
- Start a run with a fixture family/version and request ID. Poll run status and fetch evidence for a completed run. Cancel a pending or active run.
- Read or replace a baseline with its current revision. Compare a named candidate with a named baseline run, then review each changed row and its measurement.
- Save manual review by run ID and note revision. Export the selected comparison as deterministic JSON and standalone HTML. Embed textual or formatted evidence with safe escaping, no script assets and no network references. Export fields preserve manual review status separately from the capture outcome.

Bind the service to loopback, serve assets from its fixed bundle map and verify local same-origin mutation requests. Strip credentials and environment paths from reports. Prevent worker requests outside the precise local fixture origin, including navigations and embedded asset requests. Use a browser context with service workers blocked. The application exposes no public service or external integrations. Use parameterized SQLite statements and cap request fields and result sizes.

Use native buttons and form fields with persistent labels. Keep run status in a polite live region, include text alongside status colors and offer a keyboard-accessible shortcut link to captured evidence. Retain focus through polling and return focus to the initiating control after an evidence panel closes. Narrow rendering preserves readable evidence labels with scroll confined to raw evidence. The main comparison presents an ordered list and semantic details that can be inspected without relying on charts.

## Original verification fixtures

### Fixture family Contact Card

The two versions are fictional internal UI examples with only four native interactive elements at the top of a fresh page. The keys identify a Skip details anchor, a Title text input, a Save changes button and a Help anchor. All are visible, the body starts focused and there is no autofocus. Activating Save changes sets the local status to Draft saved, giving the manual reviewer a visible completion signal. The rail appears after the controls and clips horizontal overflow. Its content width is min(viewport width, 800) CSS pixels, with no padding, border or document scrollbar. The reference child fills its parent. The candidate child has a minimum width of 388 pixels. Save changes remains the visible button text in the candidate while its accessible name changes to Send. In the candidate, Help appears before Save in the document order.

| Viewport and version | FWD expected recording | REV expected recording | NAME expected result | OVER client/scroll/excess |
|---|---|---|---|---|
| 360 × 800, v1 reference | skip, title, save, help | save, title, skip | button named Save changes matches | 360 / 360 / 0 |
| 1024 × 768, v1 reference | skip, title, save, help | save, title, skip | button named Save changes matches | 800 / 800 / 0 |
| 360 × 800, v2 candidate | skip, title, help, save | help, title, skip | button named Save changes is absent on target, diagnostic name Send | 360 / 388 / 28 |
| 1024 × 768, v2 candidate | skip, title, help, save | help, title, skip | button named Save changes is absent on target, diagnostic name Send | 800 / 800 / 0 |

The checked contract is the v1 path, button role/name, and overflow limit for both runs. The v1 run has eight passing rows. The v2 comparison creates seven new failures and one unchanged pass. Only the wide overflow row remains unchanged. Each run starts with two pending manual reviews.

Fixture variations have independent expectations:

- Use the v2 evidence as the selected reference and the v1 evidence as candidate. This returns seven fixes and one unchanged pass with the same profile.
- Change the Chrome version in a saved test profile. The same run pair becomes incomparable. Show the browser-version difference and no regression count.
- Use a named-control fixture carrying a second unrelated button with the expected name. The v2 target still fails, because its stable key must match the assertion.
- Omit the ready marker in a dedicated runner-error fixture. The run ends as failed with a marker timeout and zero check rows. An earlier reference remains selectable.
- Simulate a process stop with one interrupted queued run using a disposable data directory. After restart, historical results remain complete and the stale queued item becomes interrupted without saving observation rows.

The hard-coded rows above form an independent oracle for the release tests. Do not derive expectations from the measurement implementation. Record observed Chrome/Playwright versions, fixture IDs, commands and browser results during implementation.

## Epic · Review accessibility regressions against an offline baseline

Build a useful browser workbench that can reproduce a component regression, separate automated evidence and human notes, and preserve the result after restart. Estimated implementation and verification effort is **15–23 hours**. Start with a usable focus trace, then complete keyboard and semantic/geometry capture in parallel against a fixed run contract. Integrate the comparison and browser review before the release demonstration.

### AR-01 · Define fixture contracts and capture the first trace

**Story.** As a maintainer, I want to see one real focus path before expanding the tool to more observations.

**Estimate.** 2–3 hours. **Dependencies.** None.

- **AR-01-A.** Setup preflight opens the bundled v1 fixture in the installed browser and reports the full runtime profile. An unavailable browser gives an actionable setup state with no background install.
- **AR-01-B.** Given the narrow fixture, four actual Tab inputs yield skip, title, save, help, and the stored observation remains identical after restarting the app.
- **AR-01-C.** Duplicate manifest keys, unbundled routes and unsupported schema versions are rejected before worker launch. Isolate manifest validation from browser startup for its unit test.

### AR-02 · Record bidirectional focus paths and bounded runner failures

**Story.** As a reviewer, I want to know the exact focus path in both directions and see when a run stops before measuring it.

**Estimate.** 3–4 hours. **Dependencies.** AR-01.

- **AR-02-A.** Compare four Tab steps and three Shift+Tab steps with the declared sequences at both viewports. The two v1 paths pass and both v2 paths fail per viewport.
- **AR-02-B.** A ready-marker timeout ends failed with zero observations. An external request attempted by the bundled test fixture is blocked and recorded, with no network transfer.
- **AR-02-C.** A cancelled running item leaves no observations and frees the run slot. A simulated restart records pending work as interrupted, and a retry creates a new run with its own identifier.

### AR-03 · Measure named control and region overflow evidence

**Story.** As a maintainer, I want a concrete role/name check and an exact clipping measurement to explain a changed fixture.

**Estimate.** 3–5 hours. **Dependencies.** AR-01. Use the agreed browser adapter and evidence contract independently of AR-02.

- **AR-03-A.** v1 matches the named Save changes button at its unique target key. v2 fails and stores the Send diagnostic. A different control with the expected name does not make the target pass.
- **AR-03-B.** The narrow v2 result reports 360 client pixels, 388 scroll pixels and 28 pixels of excess. Wide v2 and both v1 measurements report no excess. Test the one-pixel threshold with hand-authored numeric rows.
- **AR-03-C.** Missing and duplicated layout or semantic target keys end capture with a typed fixture error. Bounded diagnostics remain printable and treat captured content as text.

### AR-04 · Persist baselines and compare immutable observations

**Story.** As a reviewer, I want to reopen comparable results and tell changes from preexisting issues.

**Estimate.** 2–3 hours. **Dependencies.** AR-02 and AR-03.

- **AR-04-A.** A stored comparison between the two fixture versions reports seven new failures and one unchanged pass, and the inverse direction reports seven fixes and one unchanged pass.
- **AR-04-B.** A browser-version mismatch declares the pair incomparable. A stale baseline selection receives a conflict and leaves the selected reference unchanged.
- **AR-04-C.** Duplicate start requests with the same payload return the existing run. Reusing that request ID with a changed fixture is rejected. A completed run always has exactly eight ordered rows in the released family, including after restart.

### AR-05 · Review differences and export readable evidence

**Story.** As a teammate, I want to open the evidence, record what I checked by hand and share a local review report.

**Estimate.** 3–5 hours. **Dependencies.** AR-04.

- **AR-05-A.** A keyboard-only reviewer can select a fixture, start a capture, jump to results and read a changed focus path at either viewport. Polling preserves focus and evidence disclosure returns focus to its trigger when closed.
- **AR-05-B.** A reviewer can record the visible-focus and keyboard-operation checks independently. A stale note update leaves the saved note intact. Saving a review changes no recorded measurement.
- **AR-05-C.** Standalone HTML opens offline and displays two run IDs, both viewport sizes, seven new failures and one unchanged pass. Saving one manual review produces one reviewed and one pending prompt in the exported comparison. JSON retains the same observation keys, measurements and manual status fields. Fixture content is rendered as text in both the workbench and the report.

### AR-06 · Prove the complete offline review workflow

**Story.** As a maintainer, I want a reproducible end-to-end demonstration on a clean workspace.

**Estimate.** 2–3 hours. **Dependencies.** AR-05.

- **AR-06-A.** Starting with an empty data directory and external network unavailable, capture v1, select its baseline and capture v2. Confirm eight v1 observations and the expected v2 comparison and manual review counts.
- **AR-06-B.** Review a row, export both reports, restart and reopen the same comparison. Confirm that the report totals, profile and review state survive restart. Recheck the failed capture and incompatible-profile variations without changing the baseline.
- **AR-06-C.** Run the independent fixture assertions and a keyboard-only browser walkthrough. Save actual command output, runtime versions and any blocked checks with the release record.

## Scope decisions and follow-on work

The proposed default is one supported Chrome and Playwright combination, one stable family and two viewports. Keep automated scope and manual prompts visible in every review. Additional fixture families can model a dialog, dynamic form errors and a component with multiple focus groups after the reference example is stable. A broader protocol can later add other browsers without sharing baselines across incompatible profiles.
