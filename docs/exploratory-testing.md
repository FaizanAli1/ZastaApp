# Exploratory Testing


## How I approached it

Before tapping through the app, I wanted to know what the rules actually are. The app is built with Expo / React Native, so the JavaScript ends up inside the build. I unpacked the iOS build and looked at the bundled code to see the validation rules, the error messages and which elements have test IDs. It also showed me that the app doesn't talk to any server and has no analytics - everything runs locally.

Knowing the rules made the exploratory part much more targeted. Instead of guessing ("is 7 characters too short for a password?") I could go straight to the edges: exactly 8 characters, a name with 1 character, someone who turns 16 tomorrow, and so on.

Then I tested on the device in short sessions, each with one focus:

1. **Happy path** on iOS and Android - does the normal registration work end to end?
2. **Input fields** - empty values, spaces, very short and very long input, umlauts, pasting text
3. **Date of birth** - the 16-year limit, cancelling the picker, different time zones
4. **Verification** - wrong code, short code, no code, terms not ticked, tapping submit twice
5. **Navigation** - going back, Android's back button, app in the background, starting a new registration
6. **Accessibility** - VoiceOver / TalkBack, larger font sizes, touch target sizes
7. **Layout** - small screens, keyboard covering fields

I ordered them by risk: first anything that could stop people from registering, then anything that lets bad data in, then platform differences, then accessibility.

## The rules, as I understood them

- Full name: at least 2 characters (and the field stops accepting input after 20)
- Email: must look like `something@domain.xx`
- Password: at least 8 characters, at least one number
- Confirm password: must match
- Date of birth: required, user must be 16 or older. The picker opens 25 years in the past and doesn't allow future dates
- Verification code: exactly 6 digits, the valid one is `123456`. The check takes a random 0.6 to 1.2 seconds
- Terms: must be accepted

## What I found

| # | Finding | Severity | Platform |
|---|---|---|---|
| 1 | A name of only spaces is accepted | Medium | |
| 2 | Names are cut off at 20 characters | Medium | |
| 3 | "Complete Registration" only reacts when you tap the text | Medium | |
| 4 | The terms can't be opened or read | Medium | |
| 5 | The checkmark icons may not show on Android | Medium | Android |
| 6 | The age check is a day off in some time zones | Low | |
| 7 | The password rules are weak | Low | |
| 8 | Some broken email addresses are accepted | Low | |
| 9 | Locators behave differently between iOS versions | Info | iOS |

**1. A name of only spaces is accepted.**
If you type three spaces as your name, the form accepts it. At the end the success screen says "Welcome, !". The app checks the length *before* removing the spaces and removes them *after*, so "   " counts as 3 characters. The same thing lets " A" through, which ends up as the one-letter name "A". The fix is simple: trim first, then check. I wrote a test for this that fails until it's fixed (`12_BUG_only_whitespace_in_name`).

**2. Names are cut off at 20 characters.**
"Maximilian Mustermann" is 21 characters, and the last letter just doesn't appear. There's no message. Double-barrelled names like "Anna-Lena Schmidt-Köhler" are very common in Germany and get cut off too, and the shortened name then shows on the next screens.

**3. "Complete Registration" only reacts when you tap the text.**
This is the most important button in the whole flow, and it only works if you hit the label in the middle. Tapping near the edges of the button does nothing - no feedback, no error. All other buttons in the app work normally. Users won't report this, they'll just think the app is stuck and leave.


**4. The terms can't be read.**
The checkbox says "I accept the terms and conditions", but there's no link and no way to see them. People are asked to accept something they can't read. That's more of a product / legal issue than a code bug, but it needs to be raised. The checkbox is also the only element without a test ID, which makes it harder to automate.

**5. The checkmark icons may not show on Android.**
The tick in the terms checkbox and the big checkmark on the success screen use an iOS-only icon system. On Android there's no fallback, so the icons may be empty. For the checkbox that would mean you can't see whether it's ticked.

**6. The age check is a day off in some time zones.**
The birth date is saved as text like "2010-11-01" and turned back into a date in a way that JavaScript treats as UTC midnight. In time zones behind UTC, like New York, that's still the day before. Someone who turns 16 tomorrow is accepted today. In Germany it works correctly, so the risk here is low - but it's an age limit, and those should be exact.

**7. The password rules are weak.**
"12345678" and "password1" are both accepted. Not a bug against the current rules, but worth a conversation with the product owner.

**8. Some broken email addresses are accepted.**
`max..muster@example.de`, `max@.example.de` and `max@example..de` all pass. Without a backend nothing breaks here, but in a real system these would be addresses you can never reach.

**9. Locators behave differently between iOS versions.**
Not a bug in the app itself, but worth knowing: my iOS tests pass on iOS 17.4, while some selectors stop working on 17.5 and 26.1 with the exact same build. Text-based selectors were affected the most. 

### Things that worked well

Not everything is a bug, and it's useful to know what's solid:

- Future birth dates can't be picked
- An empty form shows a clear error under every field and doesn't let you continue
- A wrong code shows a clear message and you can just try again without losing anything
- Tapping "Complete Registration" twice quickly doesn't submit twice
- Going back from the verification screen keeps everything you entered
- "Start New Registration" really starts with an empty form
- Opening the verification or success screen directly via a link sends you back to the start

## My conclusion

The app does what it's supposed to on the happy path, and the validation basics are there. The biggest risks I see:

1. **The main button (finding 3).** If the most important button ignores some taps, you lose registrations, and nobody will report it as a bug. It's cheap to fix, so I'd do it first.
2. **Input is handled inconsistently (findings 1, 2, 9).** The app checks one version of the input and saves another. In a real system that means bad data in the database and things like "Welcome, !" on screen.
3. **iOS and Android are not the same app (findings 6, 10).** The date picker works completely differently, icons may be missing on Android, and even iOS versions differ. Testing one platform or one OS version isn't enough.
4. **Accessibility and legal topics (findings 4, 5, 7).** An unlabelled input field, terms you can't read, and an age check that's a day off. These are the kind of issues that turn into complaints or compliance work later.
