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


# example/android/app/src/main
# 添加从高德开放平台申请的key
<meta-data
      android:name="com.amap.api.v2.apikey"
      android:value="your api key"/>
```


## 编写flutterView 来展示android原生View

[hosting native views](https://flutter.dev/docs/development/platform-integration/platform-views?tab=android-platform-views-kotlin-tab)


## 在原生View里面展示地图

#### implement ActivityAware in AmapFlutter2Plugin(you plugin class)

```
// ActivityAware提供了监听Activity的生命周期相关的回调函数

package com.example.amap_flutter2

import android.Manifest
import androidx.annotation.NonNull
import io.flutter.embedding.engine.plugins.FlutterPlugin
import io.flutter.plugin.common.MethodCall
import io.flutter.plugin.common.MethodChannel
import io.flutter.plugin.common.MethodChannel.MethodCallHandler
import io.flutter.plugin.common.MethodChannel.Result
import io.flutter.embedding.engine.plugins.activity.ActivityAware
import io.flutter.embedding.engine.plugins.activity.ActivityPluginBinding
import androidx.core.app.ActivityCompat

/** AmapFlutter2Plugin */
class AmapFlutter2Plugin: FlutterPlugin, MethodCallHandler, ActivityAware {
    private lateinit var channel : MethodChannel
    private lateinit var flutterPluginBinding: FlutterPlugin.FlutterPluginBinding

    override fun onAttachedToEngine(@NonNull flutterPluginBinding: FlutterPlugin.FlutterPluginBinding) {
        channel = MethodChannel(flutterPluginBinding.binaryMessenger, "amap_flutter2")
        channel.setMethodCallHandler(this)

        this.flutterPluginBinding = flutterPluginBinding // 用来注册View
    }

    override fun onMethodCall(@NonNull call: MethodCall, @NonNull result: Result) { // not used yet
        if (call.method == "getPlatformVersion") {
            result.success("Android ${android.os.Build.VERSION.RELEASE}")
        } else {
            result.notImplemented()
        }
    }

    override fun onDetachedFromEngine(@NonNull binding: FlutterPlugin.FlutterPluginBinding) {
        channel.setMethodCallHandler(null)
    }

    // ActivityAware overrides
    override fun onAttachedToActivity(@NonNull binding: ActivityPluginBinding) {
        // regist native view, 只有注册过的View才能在flutter里显示
        // we need an activity to do lifecycle related work, so we pass a activity to AMapViewFactory
        flutterPluginBinding.platformViewRegistry.registerViewFactory("weride/AMapView", AMapViewFactory(binding.getActivity()))

        ActivityCompat.requestPermissions(  // request permission
            binding.getActivity(),
            arrayOf(Manifest.permission.ACCESS_COARSE_LOCATION,
                    Manifest.permission.ACCESS_FINE_LOCATION,
                    Manifest.permission.WRITE_EXTERNAL_STORAGE,
                    Manifest.permission.READ_EXTERNAL_STORAGE,
                    Manifest.permission.READ_PHONE_STATE),
            321
        )
    }

    override fun onDetachedFromActivity() {}

    override fun onReattachedToActivityForConfigChanges(@NonNull binding: ActivityPluginBinding) {
        // onAttachedToActivity(binding)
    }

    override fun onDetachedFromActivityForConfigChanges(){onDetachedFromActivity()}
}

```


#### add args to ViewFactory

```
package com.example.amap_flutter2

import android.content.Context
import android.view.View
import android.app.Activity
import io.flutter.plugin.common.StandardMessageCodec
import io.flutter.plugin.platform.PlatformView
import io.flutter.plugin.platform.PlatformViewFactory
import io.flutter.embedding.android.FlutterActivity

class AMapViewFactory(val activity: Activity) : PlatformViewFactory(StandardMessageCodec.INSTANCE) {
    override fun create(context: Context, viewId: Int, args: Any?): PlatformView {

        var creationParams = args as Map<String?, Any?>? //这里用来传递高德地图的初始化参数
        // 这里需要把Activity 转成 FlutterActivity类型
        var view = AMapView(context, viewId, activity as FlutterActivity, creationParams)

        return view
    }
}
```

#### implement DefaultLifecycleObserver in AMapView, and add args

```
import androidx.lifecycle.LifecycleOwner

import com.amap.api.maps.AMap
import com.amap.api.maps.AMapOptions
import com.amap.api.maps.CameraUpdateFactory
import com.amap.api.maps.TextureMapView
import com.amap.api.maps.MapView
import com.amap.api.maps.model.*
import io.flutter.embedding.android.FlutterActivity

// AMap.OnMapLoadedListener
// DefaultLifecycleObserver 这个借接口提供过滤Activety 生命周期相关的回调
internal class AMapView(context: Context, id: Int, activity: FlutterActivity, creationParams: Map<String?, Any?>?) : PlatformView, DefaultLifecycleObserver {
    private val mapView: MapView  // todo pass AMapOptions
    private var disposed = false

    override fun getView(): View {
        return mapView
    }

    override fun dispose() {}

    init {
        mapView = MapView(context, AMapOptions())
        activity.getLifecycle().addObserver(this)
    }

    fun setup() {
        mapView.onCreate(null)
    }

    // lifecycle 生命周期回调函数
    override fun onCreate(owner: LifecycleOwner) {
        mapView.onCreate(null)
    }

    // todo implement lifecycle methods
}
```

#### 检查效果
`flutter run`
