# QA Challenge - Registration App

This repo contains my solution for the QA challenge: exploratory testing of the registration app, automated E2E tests with Maestro, and a CI/CD concept.

The app has three screens - account creation, verification and a success screen. I tested both the iOS simulator build and the Android APK.

## What's in here

```
maestro/
  android/        E2E flows for Android (happy path + unhappy paths)
  ios/            E2E flows for iOS (happy path + unhappy paths)
docs/
  exploratory-testing.md   approach, findings, conclusion
  ci-concept.md            how I would run this in a pipeline


```

## Why Maestro

The first question was which tools can test a native mobile app at all. Cypress and Playwright, which I'd normally reach for on web projects, are out here: they drive browsers, not native iOS or Android apps. (Playwright can talk to Chrome on an Android device, but that's still web content, not the app's native UI.)

That left Detox, Appium and Maestro. I went with Maestro for practical reasons:

- We only got the compiled builds, not the source code. Detox needs its own instrumented build, so it was out.
- Appium is the classic choice for native apps and would have worked fine with these builds (UiAutomator2 on Android, XCUITest on iOS). But the setup - Appium server, drivers, plus a test framework like WebdriverIO or Java/TestNG around it - felt like too much for an app with three screens.
- Maestro works directly with the `.apk` and `.app`, the flows are plain YAML that anyone on the team can read, and it waits for the UI on its own. That last point matters here because the verification step has a short random delay.
- The app already has `testID`s on almost every element, which Maestro can use on both platforms.

What I don't like about Maestro: there's not much room for real programming logic, and native date pickers are painful. I ran into that myself (more on it below). If the test suite grew a lot or had to be data-driven, I'd look at Appium or WebdriverIO again.

## Setup

You need:

- macOS with Xcode (for the iOS simulator) and/or Android Studio (for an emulator)
- Java 17 or newer
- Maestro CLI:

```bash
curl -fsSL "https://get.maestro.mobile.dev" | bash
```

Copy the builds into `apps/`, then install the app:

```bash
# Android (emulator must be running)
adb install -r apps/testapp-android.apk

# iOS (simulator must be booted)
unzip -o apps/testapp-ios-simulator.zip -d apps
xcrun simctl install booted apps/testapp.app
```

## Running the tests

```bash
# all Android flows (the known-bug flow is skipped, see below)
maestro test maestro/android/

# all iOS flows
maestro test maestro/ios/

# a single flow
maestro test maestro/android/00_happy_signup_flow.yaml

# the known bug on its own - this one is supposed to fail
maestro test maestro/android/12_BUG_only_whitespace_in_name.yaml
```

If an emulator and a simulator are running at the same time, pick one with `maestro --device <id> test ...` (`adb devices` or `xcrun simctl list devices booted` shows the ids).

For a JUnit report and screenshots of failures:

```bash
maestro test maestro/android/ --format junit --output report.xml --debug-output maestro-debug
```

## Which flows I automated and why

I started with the happy path, because if that breaks nobody can register at all. After that I picked the unhappy paths that protect the business rules and the places where users are most likely to make mistakes.

Android (`maestro/android/`):

| File | What it checks |
|---|---|
| `00_happy_signup_flow` | Full registration through to the welcome message |
| `01_unhappy_too_short_name` | One-letter names are rejected |
| `02_unhappy_invalid_email` | `faizan@example` is not accepted |
| `03_unhappy_passwords_do_not_match` | Different values in the two password fields |
| `04_unhappy_password_too_short` | Password under 8 characters |
| `05_unhappy_password_without_number` | Password with no digit |
| `06_unhappy_wrong_code` | Correct format but wrong code |
| `07_unhappy_code_empty` | Submitting without a code |
| `08_unhappy_short_code` | Code with fewer than 6 digits |
| `09_unhappy_empty_form` | Every field shows its "required" error |
| `10_unhappy_terms_not_accepted` | Can't finish without ticking the checkbox |
| `11_unhappy_younger_than_16` | Birth date under 16 is rejected (Android only, uses the calendar dialog) |
| `12_BUG_only_whitespace_in_name` | Documents a bug I found - see below |

