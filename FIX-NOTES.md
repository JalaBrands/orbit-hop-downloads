# Orbit Hop – Android launch crash fix

## Symptom
`OrbitHop-Android-debug.apk` (release `v1.0-test-2026-09-04`) installs but the process dies
immediately on open on every device running Android 11 (API 30) or newer.

## Root cause
`MainActivity.onCreate()` calls `hideSystemBars()` **before** `setContentView()`.
On API 30+ `hideSystemBars()` runs `getWindow().getInsetsController()`, and
`PhoneWindow.getInsetsController()` is implemented as `return mDecor.getWindowInsetsController();`
with no null check (AOSP, android11-release through android16-release). The decor view is only
created by `setContentView()` / `addContentView()` / `getDecorView()`, so at that point `mDecor`
is null and the call throws `NullPointerException` inside `onCreate`, killing the app.
Devices below API 30 take the `getDecorView().setSystemUiVisibility(...)` branch, which creates
the decor as a side effect, so the bug only shows on modern phones.

## Source fix (for the next build)
```java
@Override
protected void onCreate(Bundle savedInstanceState) {
    super.onCreate(savedInstanceState);
    getWindow().setFlags(FLAG_KEEP_SCREEN_ON, FLAG_KEEP_SCREEN_ON);
    setContentView(new OrbitGameView(this));   // create the decor first
    hideSystemBars();                          // then touch the insets controller
}
```
Alternatively make `hideSystemBars()` use `getWindow().getDecorView().getWindowInsetsController()`.

## Patched build in this folder
`OrbitHop-Android-debug-fixed.apk` is the release APK with the two calls reordered in
`classes3.dex` (byte-for-byte same size; dex SHA-1/adler32 recomputed), zipaligned and re-signed
with a fresh debug key (v2 + v3). Verified with `dexdump`, `aapt2 dump badging`, `apksigner verify`.
Because the signing key differs from the original, **uninstall the old Orbit Hop first**,
then install the fixed APK.

SHA-256: 812185be474ac8f240d2b450874166e5437077e7efdaf2b7308c89afb092ccc7
