---
title: React Native iOS Podfile
catalog: true
date: 2020-08-30 19:10:49
subtitle:
header-img: head.jpg
tags:
- react-native
- iOS
- podfile
---

## React Native for iOS
`Podfile` 类似于 Android 工程的 `build.gradle`，定义了每个 target 的依赖及它们之间的关系。

如果想要集成 React Native 到现有应用中，或是为 React Native 添加新的依赖，就必须在 Podfile 中添加 React Native 相关的依赖库:
```ruby
# ios/Podfile
target 'my_project' do
  pod 'React', :path => '../node_modules/react-native', :subspecs => [
    'Core',
    'DevSupport',
    'RCTText',
    'RCTNetwork',
    'RCTWebSocket'
  ]
  pod "yoga", :path => "../node_modules/react-native/ReactCommon/yoga"
  pod 'DoubleConversion', :podspec => '../node_modules/react-native/third-party-podspecs/DoubleConversion.podspec'
  pod 'glog', :podspec => '../node_modules/react-native/third-party-podspecs/glog.podspec'
  pod 'Folly', :podspec => '../node_modules/react-native/third-party-podspecs/Folly.podspec'
end
```

然后安装它们:
```shell
pod install
```

特别地，当我们初始化一个 React Native 工程时，其工程目录默认是不含有 Podfile 的:
```tree
.
├── App.js
├── __tests__
│   └── App-test.js
├── app.json
├── babel.config.js
├── index.js
├── android
│   └── ...
├── ios
│   ├── my_project
│   │   ├── AppDelegate.h
│   │   ├── AppDelegate.m
│   │   ├── Base.lproj
│   │   │   └── LaunchScreen.xib
│   │   ├── Images.xcassets
│   │   │   ├── AppIcon.appiconset
│   │   │   │   └── Contents.json
│   │   │   └── Contents.json
│   │   ├── Info.plist
│   │   └── main.m
│   ├── my_project-tvOS
│   │   └── Info.plist
│   ├── my_project-tvOSTests
│   │   └── Info.plist
│   ├── my_project.xcodeproj
│   │   ├── project.pbxproj
│   │   └── xcshareddata
│   │       └── xcschemes
│   │           ├── my_project-tvOS.xcscheme
│   │           └── my_project.xcscheme
│   └── my_projectTests
│       ├── Info.plist
│       └── my_projectTests.m
├── node_modules
│   └── ...
├── metro.config.js
├── package.json
└── yarn.lock
```

因此，需要先在`ios`目录下执行下述命令，初始化环境:
```shell
pod init
```

以上，基于 React Native 0.59.9 版本，不同版本可能会有细微差别。

## 参考
- [Integration with Existing Apps](https://reactnative.dev/docs/integration-with-existing-apps#configuring-cocoapods-dependencies)
