Library fork used to provide _an advanced Qt Media Player_ based on [FFmpeg][ffmpeg-home] library.
> [!NOTE]
> I'm not the original author of this library, this repository is a fork from [QtAvPlayer][lib-qtavplayer]

The library allows to:
- Demuxes and decodes _video_/_audio_/_subtitle_ frames.
- Muxes, encodes and saves the streams from different sources to one output file.
- Support [FFmpeg Bitstream Filters][ffmpeg-doc-filters-bitstreams] and [FFmpeg Filters][ffmpeg-doc-filters] including `filter_complex`.
- Multiple parallel filters for one input (one input frame produces multiple outputs).
- Decoding of all available streams at the same time.
- Support [hardware acceleration][anchor-platforms].
- It is up to an application to decide how to process the frames:
  - But there is _experimental_ support of converting the video frames to QtMultimedia's [QVideoFrame][qt-doc-qvideoframe] for copy-free rendering if possible.  
  _Note:_ Not all Qt's renders support copy-free rendering. Also QtMultimedia does not always provide public API to render the video frames. And, of course, for best performance both decoding and rendering should be accelerated.
  - Audio frames could be played by `QAVAudioOutput` which is a wrapper of QtMultimedia's [QAudioSink][qt-doc-qaudiosink]
- Accurate seek, it starts playing the closest frame.
- Might be used for media analytics software like [qctools][tools-qctools] or [dvrescue][tools-dvrescue].
- Implements and replaces a combination of _FFmpeg_ and _FFplay:_
```bash
# FFmpeg command-line
ffmpeg -i we-miss-gst-pipeline-in-qt6mm.mkv -filter_complex "qt,nev:er,wanted;[ffmpeg];what:happened" - | ffplay -

# Using QML or Qt Widgets
./qml_video :/valbok "if:you:like[cats];remove[this-sentence]"
```

