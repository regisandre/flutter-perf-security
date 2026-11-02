---
name: flutter-perf-security
description: Measure, optimise and harden a Flutter app screen by screen on a real Android phone. Use when asked to make a Flutter app smoother, to find jank or raster cost, to A/B a change with real numbers, or to run a quick security pass on an Android build. Covers the measuring tools, a repeatable A/B harness, the fixes that paid off, and the pitfalls that wasted time.
---

# Flutter performance and security campaign

Rule of the campaign: **measure before, measure after, keep a change only if it beats the noise.**
A change that cannot be separated from the run-to-run spread is reverted, however plausible it was.

## 1. Set up the measurement

- Use a real device and a **profile** build (`flutter build apk --profile -t lib/main.dart`). Debug is
  JIT-slow: use it only for widget rebuild counts and the widget tree. Release hides the tooling.
- Keep the phone awake and plugged in (`adb shell svc power stayon true`), same brightness, same network.
- Read the VM service URI from logcat (`The Dart VM service is listening on http://127.0.0.1:<port>/<token>/`)
  and `adb forward tcp:<port> tcp:<port>`. Never overwrite the forward `flutter run` made for itself.
- Guard every `adb input tap` on the focused window (`dumpsys window | grep mCurrentFocus` must name the
  app), or a stray tap lands in another app.

## 2. Two tools, two jobs

| Tool | Good for | Not good for |
|---|---|---|
| Frame timing through the VM service (swipes driven by `adb`) | finding which screens are bad, janky-frame percentage | proving a gain under ~5 points: the same screen swings 0 to 8 % between runs |
| `integration_test` + `traceAction` + `TimelineSummary` (`flutter drive --profile`) | A/B of a change: average and p90/p99 build and raster time, frames over budget | discovering problems |

### The A/B harness (what it needs to work)

```dart
final binding = IntegrationTestWidgetsFlutterBinding.ensureInitialized();
testWidgets('screen scroll', (tester) async {
  unawaited(app.main());
  await pumpUntil(tester, find.byType(Scrollable));      // poll the PLAIN finder
  for (var run = 0; run < 5; run++) {
    await binding.traceAction(() async {
      for (var i = 0; i < 6; i++) {
        await tester.fling(feed, const Offset(0, -500), 2200);
        await tester.pump(const Duration(milliseconds: 600));
      }
    }, reportKey: 'screen_$run');                          // one summary per pass
  }
});
```

Driver (`test_driver/perf_driver.dart`) calls `integrationDriver(responseDataCallback: ...)` and writes
`TimelineSummary.summarize(...).writeTimelineToFile('<key>', destinationDirectory: 'build/perf', pretty: true)`.

Run: `flutter drive --profile --no-dds --keep-app-running -d <serial> --driver=test_driver/perf_driver.dart --target=integration_test/perf/<test>.dart`

Compare the **mean of 5 passes per variant** and their standard deviation. Keep the change when the gap
is several standard deviations (a 2.5 ms raster gain against a 0.3 ms spread is real; a 0.4 ms build
difference against a 0.1 ms spread on code you did not touch is variance).

### Pitfalls that cost hours

- `finder.first` throws `Bad state: No element` while nothing matches, so never hand `.first` to a
  polling helper: poll the plain finder, take `.first` afterwards.
- A profile build prints only `Instance of 'FlutterErrorDetails'` for a framework error. Install a
  `FlutterError.onError` that prints `details.exceptionAsString()`.
- An app that swaps `debugPrint` after bootstrap hides your markers: log from the test with `print`.
- A remote image answering 404 is reported as a framework error and fails the whole test, and a failed
  test returns no timeline data at all. Log `HttpException` and keep it out of the verdict.
- `flutter drive` **uninstalls the app when it ends** (the login is lost) unless `--keep-app-running`.
- A `flutter drive` build overwrites `app-profile.apk` with the test entrypoint: rebuild with
  `-t lib/main.dart` before using any VM-service tool again.
- Do not edit the source tree while the build of the variant you are measuring is compiling.
- A run that loses the VM service loses every summary it had not written yet. Keep batches short (a
  few screens, 3 passes each) and filter screens with a `--dart-define=ROUTES=a,b` list.
- On a phone with little free RAM the system kills the app mid-run and it looks like a crash. Check
  `adb shell dumpsys meminfo | grep "Free RAM"` before each batch, stop heavy background apps first, and
  read `dumpsys activity exit-info <package>` (reason `LOW_MEMORY`) before blaming the screen. Never
  `am kill-all` in the middle of a run.
- Deep links only reach the paths the manifest declares. For any other screen push the route from the
  test through the router instead of relaunching with an intent.
- Re-resolve the scrollable before every gesture: a screen that rebuilds replaces the element, and a
  stale finder throws halfway through a pass.
- Two or three passes per screen are enough to see a screen is fine; spend repeated passes only on a
  screen where a candidate change has to beat the noise.

## 3. What paid off, and what did not

Paid off (verified by A/B):
- **`BackdropFilter` over a flat colour.** A gaussian blur of a uniform colour is that colour, so the
  filter paints nothing visible but costs one offscreen blur pass per widget per frame. Nine cards on
  one screen: raster time -19 % on average, frames over the raster budget -70 %. Put the filter back
  only where the widget really sits over imagery. Always keep a clip ancestor on any `BackdropFilter`.

Looked promising, did not clear the noise (do not retry without a new idea):
- Scroll-edge blur bands, scoped rebuilds that are already constant-height, deferring analytics setup
  out of bootstrap (gain below the spread of a cold-start measure).

Scrolling a well-built Flutter app is usually already 60 fps. Expect the wins at screen entry (first
paint raster spikes) and on screens that stack several blurs, shadows or `saveLayer`s.

## 4. Fluidity bugs that are not frame time

- **Unbounded network awaits.** `Future.wait` over N RPCs with no deadline, inside a `RefreshIndicator`
  (whose callback must complete) or a loading flag, keeps the spinner alive forever when one call never
  answers (socket dropped in the background). Give every call a deadline and a fallback:

  ```dart
  Future<T> rpcOrFallback<T>(Future<T> Function() call, T fallback,
      {Duration timeout = const Duration(seconds: 12)}) async {
    try { return await call().timeout(timeout); } catch (_) { return fallback; }
  }
  ```
  Test it with a future that never completes, one that throws, and one slower than the limit.
- A cache populated from a failed call (a `0` stored as if it were data) hides the failure for the
  rest of the session. Cache only successes.

## 5. Android security pass (quick, per build)

- Startup: wrap `main()` so any throw before `runApp` shows a plain failure screen instead of a white
  screen, and report the error.
- Manifest: `allowBackup="false"`, no `debuggable`, `exported` only where an intent filter needs it,
  remove disabled or commented-out activities, and accept only the URI schemes you handle (a
  `file://` intent filter on a document intake is an avoidable surface).
- Logs: no user ids, tokens, device tokens or emails in `debugPrint`/`Log`. A release build strips
  `debugPrint` only if you do it yourself; debug and profile builds print everything.
- Secrets: search for `sk_live`, `sk_test`, `service_role`, `-----BEGIN`. Client keys must be the
  publishable kind, never a service key.
- Build: R8 and `shrinkResources` on; obfuscation (`--obfuscate --split-debug-info`) only with the
  symbol files archived.
- Auth storage: on sign-out remove only the session keys, not the user's theme and language.

## 6. Report

For each kept change, record: screen, metric, before, after, percent, method, decision. Reverted
candidates go in a separate "not kept" table with the reason. One image per category (before and after
in two columns, kept changes only) makes the result easy to share.
