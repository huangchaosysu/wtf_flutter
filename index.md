## [flutter 插件开发](/wtf_flutter/plugins)

## 系统已经存在较高版本， 此安装包无法安装
对于Android app， android/app/build.gradle里面有个flutterVersionCode, 对应Android的Build号， 这个数字也要递增, 当旧版本的Build号大于新版本的Build号的时候，就会出现这个错误
