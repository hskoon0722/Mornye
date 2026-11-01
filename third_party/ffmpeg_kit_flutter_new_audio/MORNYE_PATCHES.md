# Local FFmpegKit patch

Based on ffmpeg_kit_flutter_new_audio 2.5.7 (license retained in LICENSE).
The upstream `example` app is not copied.

The root `pubspec.yaml` overrides the package with this copy
(`dependency_overrides: ffmpeg_kit_flutter_new_audio: path:
third_party/ffmpeg_kit_flutter_new_audio`), next to the pinned
`ffmpeg_kit_flutter_new_audio: 2.5.7`. The version is the one in this
folder's `pubspec.yaml` (2.5.7, unchanged).

## Patches

### Linux architectures without a bundle

Upstream publishes Linux bundles for x86_64 only, and `linux/CMakeLists.txt`
stopped the build with a fatal error on any other architecture, so the app
could not be built for Linux arm64 at all.

On an architecture without a bundle, and with no `FFMPEGKIT_LOCAL_DIR`, the
plugin is now built without `libffmpegkit.so` and a CMake warning says so.
The plugin already loads the library with `dlopen()` and answers every call
with an error when it is missing. Mornye desktop never calls FFmpegKit's
native layer: it runs the bundled `ffmpeg` and `ffprobe` executables
(`lib/services/ffmpeg_desktop.dart` in the app) and uses only the package's
Dart code (argument parsing and the media information model).

x86_64 builds and builds with `FFMPEGKIT_LOCAL_DIR` keep upstream behavior.

### Windows: load libffmpegkit.dll lazily and never unload it

Upstream loads `libffmpegkit.dll` in the plugin constructor and calls
`FreeLibrary` in the destructor. Loading the library runs its static
initializer, which starts the log callback thread (`enableRedirection`);
nothing stops that thread before `FreeLibrary`. Unloading libffmpegkit
also unloads `libwinpthread-1.dll` under the running thread, and the app
crashes with `0xc0000005` in `libwinpthread-1.dll_unloaded`.

The plugin now loads the library on the first method call (and registers
the event callbacks then if the event channel is already listening) and
never unloads it; the process exit releases it. Mornye desktop never calls
the native layer, so the library and its thread are normally not loaded.
