# Run python on flutter

The project aims to run `youtube-dl` perfectly in Flutter.

Work in progreses but all the key parts have been written already

## Android

I built a python binary using termux.
Check this https://github.com/yausername/youtubedl-android/blob/master/BUILD_PYTHON.md

### 1. Build libpython

```
git https://github.com/termux/termux-packages
cd termux-packages
```

Create a file `build-python.sh`
Set your own arch, prefix, and android home variable.

```
#!/bin/bash
# use i686 for x86
export TERMUX_ARCH=arm    # arm, i686, x64, aarch64
export TERMUX_PREFIX=##
export TERMUX_ANDROID_HOME=##
./build-package.sh python
```

Run this

```
./scripts/run-docker.sh ./clean.sh
./scripts/run-docker.sh ./build-python.sh
```

```
cd debs

dpkg-deb -xv python_3.8.5_arm.deb .
dpkg-deb -xv libandroid-support_26_arm.deb .
dpkg-deb -xv libffi_3.3-2_arm.deb .
dpkg-deb -xv openssl_1.1.1g-4_arm.deb .
dpkg-deb -xv ca-certificates_20200722_all.deb .

dpkg-deb -xv python_3.8.5_i686.deb .
dpkg-deb -xv libandroid-support_26_i686.deb .
dpkg-deb -xv libffi_3.3-2_i686.deb .
dpkg-deb -xv openssl_1.1.1g-4_i686.deb .
dpkg-deb -xv ca-certificates_20200722_all.deb .

dpkg-deb -xv python_3.8.5_x86.deb .
dpkg-deb -xv libandroid-support_26_x86.deb .
dpkg-deb -xv libffi_3.3-2_x86.deb .
dpkg-deb -xv openssl_1.1.1g-4_x86.deb .
dpkg-deb -xv ca-certificates_20200722_all.deb .

dpkg-deb -xv python_3.8.5_aarch64.deb .
dpkg-deb -xv libandroid-support_26_aarch64.deb .
dpkg-deb -xv libffi_3.3-2_aarch64.deb .
dpkg-deb -xv openssl_1.1.1g-4_aarch64.deb .
dpkg-deb -xv ca-certificates_20200722_all.deb .
```

Then, you can get python binary in `debs/<your prefix>/bin/pyhton3.8`

### 2. Copy python3.8 binary to jniLibs directory

Copy to python3.8 binary to jniLibs directory on your flutter project.

```
cd <your flutter project>
cd android\app\src\main\jniLibs
mkdir arm64-v8a
cd arm64-v8a
copy <aarch64>/python3.8 libpython3.so
cd ..
  .
mkdir armeabi-v7a
cd armeabi-v7a
copy <arm>/python3.8 libpython3.so
cd ..
  .
mkdir x86
cd x86
copy <i386>/python3.8 libpython3.so
cd ..
  .
mkdir x86_64
cd x86_64
copy <x64>/python3.8 libpython3.so
```

### 3. Setting app/build.gradle

Setting default config like this

```
    defaultConfig {
        // TODO: Specify your own unique Application ID (https://developer.android.com/studio/build/application-id.html).
        applicationId "xyz.violet.communitydownloader"
        minSdkVersion 24
        targetSdkVersion 29
        versionCode flutterVersionCode.toInteger()
        versionName flutterVersionName
        ndk {
           abiFilters "arm64-v8a", "armeabi-v7a", "x86", "x86_64"
        }
    }
```

### 4. Create get native lib directory method

Change project name yourself

```kotlin
package xyz.violet.communitydownloader

import androidx.annotation.NonNull
import io.flutter.embedding.android.FlutterActivity
import io.flutter.embedding.engine.FlutterEngine
import io.flutter.plugin.common.MethodChannel

class MainActivity: FlutterActivity() {
    private val NATIVELIBDIR_CHANNEL = "xyz.violet.communitydownloader/nativelibdir";

    override fun configureFlutterEngine(@NonNull flutterEngine: FlutterEngine) {
        super.configureFlutterEngine(flutterEngine)
        MethodChannel(flutterEngine.dartExecutor.binaryMessenger, NATIVELIBDIR_CHANNEL).setMethodCallHandler {
            call, result ->
            // Note: this method is invoked on the main thread.
            // TODO
            if (call.method == "getNativeDir") {
                result.success(getApplicationContext().getApplicationInfo().nativeLibraryDir);
            }
            result.notImplemented()
        }
    }
}
```

