# android-bouncing

**Box2D physics on a phone, written in sysl, drawn by SDL3.** A boxful of discs, squares and triangles
with no gravity, perfect restitution and a little friction, so nothing ever settles and everything
tumbles. Tap to throw in another.

It is the demo for [androidkit](https://github.com/sysl-lang/androidkit), which is the template you
copy to start an Android project. Everything about *how* a sysl program becomes an APK is documented
there — the `SDL_main` export, the CMake inversion, the JNI bridge for the system-bar insets, the
orientation and theme traps. This README covers only what is particular to the demo.

```
sysl-lang/sdl3    windows, rendering, events and input
sysl-lang/box2d   rigid-body physics — Box2D v3, vendored
```

## Neither package knew it was on a phone

The demo is Box2D physics drawn by SDL3: a boxful of discs, squares and triangles with **no gravity**,
perfect restitution and no friction, so nothing ever settles. One body carries an image. Tap to throw
in another.

**Two of Box2D's defaults have to be turned off or it all stops**, and neither is a fact about physics:

- **`restitution_threshold`** — Box2D ignores restitution below a relative speed, one metre per second
  by default, because a stack that keeps bouncing a little is jitter. Here it means every glancing hit
  is inelastic and the box goes quiet within a minute. Zero says *always bounce*.
- **`contact_damping_ratio`** — the default of 10 is heavily damped, which is right for a pile that
  should settle and wrong for this.

**And the two settings are not enough on their own, because a rigid-body solver is not
energy-preserving and no tuning makes Box2D one.** A soft-step integrator applies restitution once
per contact with a bounded impulse, so it under-restores; and the friction below dissipates whenever
a contact slides. So the demo **measures the kinetic energy and puts it back**: `½mv² + ½Iω²` over
every loose body, and since scaling every velocity by `s` scales energy by `s²`, restoring a target
is one square root and one pass.

That is a governor rather than physics, and it is the honest way to have a demo that never winds
down — the alternative is pretending a number that keeps falling is conserved. The target rises
whenever it is exceeded, so a tap adds its body's energy to the budget rather than being scaled away.

**And a little friction, which is what lets a collision change a body's spin at all.** Every impulse
on a frictionless disc points along the line between the two centres, so it passes through the centre
of mass and its torque is exactly zero — a frictionless circle can only ever keep the spin it was
thrown with. Box2D mixes the two shapes' friction as their geometric mean, so one frictionless party
makes the whole contact frictionless, which is why it is on the walls and on everything thrown rather
than on some of it.

It is also the one thing here that genuinely dissipates, since sliding contact does — which is half
of what the governor above exists to make up.

**Neither `sh.sysl.sdl3` nor `sh.sysl.box2d` needed a line changed to run here**, and that is the
claim the demo exists to make:

- **box2d** is vendored Box2D v3 — portable C with `requires { heap, posix }`, both of which Bionic
  answers. The NDK compiles it like any other C: 40 archive members, 660 `b2*` symbols.
- **sdl3** is identical on a phone because `15 §8` says a `@link` directive names a *library* and
  never a path. Only where the library lives changes, and that is `CMakeLists.txt`'s business.
- **Taps need no finger handling.** SDL synthesises mouse events from touch, so
  `EventKind.MouseButtonDown` with `mouse_x`/`mouse_y` is the whole of the input — and the same
  source runs on a desktop.

**One thing did have to change, and it was the compiler.** An Android program is always loaded as a
`.so`, and a vendored library's globals are ordinary C globals — preemptible — so `ld.lld` refused
them:

```
ld.lld: error: relocation R_AARCH64_ADR_PREL_PG_HI21 cannot be used against symbol
  'b2AssertHandler'; recompile with -fPIC
```

`Toolchain.compileC` passed `-fPIC` nowhere. `targets.md § Android` had concluded no relocation model
was needed, which was measured on sysl's *own* object — where every global is `Linkage.Private` and so
not preemptible — and did not cover the C a package carries. Fixed in the compiler, not here.

## The Java half is Scala

**There is no Java in this repository and no Kotlin either.** `MainActivity` is Scala 3, and the
Scala standard library is a dependency of the application — so an activity here can use the language
and not only its syntax. The demo proves it rather than claiming it: the insets are logged through a
`List`, a `zip`, a `map` and an interpolated string, none of which links without the runtime.

**What it costs is a second build system, and that is the whole of the cost.** The Android Gradle
plugin compiles Java and Kotlin itself and has no Scala support, so `activity/` is an sbt project and
`app/build.gradle.kts` runs it as a task and puts the jar on the classpath. `./gradlew assembleDebug`
is still the one command; `sbt` has to be installed.

**Two things bite, and both are silent:**

- **A `private` `@native` method does not work.** Scala renames a private method reached from an
  inner class to `sh$sysl$bouncing$MainActivity$$nativeSetSystemBars` so the inner class can see
  it, and JNI then looks for a symbol with `_00024` in it that nothing defines. It compiles, links,
  and dies at the first call with an `UnsatisfiedLinkError`. The listener is an inner class, so this
  is exactly that case — leave the method non-private.
- **`minSdk` is 26 because of `scala-library`.** `d8` refuses to dex it below that — *"Increase the
  minSdkVersion to 26 or above"* — so an APK carrying the Scala runtime starts at Android 8.0. SDL's
  own floor is 21 and sysl's triple states 24; the three do not have to agree and the higher wins.

For comparison, Kotlin costs **nothing at all** under AGP 9 — it is compiled by AGP itself, and the
`org.jetbrains.kotlin.android` plugin is refused outright as no longer required. Scala is here
because it is what this project is written in everywhere else.

## One ABI

`arm64-v8a`, and that is not an apology. On an Apple Silicon host the emulator runs `arm64-v8a`, and
so does every Android device made since 2015 — one build covers both. `x86_64` matters only on an
Intel host or a CI runner, and `armeabi-v7a` only for pre-2015 hardware; either is a line in
`app/build.gradle.kts` and a registry row in the compiler that does not exist yet.

## Not a framework, and not yet named

`sdl3` + sysl + one shell per surface is framework-shaped, and the layer that would carry a name —
the surface, input and lifecycle abstraction — does not exist. `picokit` hand-wires it for a panel
and this hand-wires it for a phone. **Extracting it from one surface would be a guess**; it gets
extracted when a second one makes the duplication visible. `bouncing` keeps its name either way: a
template is named for what it starts, which is why `picokit` is named for a board.

## License

ISC. See `LICENSE`.
