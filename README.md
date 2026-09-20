# SpotLight (SpeechCoach)

An Expo / React Native app for practising spoken communication. You record a
short video of yourself speaking, Google Gemini analyses the recording, and a
Vapi voice agent coaches you through the results in a live conversation.

The npm package and the native bundle identifier are both `SpeechCoach`
(`com.phantomzneurox.SpeechCoach`); "SpotLight" is the product name used in the
UI. The two names refer to the same app.

**Status: prototype.** Sign-in is mocked and there is no backend of the
project's own — see [What is and isn't real](#what-is-and-isnt-real).

---

## What it does

**Record.** `app/camera-practice.tsx` drives `expo-camera` with a 30-second cap,
front/back switching, and a rotating set of speaking prompts. Finished takes are
written to the device photo library via `expo-media-library`, and a thumbnail
plus duration metadata is kept in AsyncStorage by `utils/recordingUtils.ts`.

**Analyse.** `utils/geminiService.ts` uploads the video to the Gemini Files API
and asks for a structured critique. `utils/speechAnalysis.ts` defines four
coaching personas, each with its own model and temperature:

| Mode | Model | Focus |
|---|---|---|
| `general` | gemini-1.5-flash | accent, pacing, filler words, clarity, prosody |
| `interview` | gemini-1.5-pro | STAR structure, relevance, behavioural signals |
| `sales` | gemini-1.5-flash | discovery, objection handling, value articulation |
| `pitch` | gemini-1.5-pro | narrative, differentiation, traction, the ask |

Results render in `app/ai-analysis.tsx` as an overall score plus per-category
breakdowns (voice, word choice, sentence structure, conversational style,
non-verbal cues, overall impression).

**Reflect while you wait.** `app/guided-analysis.tsx` runs a six-step
self-review — intro, watch, reflect, focus, notes, complete — so you critique
your own take while Gemini is still processing. Your notes are stored alongside
the recording.

**Get coached.** `utils/vapiService.ts` opens a live voice session with a Vapi
assistant, seeded with your analysis, in `app/voice-coach.tsx`. Two drill
exercises (`s-pronunciation`, `quick-introduction`) each have their own
assistant and their own instruction screen at `app/exercise/[id].tsx`.

---

## Layout

```
app/
  index.tsx              redirect: authenticated -> /dashboard, else -> /login
  _layout.tsx            root stack + AuthProvider
  login.tsx signup.tsx   mock auth screens
  (tabs)/
    _layout.tsx          tab bar (see note below)
    dashboard.tsx        practice stats + recording history
    practice.tsx         exercise list
    progress.tsx         NOT currently reachable (tab commented out)
  camera-practice.tsx    30s recorder
  ai-analysis.tsx        Gemini results
  guided-analysis.tsx    guided self-review
  voice-coach.tsx        live Vapi session
  exercise/[id].tsx      per-exercise instructions
  profile.tsx            settings, data export/clear  (stack route, not a tab)
components/              VideoPlayerModal, CustomTabBar, NewRecordingModal, ...
utils/                   geminiService, vapiService, speechAnalysis,
                         recordingUtils, exerciseSessionStorage, apiService
contexts/AuthContext.tsx mock auth state
config/environment.ts    all API configuration
android/ ios/            prebuilt native projects (Expo prebuild output,
                         committed intentionally)
```

Only **Dashboard** and **Practice** are live tabs. The `progress` and `profile`
tab entries in `app/(tabs)/_layout.tsx` are commented out; `progress.tsx` is
kept because it is finished work waiting on that tab being re-enabled, and
`profile.tsx` is still reachable by navigation from the dashboard.

---

## What is and isn't real

Worth knowing before you read the code or demo the app:

- **Authentication is mocked.** `contexts/AuthContext.tsx` accepts any
  non-empty email/password, waits a second, and writes the literal string
  `mock-token` to AsyncStorage. There is no account system and no password
  check.
- **There is no backend.** `utils/apiService.ts` posts to
  `EXPO_PUBLIC_API_URL` (default `http://localhost:3000`), but no server for it
  lives in this repo. Analysis works because `geminiService.ts` calls Google
  directly from the device.
- **Storage is local and unencrypted.** Recordings go to the photo library;
  metadata and analyses go to AsyncStorage. Nothing syncs.
- **The Gemini key ships in the client.** Any `EXPO_PUBLIC_*` variable is
  inlined into the JS bundle and is readable by anyone with the app. For
  anything beyond a prototype, proxy Gemini through a server and keep the key
  there.

---

## Getting started

Requires Node 18+, and Xcode or Android Studio for native builds.

```bash
npm ci
cp .env.example .env    # then fill it in — see below
npm start               # or: npm run ios / npm run android / npm run web
```

`node_modules/` is not tracked; `npm ci` reproduces it exactly from
`package-lock.json`.

### Environment

Create `.env` in the project root. It is gitignored and must stay that way.

```bash
EXPO_PUBLIC_GEMINI_API_KEY=...     # required — aistudio.google.com
EXPO_PUBLIC_VAPI_PUBLIC_KEY=...    # required for voice coaching — dashboard.vapi.ai
EXPO_PUBLIC_API_URL=http://localhost:3000   # optional, unused without a backend
EXPO_PUBLIC_DEBUG_MODE=false
EXPO_PUBLIC_MOCK_ANALYSIS=false    # true to skip Gemini and use canned results
```

`config/environment.ts` reads these and `validateConfig()` warns on startup if
either key is missing. There is deliberately no fallback key in the source.

Longer setup notes live in [`GEMINI_SETUP.md`](GEMINI_SETUP.md) and
[`VAPI_SETUP.md`](VAPI_SETUP.md).

### Permissions

Camera, microphone and photo library, declared in `app.json` for both
platforms. The app asks on first use of the recorder.

### Builds

Native projects are committed, so `npm run ios` / `npm run android` build
locally. Cloud builds use EAS (`eas.json`, project
`b1545b6e-ccfb-4acb-9f12-92142fadbaf7`). Build output — `.aab`, `.apk`, `.ipa` —
is gitignored; don't commit it.

---

## Security

Three credentials were committed to this public repository and **remain in git
history**. Removing them from the current tree does not undo that. All three
must be treated as compromised and rotated:

| What | Where it was | Action |
|---|---|---|
| Android signing keystore | `credentials/android/keystore.jks` | Rotate; if it is the Play upload key, request an upload key reset in Play Console |
| EAS credentials file | `credentials.json` | Revoke and reissue its contents |
| Google Gemini API key | hardcoded fallback in `config/environment.ts` | Revoke and reissue in Google AI Studio |

`.gitignore` now blocks `.env`, `credentials.json`, `credentials/`, `*.jks`,
`*.keystore`, `*.p12`, `*.p8` and `*.mobileprovision`.

---

## Licence

Not currently specified. `package.json` is marked `private`, and no LICENSE
file is present.