iOS (`maestro/ios/`) has the same 13 flows with the same file names and test data, so results can be compared side by side. The differences are only where the platforms behave differently:

- The date picker is a wheel in a bottom sheet, confirmed with "Done". The under-16 flow swipes the year column instead of using Android's calendar dialog.
- The terms checkbox is found by its accessibility text (`"checkbox, unchecked"`), and the flow checks that it really switches to `"checkbox, checked"`.
- The iOS happy path also checks the summary on the verification screen and that "Start New Registration" brings back an empty form.

### The known-bug flow

`12_BUG_only_whitespace_in_name` describes how the app *should* behave: a name made only of spaces should be rejected. Right now the app accepts it and the success screen says "Welcome, !". So this flow fails on purpose. It's tagged `known-bug` and excluded from normal runs by the `config.yaml` in each platform folder. Once the bug is fixed it should pass, and then it can become a normal test.

Every flow starts with a fresh app (`clearState`), so they don't depend on each other and can run in any order.

### What I left out on purpose

- **All the validation combinations.** That would make the E2E suite slow. These rules are simple functions and belong in unit tests in the app repo.
- **Age edge cases** like turning 16 today or tomorrow, leap years and time zones. Hitting an exact date in the native picker is fiddly and flaky. One underage case per platform is covered in E2E.

## Things I ran into

A few practical notes, mostly so the next person doesn't lose time on the same issues:

- **Don't use `hideKeyboard` on Android.** On Android it presses Back, and if the keyboard is already closed, Back closes the app. I tap the screen title instead, which closes the keyboard safely.
- **The date picker is different on each platform.** Android opens a calendar dialog with an "OK" button. iOS shows a wheel in a bottom sheet with a "Done" button. For most flows I just confirm the default date (25 years ago), which avoids touching the picker at all.
- **The year wheel on iOS.** `scrollUntilVisible` scrolls the screen, not the wheel. To pick another year you have to swipe on the year column itself.
- **Verification takes a moment.** The code check is simulated with a random delay of up to about a second, so the flows wait for the success screen instead of asserting right away.
- **Some elements have no testID.** The terms checkbox and the verification code field had to be selected by their accessibility text (`"checkbox, unchecked"` and `"Field"`). That's also one of my findings - a screen reader user hears "Field" and doesn't know what to type in.

### iOS version differences

The iOS flows pass on the **iOS 17.4** simulator. On **iOS 17.5** and **iOS 26.1**, some locators stop matching:

<!-- Fill in which steps fail, e.g.: -->
<!-- - `tapOn: "checkbox, unchecked"` - element not found -->
<!-- - `id: "registration.dateOfBirth.done"` - element not found -->

The app is the same build in all three cases, so the difference comes from the iOS version, not from the app. As far as I can tell it's the accessibility tree that changes between versions. That's what Maestro (through XCUITest) reads, and labels and the way elements are grouped don't always look the same across iOS releases. I haven't pinned down the exact cause.

What I take from this:

- **Text and accessibility labels are the most fragile selectors.** Steps that use `testID`s are much less affected than steps that match on text like `"checkbox, unchecked"`. That's another reason to ask the developers to add `testID`s to the elements that don't have one yet (the terms checkbox in particular).
- **One simulator version isn't enough.** If I had only tested on 17.4, I would have reported a green suite that breaks on newer devices. This is exactly why the CI concept runs the nightly suite on more than one iOS version.
- For now the iOS suite should be run on **iOS 17.4**. Fixing the selectors for 17.5 and 26.x is the next step.

## Exploratory testing and CI

- Findings, how I tested and my conclusion: exploratory-testing.md
- CI/CD concept: ci-concept.md


## If I had more time

- Run the flows on a few more OS versions, not just one simulator and one emulator
- Move the repeated steps (filling the form, the date picker) into shared subflows. I kept every flow standalone for now so each one can be opened and run on its own in Maestro Studio, but it means a UI change has to be fixed in many files.
