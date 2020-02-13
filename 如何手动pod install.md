首先，如果是卡在了pod repo update，也就是从远端拿这些库最新的地址tag之类的，这个时候可以让能够过这一步的同事把`/Users/用户名/.cocoapods/repos/trunk/Specs`整个文件夹压缩打包发给你，直接替换就好，这样就可以成功的过update那关啦。

Specs里面存了哪些东西呢？我们以一会儿想要实验的库libwebp为例吧：
![libwebp的各个版本](https://upload-images.jianshu.io/upload_images/5219632-7965183027f77c57.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

其实里面就是各个版本有个json以及json tag文件，我们打开json看一下：
```
{
  "name": "libwebp",
  "version": "1.1.0",
  "summary": "Library to encode and decode images in WebP format.",
  "homepage": "https://developers.google.com/speed/webp/",
  "authors": "Google Inc.",
  "license": {
    "type": "BSD",
    "file": "COPYING"
  },
  "source": {
    "git": "https://chromium.googlesource.com/webm/libwebp",
    "tag": "v1.1.0"
  },
  "compiler_flags": "-D_THREAD_SAFE",
  "requires_arc": false,
  "platforms": {
    "osx": "10.8",
    "ios": "6.0",
    "tvos": "9.0",
    "watchos": "2.0"
  },
  "pod_target_xcconfig": {
    "USER_HEADER_SEARCH_PATHS": "$(inherited) ${PODS_ROOT}/libwebp/ ${PODS_TARGET_SRCROOT}/"
  },
  "preserve_paths": "src",
  "default_subspecs": [
    "webp",
    "demux",
    "mux"
  ],
  "prepare_command": "sed -i.bak 's/<inttypes.h>/<stdint.h>/g' './src/webp/types.h'",
  "subspecs": [
    {
      "name": "webp",
      "source_files": [
        "src/webp/decode.h",
        "src/webp/encode.h",
        "src/webp/types.h",
        "src/webp/mux_types.h",
        "src/webp/format_constants.h",
        "src/utils/*.{h,c}",
        "src/dsp/*.{h,c}",
        "src/dec/*.{h,c}",
        "src/enc/*.{h,c}"
      ],
      "public_header_files": [
        "src/webp/decode.h",
        "src/webp/encode.h",
        "src/webp/types.h",
        "src/webp/mux_types.h",
        "src/webp/format_constants.h"
      ]
    },
    {
      "name": "demux",
      "dependencies": {
        "libwebp/webp": [

        ]
      },
      "source_files": [
        "src/demux/*.{h,c}",
        "src/webp/demux.h"
      ],
      "public_header_files": "src/webp/demux.h"
    },
    {
      "name": "mux",
      "dependencies": {
        "libwebp/demux": [

        ]
      },
      "source_files": [
        "src/mux/*.{h,c}",
        "src/webp/mux.h"
      ],
      "public_header_files": "src/webp/mux.h"
    }
  ]
}
```

然后我们就可以去source的git下载相应tag的库啦
```
"source": {
  "git": "https://chromium.googlesource.com/webm/libwebp",
  "tag": "v1.1.0"
},
```

![git库](https://upload-images.jianshu.io/upload_images/5219632-43bc5e9bfe983269.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

下载tar.gz或者zip都OK，然后解压放到我们project的本地pod里面：
![工程里面的pod目录](https://upload-images.jianshu.io/upload_images/5219632-9c291c904a2a5d74.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

> 注意哦你放进去的就是你从库仓库下载下来的文件，可能和你看到本地这个库的内容不一样，没关系的，你放进去再pod install过了以后就一样啦。

现在我们实现了下载需要的版本，并放入本地缓存，之后就可以来看如何pod install啦。在pod install的过程中，其实它就是比较了`Podfile.lock`文件和本地缓存文件版本差异，如果有区别就去远端下载，所以现在我们需要让本地缓存的库和`podfile.lock`里面一致，当然有一种方法是你把`podfile.lock`里面的库版本降低到你本地的版本，这样你也不用从远端下载就可以成功pod install啦，只是这样的话你需要经常stash着这个变化。

那么本地缓存的版本由什么管理呢？其实就是`项目/Pods/Manifest.lock`文件，你打开会发现和`Podfile.lock`文件非常一致，只要把自己替换的库例如libwebp相关的内容都从`Podfile.lock`拷到`项目/Pods/Manifest.lock`即可，如果`Podfile.lock`里面的版本不是最新的，你就手动改一下版本号以及spec checksum即可。

可参考：https://www.jianshu.com/p/06f507d2987d

例如podfile.lock里需要改成酱紫，manifest.lock必须相关的和podfile.lock保持一致，包括checksum的编号：
```
  - libwebp (1.1.0):
    - libwebp/demux (= 1.1.0)
    - libwebp/mux (= 1.1.0)
    - libwebp/webp (= 1.1.0)
  - libwebp/demux (1.1.0):
    - libwebp/webp
  - libwebp/mux (1.1.0):
    - libwebp/demux
  - libwebp/webp (1.1.0)

SPEC CHECKSUMS:
  ……
  KVOController: d72ace34afea42468329623b3379ab3cd1d286b6
  libwebp: 946cb3063cea9236285f7e9a8505d806d30e07f3
```

那么CHECKSUMS的编号是如何生成的呢？我最开始以为是远程仓库的commit号，后来查了一下看到上面的refer文章才知道是这么生成的：
```
pod ipc spec /Users/用户名/.cocoapods/repos/trunk/Specs/1/9/2/libwebp/1.1.0/libwebp.podspec.json  | openssl sha1
946cb3063cea9236285f7e9a8505d806d30e07f3
```

然后你再pod install就可以成功了吼~ （P.S. 想要知道自己卡在哪里，加上--verbose就可以啦）
