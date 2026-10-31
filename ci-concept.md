# CI/CD Concept

This is how I would run the Maestro tests in a pipeline. I've used GitHub Actions as the example, but the same idea works with Bitrise or any other CI tool. 

## When does what run?

Running every test on every push would be slow and expensive, so I'd split it up:

**On every pull request**
Linting, the TypeScript check and unit tests first. If those pass, the Maestro smoke tests run - the happy path and a few key validation checks - on one Android emulator. The whole thing should take around 10 minutes including starting the emulator. That's quick enough that developers don't sit around waiting, and it catches the "registration is completely broken" kind of problem before anything is merged.

**When something is merged to main**
The full suite runs - happy path and all unhappy paths - on one iPhone simulator and one Android emulator in parallel.

**Every night**
The full suite again, but on more than one OS version per platform: the oldest one we support and the newest one. For this app that means iOS 16.4 (the minimum the build allows) and the current iOS 26, and the same idea on Android. I saw why this matters while writing the tests: my iOS flows passed on 17.4 but some selectors broke on 17.5 and 26.1. The known-bug flows also run at night, just for information.

**Before a release**
The nightly run on all versions, plus a manual exploratory session.

## How the app gets built and where the tests run

In the real app repo, the pipeline would build the app itself. For Android that's `npx expo prebuild` and Gradle. For iOS it's `xcodebuild` for the simulator, which doesn't need any signing certificates.

The important part: **build once, then reuse it.** The build job saves the `.apk` and `.app`, and all test jobs download that same file instead of building again. 

Where the tests run:

- **The simple option:** GitHub's own runners. Linux for Android , macOS for iOS (simulators come with Xcode).
- **If it needs to scale:** Maestro Cloud runs the flows in parallel on hosted emulators and simulators, so you're not waiting in line for macOS runners. For real devices you'd need a device farm like BrowserStack.

## Flaky tests

I'd rather prevent flaky tests than retry them, so:

- Use `testID`s wherever possible. Text selectors were the ones that broke between iOS versions.
- No fixed waiting times. The verification check has a random delay, so the tests wait for the success screen instead of sleeping for 2 seconds and hoping.
- Every test starts with a fresh app, so one failing test can't break the next one.
- Animations off on the emulator.

If a test fails anyway, it gets **one** retry. If it passes on the second try, it's marked as flaky in the report - not green. Otherwise retries just hide real problems.

If a test keeps being flaky (the native date picker is a good candidate), it gets moved to quarantine: it still runs at night but doesn't block pull requests anymore. Someone gets a ticket to fix it, with a date, so quarantine doesn't turn into a place where tests go to be forgotten.

## Seeing the results

- If a test fails, the pipeline fails and the PR can't be merged.
- Maestro writes a JUnit report (`--format junit`), which GitHub shows directly on the pull request.
- For failed runs, the screenshots, logs and the screen layout at the moment of failure (`--debug-output`) are uploaded so developers can look at them.
- Maestro doesn't record video by default. I'd turn on screen recording for failing flows - a short video is usually the fastest way to understand what went wrong.
- If the nightly run fails, a message goes to the team's Slack or Teams channel with a link to the results.

## Time, cost and what I'm trading off

**Time:**
- Pull request checks: about 8-12 minutes, mostly because starting an emulator takes a few minutes
- Full run after a merge: about 15-20 minutes per platform, both in parallel
- Nightly run on all versions: about 30-45 minutes

These are estimates. I'd replace them with real numbers after the first few runs.

**Cost:** macOS runners, which iOS needs, are about ten times more expensive than Linux on GitHub ($0.062 vs. $0.006 per minute). An Android smoke run costs a few cents, a 20-minute iOS run a bit over a dollar. With around 100 pull requests a month and nightly runs on both platforms, that comes to roughly $45 a month. For a public repo it's free.

**Trade-offs I'm making on purpose:**

1. **iOS doesn't run on every pull request.** It's too slow and expensive for that. Most of the code is shared JavaScript anyway, so Android catches most problems. iOS runs after every merge and every night, and on pull requests that change native dependencies or the app config. Worst case, an iOS-only bug sits on main for a day.
2. **Date logic is tested in unit tests, not E2E.** Age limits, leap years and time zones are much easier and more reliable to test in Jest with a fixed time zone. Finding 7 (the age check being a day off in some time zones) is a good example - a unit test catches it, an E2E test on a German simulator never would. E2E just checks that the picker works and that one underage date is rejected.
3. **Emulators and simulators instead of real devices in CI.** They're cheaper and more stable, but they miss things like performance problems or odd keyboards on some Android phones. That's what the manual check with real devices before a release is for.
