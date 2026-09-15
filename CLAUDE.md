# CLAUDE.md

Guidance for Claude Code and other AI assistants working in this repository.

## What this is

A React Native camera library: a thin TypeScript surface over a Swift camera implementation on
iOS and a CameraX one on Android. Almost nothing happens in JavaScript — `src/` declares props
and types, and the behaviour lives in `ios/ReactNativeCameraKit/` and
`android/src/main/java/com/rncamerakit/`.

The consequence is that most features cannot be added in one place. Adding a prop or an event
means touching TypeScript, both native platforms, and — because the library supports both React
Native architectures — two bridging paths per platform.

## Layout

| Path | Contents |
| :--- | :--- |
| `src/` | `CameraProps.ts` is the user-facing API; `types.ts` the shared enums; `Camera.tsx` lazily picks `Camera.ios.tsx` or `Camera.android.tsx`. |
| `src/specs/` | Codegen specs. `CameraNativeComponent.ts` is the Fabric component spec, `NativeCameraKitModule.ts` the TurboModule one. These drive the generated C++/Java, so they are the contract, not documentation. |
| `ios/ReactNativeCameraKit/` | `CameraView.swift` is the view; `RealCamera.swift` / `SimulatorCamera.swift` implement `CameraProtocol`; `CKCameraManager.mm` is the old-arch bridge and `CKCameraViewComponentView.mm` the new-arch one. |
| `android/src/main/` | `CKCamera.kt` is the view; `events/` holds one `Event` subclass per event. |
| `android/src/newarch/`, `android/src/oldarch/` | One `CKCameraManager.kt` each, selected by `IS_NEW_ARCHITECTURE_ENABLED` in `android/build.gradle`. **Both must be kept in step.** |
| `example/` | The demo app. CI builds it for both platforms, so it is effectively a compile test of the library. |

## Adding an event

Follow `onZoom` — it is the cleanest example to copy, and it touches every layer:

1. `src/specs/CameraNativeComponent.ts` — declare the payload type and
   `onX?: DirectEventHandler<...>`. Codegen derives the C++ struct from this.
2. `src/CameraProps.ts` — the public `OnXData` type and a documented prop.
3. **iOS**: an `@objc public var onX: RCTDirectEventBlock?` on `CameraView.swift`, fired from
   wherever the event originates; `RCT_EXPORT_VIEW_PROPERTY(onX, RCTDirectEventBlock)` in
   `CKCameraManager.mm` (old arch); and a `[_view setOnX:...]` block in
   `CKCameraViewComponentView.mm` that forwards to the generated emitter (new arch).
4. **Android**: a new `events/XEvent.kt` whose `EVENT_NAME` is `topX`, dispatched through
   `UIManagerHelper.getEventDispatcherForReactTag(...)`, and registered in
   `getExportedCustomDirectEventTypeConstants()` in **both** `newarch` and `oldarch`
   `CKCameraManager.kt`.
5. `README.md` has a props table; add a row. Demo it in `example/src/CameraExample.tsx`.

Verify the codegen contract rather than guessing at it:

```sh
npx react-native codegen --path . --platform ios --source library
grep -A5 "struct OnX" build/generated/ios/react/renderer/components/NativeCameraKitSpec/EventEmitters.h
```

The generated struct's field names are exactly what the `.mm` designated initializer must use.
`build/` is gitignored, so running codegen does not dirty the tree.

Two traps worth knowing. `CameraProps extends ViewProps`, so **prop names RN already defines are
taken** — `onFocus` and `onBlur` among them. And Fabric has no optional numeric props yet, so
numbers are declared with `WithDefault<..., -1>` in the spec and defaulted in `Camera.ios.tsx` /
`Camera.android.tsx`; any new numeric prop needs a line there too.

## Checks

```sh
yarn build     # tsc; the real typecheck
yarn lint      # see below
```

`yarn test` is declared but `jest` is not installed, and CI does not run it. `yarn lint` is
`eslint -c .eslintrc.js` with **no file arguments**, so it lints nothing and always passes —
don't read a green lint as meaning your TypeScript was checked. `yarn build` is the check that
actually bites.

CI (`.github/workflows/`) runs SwiftLint over changed `.swift` files, and builds both example
apps — `xcodebuild` for iOS and `./gradlew assembleDebug` for Android. Those builds are the real
gate, because a Kotlin or Swift mistake shows up nowhere else.

Building the Android example locally needs **Node ≥ 20.19** (the example is on React Native
0.81, while the root package installs fine on older Node), JDK 17, and an Android SDK:

```sh
export JAVA_HOME=$(brew --prefix openjdk@17)
export ANDROID_HOME=$(brew --prefix)/share/android-commandlinetools
echo "sdk.dir=$ANDROID_HOME" > example/android/local.properties
cd example && yarn && cd android && ./gradlew assembleDebug
```

`cd android && ./gradlew assembleDebug` on the library alone fails with "Plugin with id
'com.android.library' not found" — the library module has no buildscript of its own and is only
buildable through the example app.

SwiftLint needs SourceKit. With only the Command Line Tools installed it aborts at startup; point
it at the CLT copy rather than installing all of Xcode:

```sh
DYLD_FRAMEWORK_PATH=/Library/Developer/CommandLineTools/usr/lib swiftlint -- <files>
```

`CameraView.swift` already exceeds SwiftLint's `file_length` limit, so it reports a warning
before you touch it. Diff the violations before and after your change rather than reading the
raw count.

## Conventions

- Keep the two `CKCameraManager.kt` files symmetrical. A registration added to one and not the
  other produces an event that silently never reaches JS on that architecture.
- Normalize geometry to 0–1 of the preview when reporting positions to JS, as `onFaceDetected`
  and `onTapToFocus` do, so payloads don't depend on device pixel dimensions.
- Put geometry conversions in the view that owns the coordinate space, not in `CameraView.swift`
  — it is already long enough to trip the linter.
- Prop docs are TSDoc on `CameraProps.ts` with a short usage example, mirrored as a row in the
  README table. Platform-specific props say so in the first sentence.
