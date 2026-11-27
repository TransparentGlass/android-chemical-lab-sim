# Setup Guide

This guide covers setting up and running the `chemistry_lab` Flutter project on
Windows. Flutter includes the Dart SDK, so you normally install Flutter rather
than installing Dart separately.

## 1. Install Flutter and the required tools

1. Install the latest stable Flutter SDK by following the
   [Flutter installation guide](https://docs.flutter.dev/get-started/install).
   Add the Flutter `bin` directory to your `PATH`.
2. This project's `pubspec.yaml` requires Dart `^3.14.0`. Use a Flutter stable
   release whose bundled Dart SDK satisfies that constraint.
3. Install [Android Studio](https://developer.android.com/studio) if you want
   to run the Android app. In Android Studio's SDK Manager, install an Android
   SDK and platform tools. Create and start an Android Virtual Device (AVD), or
   connect an Android device with USB debugging enabled.
4. Open PowerShell and check the installation:

   ```powershell
   flutter --version
   dart --version
   flutter doctor
   ```

   Follow any `flutter doctor` instructions for the platforms you plan to use.
   You do not need to install Dart separately when using Flutter.

## 2. Get the project dependencies

In PowerShell, change to the project directory (the directory containing
`pubspec.yaml`), then fetch its dependencies:

```powershell
cd "path\to\android-chemical-lab-sim\chemistry_lab"
flutter pub get
```

Dependencies are declared in `pubspec.yaml` and resolved versions are recorded
in `pubspec.lock`. Run `flutter pub get` after cloning the project or changing
the dependency manifest.

### Add Flame (optional)

Flame is not currently listed as a project dependency. If you plan to build
game features with Flame, add it from the project directory:

```powershell
flutter pub add flame
```

This adds Flame to `pubspec.yaml` and resolves the package. Import it in Dart
code where needed, for example:

```dart
import 'package:flame/game.dart';
```

### JSON support

For basic JSON encoding and decoding, Dart already provides `dart:convert`;
there is no package to install:

```dart
import 'dart:convert';

final Map<String, dynamic> data =
    jsonDecode('{"name":"Water"}') as Map<String, dynamic>;
final String json = jsonEncode(data);
```

For generated JSON serialization code, add the packages only if the project
needs that workflow:

```powershell
flutter pub add json_annotation
flutter pub add --dev json_serializable build_runner
```

## 3. Run the app

Start an Android emulator or connect a device, then list available targets:

```powershell
flutter devices
```

From the project directory, start the app:

```powershell
flutter run
```

If more than one device is available, choose one using the device ID shown by
`flutter devices`:

```powershell
flutter run -d <device-id>
```

You can also run the app from VS Code after installing the Flutter extension:
open this project, select a device in the status bar, and press **F5**.

## 4. Useful commands

```powershell
flutter analyze
flutter test
flutter clean
flutter pub get
```

`flutter clean` removes generated build files. Run `flutter pub get` afterward
if needed.
