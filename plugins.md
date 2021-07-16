本文以高德地图为例，记录flutter插件开发的过程

## Step 1， 创建项目
```
flutter create --template=plugin amap_flutter2
```

执行完上述命令， 会看到如下提示
```
Your plugin code is in amap_flutter2/lib/amap_flutter2.dart.
You example app code is in amap_flutter2/example/lib/main.dart.
To add platforms, run `flutter create -t plugin --platforms <platforms> .` under amap_flutter2.
```

## Step2, 添加支持的系统类型(android)
```
flutter create -t plugin --platforms=android .
```

Step3, 更新pubspec.yaml
```
flutter:
  plugin:
    platforms:
      android:
        package: com.example.amap_flutter2  # 位于android/src/main下的完整路径
        pluginClass: AmapFlutter2Plugin  # 自动生产的plugin类名
```

## step4, Add amap
```
# android/build.gradle

dependencies {
    implementation "org.jetbrains.kotlin:kotlin-stdlib-jdk7:$kotlin_version"
    compile fileTree(dir: 'libs', include: ['*.jar'])
    //3D地图so及jar
    compile 'com.amap.api:3dmap:latest.integration'
    //定位功能
    compile 'com.amap.api:location:latest.integration'
    //搜索功能
    compile 'com.amap.api:search:latest.integration'
}
```

```
# android/src/main/androidmanifest.xml

<manifest xmlns:android="http://schemas.android.com/apk/res/android"
  package="com.example.amap_flutter2">

  <!--允许程序打开网络套接字-->
  <uses-permission android:name="android.permission.INTERNET" />
  <!--允许程序设置内置sd卡的写权限-->
  <uses-permission android:name="android.permission.WRITE_EXTERNAL_STORAGE" />   
  <!--允许程序获取网络状态-->
  <uses-permission android:name="android.permission.ACCESS_NETWORK_STATE" /> 
  <!--允许程序访问WiFi网络信息-->
  <uses-permission android:name="android.permission.ACCESS_WIFI_STATE" /> 
  <!--允许程序读写手机状态和身份-->
  <uses-permission android:name="android.permission.READ_PHONE_STATE" />     
  <!--允许程序访问CellID或WiFi热点来获取粗略的位置-->
  <uses-permission android:name="android.permission.ACCESS_COARSE_LOCATION" /> 
</manifest>
```