**Table of contents:**
- [1. Library details](#1-library-details)
  - [1.1. Fork Purposes](#11-fork-purposes)
  - [1.2. Supported platforms](#12-supported-platforms)
- [2. Features](#2-features)
  - [2.1. Requirements](#21-requirements)
    - [2.1.1. C++ Standards](#211-c-standards)
  - [2.2. Dependencies](#22-dependencies)
- [3. How to build](#3-how-to-build)
  - [3.1. CMake usage](#31-cmake-usage)
  - [3.2. CMake options](#32-cmake-options)
- [4. How to use](#4-how-to-use)
  - [4.1. Logging category](#41-logging-category)
- [5. FFmpeg library usage](#5-ffmpeg-library-usage)
- [6. License](#6-license)

# 1. Library details
## 1.1. Fork Purposes

The fork is mainly due to the library being hard to build on mainline version due to multiple _CMake hacks:_ source directory had to be set manually, some directories where includes and exported when building target.  
So this fork aim to mainly fix those issues (and try some "workaround" to later propose a pull request to the original repository).

Current patches from this fork are all inside `main` branch, each features is represented by its own branch to ease synchronization with upstream project updates:
- Branch `refactor-library-structure`: All details are explained in the [pull request][pr-qtavplayer-lib-struct]. This is the main refactor to ease building the library
- Branch `add-log-categories`:
  -  Stop using `qDebug()` and use log levels instead (info, warning, critical, etc...)
  -  Allow to filter log according to the QtAvPlayer entity (player, demuxer, etc...), allowing more [fine-grain for logs][anchor-logs]

## 1.2. Supported platforms

Those symbols will be used:
- :dizzy:: Untested (either lack of time, either because unable to access the device to perform tests)
- :x:: Not working
- :last_quarter_moon:: Partial support
- :white_check_mark:: Tested and working

| / | Qt `5.12 -> 5.15.2` | Qt `6.X` | HW accelerated feature | Comments |
|:-:|:-:|:-:|:-:|:-:|
| Linux (Ubuntu) | :white_check_mark: | :white_check_mark: | :white_check_mark: | - Using `libva-drm` (also require packages: `libva-dev`, `libegl1-mesa-dev` and `libgles2-mesa-dev`)<br>- Using `libva-x11` (also require packages: `libva-dev`, `libva-x11-2` and `libx11-dev`)<br>- Using `libvdpau` (also require package: `libvdpau-dev`) |
| Windows | :white_check_mark: | :white_check_mark: | :white_check_mark: | Use API **d3d11** |
| MacOS | :white_check_mark: | :white_check_mark: | :white_check_mark: | Use APIs [CoreVideo][apple-doc-api-corevideo], [IOSurface][apple-doc-api-iosurface] and [Metal][apple-doc-api-iosurface] |
| iOS | :dizzy: | :dizzy: | :dizzy: | Use APIs [CoreVideo][apple-doc-api-corevideo], [IOSurface][apple-doc-api-iosurface] and [Metal][apple-doc-api-iosurface] |
| Android | :dizzy: | :dizzy: | :dizzy: | Use APIs [MediaCodec][android-doc-api-mediacodec] and [SurfaceTexture][android-doc-api-surfacetexture] |

# 2. Features

1. QAVPlayer supports playing from an url or QIODevice or from avdevice:
```cpp
player.setSource("http://clips.vorwaerts-gmbh.de/big_buck_bunny.mp4");
player.setSource("/home/lana/The Matrix Resurrections.mov");
// Dash player
player.setSource("https://bitdash-a.akamaihd.net/content/MI201109210084_1/m3u8s/f08e80da-bf1d-4e3d-8899-f0f6155f6efa.m3u8")

// Playing from qrc
QSharedPointer<QIODevice> file(new QFile(":/alarm.wav"));
file->open(QIODevice::ReadOnly);
QSharedPointer<QAVIODevice> dev(new QAVIODevice(file));
player.setSource("alarm", dev);

// Getting frames from the camera in Linux
player.setSource("/dev/video0");
// Or Windows
player.setInputFormat("dshow");
player.setSource("video=Integrated Camera");
// Or MacOS
player.setInputFormat("avfoundation");
player.setSource("default");
// Or Android
player.setInputFormat("android_camera");
player.setSource("0:0");

player.setInputOptions({{"user_agent", "QAVPlayer"}});
// Save to file
player.setOutput("output.mkv");
// Using various protocols
player.setSource("subfile,,start,0,end,0,,:/root/Downloads/why-qtmm-must-die.mkv");
```

2. Easy getting video and audio frames:
```cpp
QObject::connect(&player, &QAVPlayer::videoFrame, [&](const QAVVideoFrame &frame) {
    // QAVVideoFrame is comppatible with QVideoFrame
    QVideoFrame videoFrame = frame;
    
    // QAVVideoFrame can be converted to various pixel formats
    auto convertedFrame = frame.convert(AV_PIX_FMT_YUV420P);
    
    // Easy getting data from video frame
    auto mapped = videoFrame.map(); // downloads data if it is in GPU
    qDebug() << mapped.format << mapped.size;
    
    // The frame might contain OpenGL or MTL textures, for copy-free rendering
    qDebug() << frame.handleType() << frame.handle();
}, Qt::DirectConnection);

// Audio frames could be played using QAVAudioOutput
QAVAudioOutput audioOutput;
QObject::connect(&player, &QAVPlayer::audioFrame, [&](const QAVAudioFrame &frame) { 
    // Access to the data
    qDebug() << autioFrame.format() << autioFrame.data().size();
    audioOutput.play(frame);
}, Qt::DirectConnection);

QObject::connect(&p, &QAVPlayer::subtitleFrame, &p, [](const QAVSubtitleFrame &frame) {
    for (unsigned i = 0; i < frame.subtitle()->num_rects; ++i) {
        if (frame.subtitle()->rects[i]->type == SUBTITLE_TEXT)
            qDebug() << "text:" << frame.subtitle()->rects[i]->text;
        else
            qDebug() << "ass:" << frame.subtitle()->rects[i]->ass;
    }
}, Qt::DirectConnection);
```  

3. Each action is confirmed by a signal:
```cpp
// All signals are added to a queue and guaranteed to be emitted in proper order.
QObject::connect(&player, &QAVPlayer::played, [&](qint64 pos) { qDebug() << "Playing started from pos" << pos;  });
QObject::connect(&player, &QAVPlayer::paused, [&](qint64 pos) { qDebug() << "Paused at pos" << pos; });
QObject::connect(&player, &QAVPlayer::stopped, [&](qint64 pos) { qDebug() << "Stopped at pos" << pos; });
QObject::connect(&player, &QAVPlayer::seeked, [&](qint64 pos) { qDebug() << "Seeked to pos" << pos; });
QObject::connect(&player, &QAVPlayer::stepped, [&](qint64 pos) { qDebug() << "Made a step to pos" << pos; });
QObject::connect(&player, &QAVPlayer::mediaStatusChanged, [&](QAVPlayer::MediaStatus status) { 
    switch (status) {
        case QAVplayer::EndOfMedia:
            qDebug() << "Finished to play, no frames in queue"; 
            break;
        case QAVplayer::NoMedia:
            qDebug() << "Demuxer threads are finished";
            break;
        default:
            break;
        }
});
```

4. Accurate seek:
```cpp
QObject::connect(&p, &QAVPlayer::seeked, &p, [&](qint64 pos) { seekPosition = pos; });
QObject::connect(&player, &QAVPlayer::videoFrame, [&](const QAVVideoFrame &frame) { seekFrame = frame; });
player.seek(5000)
QTRY_COMPARE(seekPosition, 5000);
QTRY_COMPARE(seekFrame.pts(), 5.0);
```
If there is a frame with needed pts, it will be returned as first frame.

5. FFmpeg filters:
```cpp
player.setFilter("crop=iw/2:ih:0:0,split[left][tmp];[tmp]hflip[right];[left][right] hstack");
// Render bundled subtitles
player.setFilter("subtitles=file.mkv");
// Render subtitles from srt file
player.setFilter("subtitles=file.srt");
// Multiple filters
player.setFilters({
    "drawtext=text=%{pts\\\\:hms}:x=(w-text_w)/2:y=(h-text_h)*(4/5):box=1:boxcolor=gray@0.5:fontsize=36[drawtext]",
    "negate[negate]",
    "[0:v]split=3[in1][in2][in3];[in1]boxblur[out1];[in2]negate[out2];[in3]drawtext=text=%{pts\\\\:hms}:x=(w-text_w)/2:y=(h-text_h)*(4/5):box=1:boxcolor=gray@0.5:fontsize=36[out3]"
}); // Return frames from 3 filters with 5 outputs
```

6. Step by step:
```cpp
// Pausing will always emit one frame
QObject::connect(&player, &QAVPlayer::videoFrame, [&](const QAVVideoFrame &frame) { receivedFrame = frame; });
if (player.state() != QAVPlayer::PausedState) { // No frames if it is already paused
    player.pause();
    QTRY_VERIFY(receivedFrame);
}

// Always makes a step forward and emits only one frame
player.stepForward();
// the same here but backward
player.stepBackward();
```

7. Multiple streams:
```cpp
qDebug() << "Audio streams" << player.availableAudioStreams().size();
qDebug() << "Current audio stream" << player.currentAudioStreams().first().index() << player.currentAudioStreams().first().metadata();
player.setAudioStreams(player.availableAudioStreams()); // Return all frames for all available audio streams
// Reports progress of playing per stream, like current pts, fps, frame rate, num of frames etc
for (const auto &s : p.availableVideoStreams())
    qDebug() << s << p.progress(s);
```

8. Muxing the streams:
```cpp
// Muxes all streams to the file without reencoding the packets.
// `QAVMuxerPackets` is used internally.
player.setOutput("output.mkv");

// Multiple players could be used to mux to one files
QAVPlayer p1;
QAVPlayer p2;
QAVMuxerFrames m;

// Wait until QAVPlayer::LoadedMedia
QTRY_VERIFY(p1.mediaStatus() == QAVPlayer::LoadedMedia);
QTRY_VERIFY(p2.mediaStatus() == QAVPlayer::LoadedMedia);
// Use all available streams from both players
auto streams = p1.availableStreams() + p2.availableStreams();
// Mux the streams to one file
m.load(streams, "output.mkv");

QObject::connect(&p1, &QAVPlayer::videoFrame, &p1, [&](const QAVVideoFrame &f) { m.enqueue(f); }, Qt::DirectConnection);
QObject::connect(&p1, &QAVPlayer::audioFrame, &p1, [&](const QAVAudioFrame &f) { m.enqueue(f); }, Qt::DirectConnection);
QObject::connect(&p2, &QAVPlayer::videoFrame, &p2, [&](const QAVVideoFrame &f) { m.enqueue(f); }, Qt::DirectConnection);
QObject::connect(&p2, &QAVPlayer::audioFrame, &p2, [&](const QAVAudioFrame &f) { m.enqueue(f); }, Qt::DirectConnection);

p1.play();
p2.play();
```

9. HW accelerations:
- `VA-API` and `VDPAU` for **Linux**: the frames are returned with _OpenGL_ textures.
- `Video Toolbox` for **macOS** and **iOS**: the frames are returned with _Metal_ Textures.
- `D3D11` for **Windows**: the frames are returned with _D3D11Texture2D_ textures. 
- `MediaCodec` for **Android**: the frames are returned with _OpenGL_ textures.

> [!NOTE]
> Not all ffmpeg decoders or filters support HW acceleration. In this case software decoders are used.

1.  QtMultimedia could be used to render video frames to QML or Widgets. See [examples](examples)
2.  Widget `QAVWidget_OpenGL` could be used to render to OpenGL. See [examples/widget_video_opengl](examples/widget_video_opengl)
3.  Qt **5.12 - 6.x** is supported

## 2.1. Requirements
### 2.1.1. C++ Standards

This library requires at least **C++ 17** standard

## 2.2. Dependencies

Below, list of required dependencies:

| Dependencies | Packages | Comments |
|:-:|:-:|:-:|
| [Qt][qt-official] | / | Library built with **Qt framework** |
| [FFmpeg][ffmpeg-home] | `ffmpeg` | / |

> [!NOTE]
> Some platforms required more packages, please refer to [platform section][anchor-platforms] for more details.

# 3. How to build
## 3.1. CMake usage

_QtAvPlayer_ can be used as an embedded library in a subdirectory of your project (like a git submodule for example):
1. In the **root** CMakeLists, add instructions:
```cmake
add_subdirectory(qtavplayer) # Or if library is put in a folder "dependencies": add_subdirectory(dependencies/qtavplayer)
```

2. In the **application/library** CMakeLists, add instructions:
```cmake
target_link_libraries(${PROJECT_NAME} PRIVATE qtavplayer)
```

## 3.2. CMake options

This library provide some **CMake** build options:
- Features:
  - `QT_AVPLAYER_MULTIMEDIA` (default: `OFF`): enables support of `QtMultimedia` which will used Qt packages `Multimedia`.
  - `QTAVPLAYER_WIDGET_OPENGL` (default: `OFF`): builds the widget based on OpenGL which will used Qt packages `OpenGLWidgets` on Qt6 and `QWidgets` on Qt5.
- Hardware acceleration (see [platforms support section][anchor-platforms] for required packages):
  - `QTAVPLAYER_HW_SUPPORT_WINDOWS` (default: `ON`): enable HW acceleration for **Windows** platforms.
  - `QTAVPLAYER_HW_SUPPORT_MACOS` (default: `ON`): enable HW acceleration for **MacOS** platforms.
  - `QTAVPLAYER_HW_SUPPORT_LINUX_VA_DRM` (default: `OFF`): enable HW acceleration for **Linux** platforms using `libva-drm`.
  - `QTAVPLAYER_HW_SUPPORT_LINUX_VA_X11` (default: `OFF`): enable HW acceleration for **Linux** platforms using `libva-x11`.
  - `QTAVPLAYER_HW_SUPPORT_LINUX_VDPAU` (default: `OFF`): enable HW acceleration for **Linux** platforms using `libvdpau`.
  - `QTAVPLAYER_HW_SUPPORT_ANDROID` (default: `ON`): enable HW acceleration for **Android** platforms.
  - `QTAVPLAYER_HW_SUPPORT_IOS` (default: `ON`): enable HW acceleration for **iOS** platforms.

# 4. How to use
## 4.1. Logging category

Some classes have more fine-grained logging category:
- **Player backend** (which is _FFmpeg_): `qtavplayer.backend`
- **QAVPlayer:** `qtavplayer.player`
- **QAVVideoCodec:** `"qtavplayer.videocodec"`

Then we can enable/disable logs for those by using [QLoggingCategory][qt-doc-qlogging] appropriate method:
```cpp
// Manage Qt logging category
QLoggingCategory::setFilterRules(QStringLiteral(
    "qtavplayer.backend.*=true\n"           // Enable all logging level for "backend" category
    "qtavplayer.player.debug=false\n"       // Disable debug logging, all others are enabled
    "qtavplayer.videocodec.debug=false\n"
    "qtavplayer.videocodec.info=true"
));
```

# 5. FFmpeg library usage

FFmpeg documentation is available on [their website][ffmpeg-home] ([and API documentation][ffmpeg-doc-api]).  
FFmpeg allow to set multiple options through their dictionary:
- `fflags`: match flags defined under `AVFMT_FLAG_*` values, available at [avformat header API doc][ffmpeg-doc-api-fflags]
- `flags`: match flags defined under `AV_CODEC_FLAG_*` values, available at [avcodec header API doc][ffmpeg-doc-api-flags]
- Context options: multiple context structure fields can be set through the dictionary, like `max_delay`, `probesize`, etc.... Available fields can be seen at [AVFormatContext API doc][ffmpeg-doc-api-context]

# 6. License

[QtAvPlayer][lib-qtavplayer] library is released under [MIT License][repo-license], so this library too.

<!-- Link of this file -->
[anchor-logs]: #41-logging-category
[anchor-platforms]: #12-supported-platforms

<!-- Link of this repository -->
[repo-changelog]: CHANGELOG.md
[repo-license]: LICENSE.md

<!-- External links -->
[android-doc-api-mediacodec]: https://developer.android.com/reference/android/media/MediaCodec
[android-doc-api-surfacetexture]: https://developer.android.com/reference/android/graphics/SurfaceTexture

[apple-doc-api-corevideo]: https://developer.apple.com/documentation/corevideo
[apple-doc-api-iosurface]: https://developer.apple.com/documentation/iosurface
[apple-doc-api-metal]: https://developer.apple.com/metal/

[ffmpeg-home]: https://www.ffmpeg.org/
[ffmpeg-doc-api]: https://ffmpeg.org/doxygen/trunk/index.html
[ffmpeg-doc-api-context]: https://ffmpeg.org/doxygen/trunk/structAVFormatContext.html
[ffmpeg-doc-api-flags]: https://www.ffmpeg.org/doxygen/trunk/group__lavc__core.html
[ffmpeg-doc-api-fflags]: https://www.ffmpeg.org/doxygen/trunk/avformat_8h.html
[ffmpeg-doc-filters]: https://ffmpeg.org/ffmpeg-filters.html
[ffmpeg-doc-filters-bitstreams]: https://ffmpeg.org/ffmpeg-bitstream-filters.html

[lib-qtavplayer]: https://github.com/valbok/QtAVPlayer

[pr-qtavplayer-lib-struct]: https://github.com/valbok/QtAVPlayer/pull/550

[qt-doc-qaudiosink]: https://doc-snapshots.qt.io/qt6-dev/qaudiosink.html
[qt-doc-qlogging]: https://doc.qt.io/qt-6/qloggingcategory.html
[qt-doc-qvideoframe]: https://doc.qt.io/qt-6/qvideoframe.html
[qt-official]: https://www.qt.io/

[tools-qctools]: https://github.com/bavc/qctools
[tools-dvrescue]: https://github.com/mipops/dvrescue