### 5. Run python on flutter

This example runs `youtube-dl`

```dart
import 'dart:async';
import 'dart:convert';

import 'dart:io';
import 'dart:math';
import 'dart:typed_data';

import 'package:device_info/device_info.dart';
import 'package:flutter/foundation.dart';
import 'package:flutter/services.dart';
import 'package:flutter_archive/flutter_archive.dart';
import 'package:path/path.dart';
import 'package:path_provider/path_provider.dart';
import 'package:shared_preferences/shared_preferences.dart';
import 'package:shell/shell.dart';

typedef ExtractProgress = void Function(String file, double progress);

typedef DownloadPath = Future<void> Function(String path);
typedef DownloadProgress = Future<void> Function(double percent);

class YoutubeDLCallback {
  final DownloadPath pathCallback;
  final DownloadProgress progressCallback;

  YoutubeDLCallback({
    this.pathCallback,
    this.progressCallback,
  });
}

class YoutubeDLFormatInfo {
  final int formatCode;
  final String extension;
  final String resolution;
  final String note;
  final String size;
  final bool isVideoOnly;
  final bool isAudioOnly;

  YoutubeDLFormatInfo({
    this.formatCode,
    this.extension,
    this.resolution,
    this.note,
    this.size,
    this.isAudioOnly,
    this.isVideoOnly,
  });
}

// Youtube-DL Wrapper Class
// The reason ffi is not used is for licensing and error handling.
class YoutubeDL {
  static const platform =
      const MethodChannel('xyz.violet.communitydownloader/nativelibdir');
  static Directory nativeDir;

  static Future<Directory> getLibraryDirectory() async {
    if (nativeDir != null) return nativeDir;
    final String result = await platform.invokeMethod('getNativeDir');
    print(await getApplicationSupportDirectory());
    nativeDir = Directory(result);
    return nativeDir;
  }

  static Future<bool> extractRequire() async {
    final dir = await getApplicationSupportDirectory();
    final destinationDir = Directory(dir.path + '/python');
    if (!await destinationDir.exists()) return true;

    if (!await File(join(dir.path, 'python', 'usr', 'bin', 'python3'))
        .exists()) {
      await destinationDir.delete();
      return true;
    }

    return false;
  }

  // Extracting Python Binary from Assets
  //
  static Future<void> init(ExtractProgress prog) async {
    var mode = 'release';
    if (kDebugMode) mode = 'debug';
    final dir = await getApplicationSupportDirectory();
    final destinationDir = Directory(dir.path + '/python');
    if (await destinationDir.exists())
      await destinationDir.delete(recursive: true);

    var postfix = '';
    if (!kDebugMode) {
      final devicePlugin = DeviceInfoPlugin();
      final deviceInfo = await devicePlugin.androidInfo;
      final support64 = deviceInfo.supported64BitAbis;
      postfix = '-arm';
      if (support64 != null && support64.length > 0) postfix = '-aarch64';
    }

    final file = File(dir.path + "/youtube-dl-$mode.zip");
    final data =
        await rootBundle.load('assets/youtube-dl/youtube-dl-$mode$postfix.zip');
    if (data == null) return null;
    final createFile = await file.create();
    if (createFile == null) return null;
    final writeFile = await createFile.open(mode: FileMode.write);
    if (writeFile == null) return null;
    await writeFile.writeFrom(Uint8List.view(data.buffer));

    final zipFile = File(dir.path + '/youtube-dl-$mode.zip');
    try {
      await ZipFile.extractToDirectory(
        zipFile: zipFile,
        destinationDir: destinationDir,
        onExtracting: (zipEntry, progress) {
          prog(zipEntry.name, progress);
          return ExtractOperation.extract;
        },
      );
    } catch (e) {
      print(e);
    }
    file.deleteSync();

    await (await SharedPreferences.getInstance())
        .setBool('youtube-dl-init-q', true);
  }

  static Future<void> test() async {
    var dir = await getApplicationSupportDirectory();

    print(
        await requestThumbnail('https://www.youtube.com/watch?v=prTlJcipGbo'));
  }

  static bool checkHost(String host) {
    // This is not possible.
    return true;
  }

  // Get format info
  //
  // [info] Available formats for aW0cSY9gZ-8:
  // format code  extension  resolution note
  // 249          webm       audio only tiny   56k , opus @ 50k (48000Hz), 1.48MiB
  // 250          webm       audio only tiny   73k , opus @ 70k (48000Hz), 1.96MiB
  // 140          m4a        audio only tiny  131k , m4a_dash container, mp4a.40.2@128k (44100Hz), 3.81MiB
  // 251          webm       audio only tiny  140k , opus @160k (48000Hz), 3.85MiB
  // 160          mp4        144x144    144p   42k , avc1.4d400b, 25fps, video only, 589.79KiB
  // 278          webm       144x144    144p   58k , webm container, vp9, 25fps, video only, 690.64KiB
  // 133          mp4        240x240    240p  111k , avc1.4d400c, 25fps, video only, 1.27MiB
  // 242          webm       240x240    240p  150k , vp9, 25fps, video only, 1.35MiB
  // 134          mp4        360x360    360p  247k , avc1.4d4015, 25fps, video only, 2.48MiB
  // 243          webm       360x360    360p  275k , vp9, 25fps, video only, 2.33MiB
  // 244          webm       480x480    480p  410k , vp9, 25fps, video only, 3.84MiB
  // 1
  static Future<List<YoutubeDLFormatInfo>> getFormats(String url) async {
    var dir = await getApplicationSupportDirectory();
    var shell = await _createShell();

    var echo = await shell.start('./libpython3', [
      join(dir.path, 'python', 'usr', 'youtube_dl', '__main__.py'),
      '-F',
      url,
      '--cache-dir',
      join(dir.path, 'python', 'usr', 'bin', '.cache'),
    ]);

    var outputs = List<String>();

    // await echo.stdout.readAsString();
    await echo.stdout.listen((event) {
      var xx = List<int>.from(event);
      for (var ss in utf8.decode(xx).trim().split('\r'))
        outputs.addAll(ss.trim().split('\n'));
    }).asFuture();

    int starts = 0;
    for (; starts < outputs.length; starts++) {
      if (outputs[starts].trim().startsWith('[info]')) {
        starts += 2;
        break;
      }
    }

    if (starts == outputs.length) return null;

    var result = List<YoutubeDLFormatInfo>();

    for (; starts < outputs.length; starts++) {
      if (outputs[starts].trim() == '1') break;
      var s = outputs[starts]
          .trim()
          .split(' ')
          .where((element) => element != '')
          .toList();

      result.add(YoutubeDLFormatInfo(
        formatCode: int.parse(s[0]),
        extension: s[1],
        resolution:
            !outputs[starts].trim().contains('audio only') ? s[2] : null,
        note: !outputs[starts].trim().contains('audio only') ? s[3] : null,
        size: s.last,
        isAudioOnly: outputs[starts].trim().contains('audio only'),
        isVideoOnly: outputs[starts].trim().contains('video only'),
      ));
    }

    print(result);
  }

  static Future<String> requestThumbnail(String url,
      [List<String> options]) async {
    options ??= List<String>();

    var dir = await getApplicationSupportDirectory();
    var shell = await _createShell();

    var pparam = List<String>.from(options);

    pparam.insert(0, url);
    pparam.insert(0, '--get-thumbnail');
    pparam.insert(0, '-q');
    pparam.insert(
        0, join(dir.path, 'python', 'usr', 'youtube_dl', '__main__.py'));
    pparam.add('--cache-dir');
    pparam.add(join(dir.path, 'python', 'usr', 'bin', '.cache'));
    var echo = await shell.start('./libpython3.so', pparam);

    var thumbnail = await echo.stdout.readAsString();
    var err = (await echo.stderr.readAsString()).trim();
    if (err.length != 0) throw Exception(err);

    return thumbnail.trim();
  }

  //  Request download from url
  static Future<void> requestDownload(
      YoutubeDLCallback callback, String url, String path,
      [String format, List<String> options]) async {
    options ??= List<String>();

    var dir = await getApplicationSupportDirectory();
    var shell = await _createShell();

    var pparam = List<String>.from(options);

    if (format != null) {
      pparam.insert(0, format);
      pparam.insert(0, '-f');
    }
    pparam.insert(0, url);
    pparam.insert(0, join(path, "%(extractor)s", "%(title)s.%(ext)s"));
    pparam.insert(0, '-o');
    pparam.insert(
        0, join(dir.path, 'python', 'usr', 'youtube_dl', '__main__.py'));
    pparam.add('--cache-dir');
    pparam.add(join(dir.path, 'python', 'usr', 'bin', '.cache'));
    var echo = await shell.start('./libpython3.so', pparam);

    var prog = RegExp(r'\[download\]\s+(\d+(\.\d)?)%.*?');

    var cannot = false;
    await echo.stdout.listen((event) {
      if (cannot) return;
      var xx = List<int>.from(event);
      for (var ss in utf8.decode(xx).trim().split('\r')) {
        print(ss);
        if (ss.startsWith('[download]')) {
          if (ss.contains('has already been downloaded')) {
            // just keep going
            cannot = true;
            return;
          }

          if (ss.contains('Destination')) {
            callback.pathCallback(ss.split('Destination:').last.trim());
            continue;
          }

          var pp = prog.allMatches(ss);
          callback.progressCallback(double.parse(pp.first[1]));
        }
      }
    }).asFuture();

    var err = (await echo.stderr.readAsString()).trim();
    if (err.length != 0) throw Exception(err);
  }

  // TODO:
  static Future<void> requestDownloadWithFFmpeg(
      YoutubeDLCallback callback, String url, String path,
      [String format, List<String> options]) async {
    options ??= List<String>();

    var dir = await getApplicationSupportDirectory();
    var shell = await _createShell();

    var pparam = List<String>.from(options);

    if (format != null) {
      pparam.insert(0, format);
      pparam.insert(0, '-f');
    }
    pparam.insert(0, url);
    pparam.insert(0, join(path, "%(extractor)s", "%(title)s.%(ext)s"));
    pparam.insert(0, '-o');
    pparam.insert(
        0, join(dir.path, 'python', 'usr', 'youtube_dl', '__main__.py'));
    pparam.add('--cache-dir');
    pparam.add(join(dir.path, 'python', 'usr', 'bin', '.cache'));
    var echo = await shell.start('./libpython3.so', pparam);

    var prog = RegExp(r'\[download\]\s+(\d+(\.\d)?)%.*?');

    await echo.stdout.listen((event) {
      var xx = List<int>.from(event);
      for (var ss in utf8.decode(xx).trim().split('\r')) {
        print(ss);
        if (ss.startsWith('[download]')) {
          if (ss.contains('Destination')) {
            callback.pathCallback(ss.split('Destination:').last.trim());
            continue;
          }

          var pp = prog.allMatches(ss);
          callback.progressCallback(double.parse(pp.first[1]));
        }
      }
    }).asFuture();

    print(await echo.stderr.readAsString());
  }

  //  Test youtube-dl command line argument
  //
  static Future<void> _testCommand(List<String> param) async {
    var dir = await getApplicationSupportDirectory();
    var shell = await _createShell();
    var pparam = List<String>.from(param);

    pparam.insert(0, '../youtube_dl/__main__.py');
    pparam.add('--cache-dir');
    pparam.add(join(dir.path, 'python', 'usr', 'bin', '.cache'));
    var echo = await shell.start('python3', pparam);

    await echo.stdout.listen((event) {
      var xx = List<int>.from(event);
      for (var ss in utf8.decode(xx).trim().split('\r')) print(ss.trim());
    }).asFuture();
    print(await echo.stderr.readAsString());
  }

  static Future<Shell> _createShell() async {
    var shell = new Shell();
    var dir = await getApplicationSupportDirectory();
    var bin = await getLibraryDirectory();

    shell.navigate(bin.path);
    shell.environment['LD_LIBRARY_PATH'] =
        join(dir.path, 'python', 'usr', 'lib');
    shell.environment['SSL_CERT_FILE'] =
        join(dir.path, 'python', 'usr', 'etc', 'tls', 'cert.pem');
    shell.environment['PYTHONHOME'] = join(dir.path, 'python', 'usr');

    return shell;
  }
}
```
