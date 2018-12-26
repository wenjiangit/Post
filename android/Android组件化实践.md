# Android组件化实践

## 什么是组件化?

我所理解的组件化即是将项目按业务逻辑拆分成若干个模块,每个模块以Android Studio 中的Module为载体,模块之间没有相互依赖关系,它们依赖于公共的基础模块,模块之间使用路由进行通信,单个模块可以在开发时作为一个Application独立运行调试,而在打包时则作为一个Android Libary被App模块依赖,能够实现动态切换.

## 如何实现组件化?

方法论:

1. 抽取公共基础模块,如Utils,自定义控件,网络请求,数据库等.
2. 模块化,即将业务逻辑进行拆分.使用Module承载
3. 配置Gradle脚本,能够动态切换library和Application
4. 加入路由模块.实现模块间通信.

前两步略过,每个项目的业务逻辑都不一样

####  配置Gradle脚本

- 新建``config.gradle`` 文件,置于项目根目录,如下

  ```groovy
  ext {

    //定义切换开关
      isModule = true
    //统一管理公共变量
      android = [
              compileSdk              : 26,
              minSdk                  : 16,
              targetSdk               : 26,
              versionCode             : 1,
              versionName             : "1.0",
              buildToolsVersion       : '26.0.2',
              supportVersion          : "26.1.0",
              constraintVersion       : "1.0.2",
              butterknifeVersion      : "8.8.1",
              glideVersion            : '3.7.0',
              rx2FastAndroidNetworking: '1.0.1',
              retrofitVersion         : '2.3.0',
              dagger2Version          : '2.8',
              rxjava2Version          : '2.0.6',
              rxandroidVersion        : '2.0.1',
              gsonVersion             : '2.8.0',
              timberVersion           : '4.5.1',
              roomVersion             : "1.0.0",
              recyclerAdapterHelper   : '2.9.30',
              matisseVersion          : "0.4.3"

      ]

    //配置模块的applicationId,因为当library转换成Application时需要一个id
      appId = [
              app      : "com.wenjian.myplayer",
              home     : "com.wenjian.home",
              finder   : "com.wenjian.finder",
              videoplay: "com.wenjian.videoplay",
              mine     : "com.wenjian.mine"
      ]


    //统一管理依赖库

      dependencies =
              [
                      "appcompat-v7"                 : "com.android.support:appcompat-v7:$android.supportVersion",
                      "recyclerview-v7"              : "com.android.support:recyclerview-v7:$android.supportVersion",
                      "support-v4"                   : "com.android.support:support-v4:$android.supportVersion",
                      "design"                       : "com.android.support:design:$android.supportVersion",
                      "cardview-v7"                  : "com.android.support:cardview-v7:$android.supportVersion",
                      "rxjava"                       : "io.reactivex.rxjava2:rxjava:$android.rxjava2Version",
                      "rxandroid"                    : "io.reactivex.rxjava2:rxandroid:$android.rxandroidVersion",
                      "retrofit"                     : "com.squareup.retrofit2:retrofit:$android.retrofitVersion",
                      "adapter-rxjava2"              : "com.squareup.retrofit2:adapter-rxjava2:$android.retrofitVersion",
                      "converter-gson"               : "com.squareup.retrofit2:converter-gson:$android.retrofitVersion",
                      "butterknife"                  : "com.jakewharton:butterknife:$android.butterknifeVersion",
                      "butterknife-compiler"         : "com.jakewharton:butterknife-compiler:$android.butterknifeVersion",
                      "gson"                         : "com.google.code.gson:gson:$android.gsonVersion",
                      "glide"                        : "com.github.bumptech.glide:glide:$android.glideVersion",
                      "BaseRecyclerViewAdapterHelper": "com.github.CymChad:BaseRecyclerViewAdapterHelper:$android.recyclerAdapterHelper",
                      "logging-interceptor"          : "com.squareup.okhttp3:logging-interceptor:3.8.1",
                      "room-rxjava2"                 : "android.arch.persistence.room:rxjava2:1.0.0",
                      "cardstack"                    : "com.daprlabs.aaron:cardstack:0.3.1-beta0",
                      "MZBannerView"                 : "com.github.pinguo-zhouwei:MZBannerView:v2.0.0",
                      "constraint-layout"            : "com.android.support.constraint:constraint-layout:1.0.2",
                      "circleimageview"              : "de.hdodenhof:circleimageview:2.2.0",
                      "room-runtime"                 : "android.arch.persistence.room:runtime:$android.roomVersion",
                      "room-compiler"                : "android.arch.persistence.room:compiler:$android.roomVersion",
                      "timber"                       : "com.jakewharton.timber:timber:$android.timberVersion",
                      "matisse"                      : "com.zhihu.android:matisse:$android.matisseVersion",
                      "arouter-api"                  : "com.alibaba:arouter-api:1.3.1",
                      "arouter-compiler"             : "com.alibaba:arouter-compiler:1.1.4",

              ]

  }
  ```

- 在根项目的``build.gradle`` 文件中引入以上配置文件

  ```groovy
  apply from: "config.gradle"
  ```

- 然后再module的``build.gradle`` 添加如下配置

  **配置开关** 

  ```
  if (isModule) {
      apply plugin: 'com.android.library'
      apply plugin: 'com.jakewharton.butterknife'
  }else {
      apply plugin: 'com.android.application'
  }
  ```

  用来动态切换,``butterknife`` 插件是用来生成``R2`` 文件的,以便能在``module`` 当中使用``butterknife`` ,但是这里有个问题,当处于集成模式时需要使用``R2.id.xxx`` ,而在独立开发时需要使用``R.id.xxx`` ,这个切换工作完全是体力活,所以一般组件化不推荐使用``butterknife`` ,还是老老实实``findViewById`` 吧(虽然写起来不够优雅).

  **定义全局变量** 

  ```groovy
  def cfg = rootProject.ext.android
  def appId = rootProject.ext.appId
  def deps = rootProject.ext.dependencies
  ```

  主要是为了之后书写更加优雅简洁

  **配置``ApplicationId`` ,如果是单独运行模式** 

  ```groovy
    if (!isModule) {
              applicationId appId.home
    }
  ```

  **配置``java`` 资源目录和``AndroidManifest`` **

  ```
  sourceSets{
      main{
          //在组件模式下 使用不同的manifest文件
          if(!isModule){
              manifest.srcFile 'src/main/module/AndroidManifest.xml'
              java.srcDirs 'src/main/module/java','src/main/java'
          }else{
              manifest.srcFile 'src/main/AndroidManifest.xml'
          }
      }
  }
  ```

  因为你想要单独运行必须要有启动页和配置了启动页的``AndroidManifest`` 文件

  **添加基础模块和路由模块的依赖** 

  ```groovy
  dependencies {
      implementation fileTree(dir: 'libs', include: ['*.jar'])

      testImplementation 'junit:junit:4.12'
      androidTestImplementation 'com.android.support.test:runner:1.0.1'
      androidTestImplementation 'com.android.support.test.espresso:espresso-core:3.0.1'
      implementation deps["recyclerview-v7"]
      implementation deps["cardview-v7"]
      implementation deps["appcompat-v7"]
      implementation deps["rxjava"]
      implementation deps["BaseRecyclerViewAdapterHelper"]
      implementation deps["glide"]
      implementation deps["design"]
      implementation deps["butterknife"]
      annotationProcessor deps["butterknife-compiler"]
      implementation deps["MZBannerView"]
      implementation project(':base')
      implementation deps["arouter-api"]
      annotationProcessor deps["arouter-compiler"]
  }
  ```

