# XCode 13 Loop依赖报错问题排除记录


# DiaBox 过期了

diaBox 软件IOS版本过期了,没法子。官方说暂时没有办法，已经把代码开源了，只能自己编译。没搞过ios开发的只能硬着头皮做了。

拉取完 代码 `https://github.com/bubbledevteam/bubble-client-swift.git` 。头有点大Readme 里面是空的，xcode 也不熟悉，xcode 也不熟悉，后运行编译直接就报异常了：

```
import LoopKit 
// No such module 'LoopKit'
```

看样子应该是三分库引入问题。但是这个库怎么引入呢。查看一下代码目录：

```
./
├── BubbleClient
├── BubbleClient.xcodeproj
├── BubbleClientPlugin
├── BubbleClientTests
├── BubbleClientUI
├── Cartfile
├── Cartfile.resolved
├── Carthage
├── Common
├── CoreData
├── LICENSE
├── LibreSensor
├── New\ Group
├── README.md
├── Scripts
├── bin
├── singleviewtesterapp
└── tmp.xcconfig

```

搜索一下关键字 `Cartfile`。果然，这个就是三方模块导入的方式。学习了半天以后，原来要先按照编译
具体参考 https://www.jianshu.com/p/f33972b08648 和 https://www.cnblogs.com/strengthen/p/14084080.html 两篇文章
但是还是根据提示还是出现错误

```
A shell task (/usr/bin/xcrun lipo -create /Users/chinaddos/Library/Caches/org.carthage.CarthageKit/DerivedData/13.2_13C90/SwiftCharts/0.6.5/Build/Intermediates.noindex/ArchiveIntermediates/SwiftCharts/IntermediateBuildFilesPath/UninstalledProducts/iphoneos/SwiftCharts.framework/SwiftCharts /Users/chinaddos/Library/Caches/org.carthage.CarthageKit/DerivedData/13.2_13C90/SwiftCharts/0.6.5/Build/Products/Release-iphonesimulator/SwiftCharts.framework/SwiftCharts -output /Users/chinaddos/Workspace/yudi/bubble-client-swift/Carthage/Build/iOS/SwiftCharts.framework/SwiftCharts) failed with exit code 1:
fatal error: /Applications/Xcode.app/Contents/Developer/Toolchains/XcodeDefault.xctoolchain/usr/bin/lipo: /Users/chinaddos/Library/Caches/org.carthage.CarthageKit/DerivedData/13.2_13C90/SwiftCharts/0.6.5/Build/Intermediates.noindex/ArchiveIntermediates/SwiftCharts/IntermediateBuildFilesPath/UninstalledProducts/iphoneos/SwiftCharts.framework/SwiftCharts and /Users/chinaddos/Library/Caches/org.carthage.CarthageKit/DerivedData/13.2_13C90/SwiftCharts/0.6.5/Build/Products/Release-iphonesimulator/SwiftCharts.framework/SwiftCharts have the same architectures (arm64) and can't be in the same fat output file

Building universal frameworks with common architectures is not possible. The device and simulator slices for "SwiftCharts" both build for: arm64
Rebuild with --use-xcframeworks to create an xcframework bundle instead.
```

或者

```
youzi@you-iMac bubble-client-swift % bash Scripts/carthage.sh update --platform iOS
*** Fetching LoopKit
*** Fetching CGMBLEKit
*** Fetching SwiftCharts
*** Fetching dexcom-share-client-swift
*** Checking out LoopKit at "acd1212467ff7f7a3da6ff695f83764bbd0779a4"
*** Checking out SwiftCharts at "0.6.5"
*** Checking out dexcom-share-client-swift at "e3eb1b2baee97d95144c3f3f40c77ded42226604"
*** Checking out CGMBLEKit at "f8dd4bc3f025c0bd8964f36f7780fb7c4b756ecd"
*** xcodebuild output can be found in /var/folders/hf/862jyd094kz346scvjtkvv1r0000gn/T/carthage-xcodebuild.iQNWpz.log
*** Building scheme "SwiftCharts" in SwiftCharts.xcodeproj
Build Failed
	Task failed with exit code 1:
	/usr/bin/xcrun lipo -create /Users/youzi/Library/Caches/org.carthage.CarthageKit/DerivedData/13.2_13C90/SwiftCharts/0.6.5/Build/Intermediates.noindex/ArchiveIntermediates/SwiftCharts/IntermediateBuildFilesPath/UninstalledProducts/iphoneos/SwiftCharts.framework/SwiftCharts /Users/youzi/Library/Caches/org.carthage.CarthageKit/DerivedData/13.2_13C90/SwiftCharts/0.6.5/Build/Products/Release-iphonesimulator/SwiftCharts.framework/SwiftCharts -output /Users/youzi/Workspace/yudi/bubble-client-swift/Carthage/Build/iOS/SwiftCharts.framework/SwiftCharts

This usually indicates that project itself failed to compile. Please check the xcodebuild log for more details: /var/folders/hf/862jyd094kz346scvjtkvv1r0000gn/T/carthage-xcodebuild.iQNWpz.log
```

怎么回事，找一下github 上有人也遇到这个问题 因为我现在的是最新的XCode 13;原来的只支持XCode 12 ，血都吐干净了！！

```diff
chinaddos@liuyou-iMac bubble-client-swift % git diff Scripts/carthage.sh 
diff --git a/Scripts/carthage.sh b/Scripts/carthage.sh
index 3492f3c..7342243 100755
-- a/Scripts/carthage.sh
++ b/Scripts/carthage.sh
@@ -10,10 +10,10 @@ set -euo pipefail
 xcconfig=$(mktemp /tmp/static.xcconfig.XXXXXX)
 trap 'rm -f "$xcconfig"' INT TERM HUP EXIT
 
-# For Xcode 12 make sure EXCLUDED_ARCHS is set to arm architectures otherwise
+# For Xcode 13 make sure EXCLUDED_ARCHS is set to arm architectures otherwise
 # the build will fail on lipo due to duplicate architectures.
-echo 'EXCLUDED_ARCHS__EFFECTIVE_PLATFORM_SUFFIX_simulator__NATIVE_ARCH_64_BIT_x86_64__XCODE_1200 = arm64 arm64e armv7 armv7s armv6 armv8' >> $xcconfig
+echo 'EXCLUDED_ARCHS__EFFECTIVE_PLATFORM_SUFFIX_simulator__NATIVE_ARCH_64_BIT_x86_64__XCODE_1300 = arm64 arm64e armv7 armv7s armv6 armv8' >> $xcconfig
 echo 'EXCLUDED_ARCHS = $(inherited) $(EXCLUDED_ARCHS__EFFECTIVE_PLATFORM_SUFFIX_$(EFFECTIVE_PLATFORM_SUFFIX)__NATIVE_ARCH_64_BIT_$(NATIVE_ARCH_64_BIT)__XCODE_$(XCODE_VERSION_MAJOR))' >> $xcconfig
 
 export XCODE_XCCONFIG_FILE="$xcconfig"
"${SRCROOT}/bin/carthage" "$@"

```

重要没有报错了。开始下一步了 
