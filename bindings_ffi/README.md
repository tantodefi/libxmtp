# Uniffi-based bindings for XMTP v3

This crate provides cross-platform Uniffi bindings for XMTP v3.

# Status

- Android is tested end-to-end via an example app in `../examples/android`.
- iOS has not been tested.
- Dev network is NOT supported, only local network is currently supported.

# Consuming this crate (Android example)

The generated artifacts of this crate are the bindings interface (`xmtpv3.kt`) generated by `uniffi`, and the cross-compiled binaries (`jniLibs/`) generated by `cross`. The binaries are checked into source so that you don't have to recompile.

- Run `./gen_kotlin.sh` to generate the bindings interface.
- Run `./setup_android_example.sh` to copy these artifacts into the example Android app. Alternatively, modify the script to set up an app of your choice.
- Open the `build.gradle` of the example Android app in Android Studio.
- Run the local server via `dev/up` from the root of this repo. If running from elsewhere, make sure your `docker-compose.yml` matches the one in this repo.

# Rebuilding this crate

The cross-compiled binaries (`jniLibs`) have been committed alongside the crate so you do not need to rebuild unless you make changes. The build is very slow (~3 mins on incremental builds, ~30 mins on full builds, per-target). Future changes will simplify the process and improve the build time as well as setting up async builds in CI.

- Run `./gen_kotlin.sh` to regenerate the bindings
- Install Docker
- Install Cross for zero setup cross-platform builds: `cargo install cross --git https://github.com/cross-rs/cross`
- Run `./cross_build.sh` to cross-compile (this is SLOW)

`Cross` allows us to run cross-platform builds without needing to download all of the relevant toolchains and should work regardless of your host OS. It is possible that the build time can be improved by building natively without Cross.

# Running tests

Ensure a local API host is running - run `dev/up` from the repo root.

You'll need to do the following one-time setup to run Kotlin tests:

- Run `brew install kotlin` to get `kotlinc`
- Install the JRE from `www.java.com`
- Run `make install-jar` and paste the command in the output to add the jars to your CLASSPATH environment variable

If you want to skip the setup, you can also run `cargo test -- --skip kts` to only run Rust unit tests. CI will run all tests regardless.

# Debugging

There is no support for using breakpoints or a debugger with FFI currently. Methods of debugging:

1. Examine any error messages that are returned from libxmtp.
1. Produce a minimal repro of the issue.
1. Use platform-native logging:
   1. Set up an FFI logger for your platform ([example](https://github.com/xmtp/libxmtp/blob/7e7bf7aabe7c758507ae982834d583c1d88c3ce2/bindings_ffi/examples/MainActivity.kt#L33))
   1. Add logs where you need them ([example](https://github.com/xmtp/libxmtp/assets/696206/bb1be87e-7a9b-47f2-a0f4-e93a92346b18))
   1. Examine the logs for your platform (for example, on Android emulator this would be logcat in Android Studio)
1. Examine the database in your app
   1. Use logs to find the location of the Sqlite database and the database encryption key
   1. Find the database (for example, on Android emulator use Device File Explorer in Android Studio)
   1. `brew install sqlcipher`
   1. Copy the database to your local machine and open it on the command line using `sqlcipher <file>.db3`
   1. Decrypt the database if needed [as follows](https://utelle.github.io/SQLite3MultipleCiphers/docs/configuration/config_sql_pragmas/#pragma-key) (or disable database encryption before running the app)
      1. Note that decryption is buggy if using `sqlite3` - `sqlcipher` is necessary here

# Releasing new version

Tag the commit you want to release with the appropriate version (e.g. 0.3.0-beta0).
The Release github workflow will run the following jobs:

- android

  - downloads the `libxmtp-android.zip` build artifact
  - make a release tagged the same way with the artifact attached

- swift
  - downloads the `libxmtp-swift.zip` build artifact
  - checks out `libxmtp-swift` repo and updates it with the contents of the zip file
  - pushes new commit to the `libxmtp-swift` repo and tags it with the same tag

NOTES: To allow the workflow to push to another repo the setup follows [this guide](https://docs.github.com/en/apps/creating-github-apps/authenticating-with-a-github-app/making-authenticated-api-requests-with-a-github-app-in-a-github-actions-workflow#authenticating-with-a-github-app). It uses [this app installed on the org](https://github.com/organizations/xmtp/settings/apps/libxmtp-release). The relevant secrets are stored only [in this repo](https://github.com/xmtp/libxmtp/settings/secrets/actions). If additional repos are added to this workflow they MUST be added to [this installation](https://github.com/organizations/xmtp/settings/installations/39118494) of the app.

# Uniffi

We are using Uniffi with the latest procedural macros syntax where possible, which also gives us async support. It is important to learn the syntax: <https://github.com/mozilla/uniffi-rs/blob/main/docs/manual/src/proc_macro/index.md>.

For the most part, any mistakes in the Uniffi interface will manifest as a compile error when running `./gen_kotlin.sh`. Some details are described below so that they are easier to understand.

## Object Lifetimes

Any objects crossing the Uniffi interface boundary must be wrapped in `Arc<>`, so that Uniffi can marshall it back and forth between [raw pointers](https://mozilla.github.io/uniffi-rs/internals/object_references.html#lifetimes) before passing it to the foreign language. The usage of Arc means that we do not need to manually destroy objects on the Rust side, however depending on the target platform, the foreign language may need to automatically or manually [release the pointer back to Rust](https://mozilla.github.io/uniffi-rs/kotlin/lifetimes.html) when done.

## Async and concurrency

We use Tokio as our [async runtime](https://rust-lang.github.io/async-book/08_ecosystem/00_chapter.html). Uniffi can use this runtime on async methods and objects using the annotation `#[uniffi::export(async_runtime = ‘tokio’)]`. Uniffi plumbs up an executor (scheduler) in the foreign language to the Tokio runtime in Rust. More details [here](https://github.com/mozilla/uniffi-rs/blob/734050dbf1493ca92963f29bd3df49bb92bf7fb2/uniffi_core/src/ffi/rustfuture.rs#L11-L18). By default, Tokio's [multi-thread scheduler](https://docs.rs/tokio/latest/tokio/runtime/index.html#multi-thread-scheduler) is used on the Rust side. This means that functions may resume execution in a different thread when an `await` is encountered. This in turn means that structs held across an `await` in libxmtp must be `Send` and `Sync`.

Additionally, because the foreign language may be multi-threaded, any objects passed to the foreign language must also be `Send` and `Sync`, and [no references to `&mut self` are permitted](https://mozilla.github.io/uniffi-rs/udl/interfaces.html#concurrent-access). For now, use the mutability pattern described in <https://github.com/xmtp/libxmtp/pull/138>.
