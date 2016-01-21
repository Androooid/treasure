title: 携程动态加载实践DynamicAPK项目学习
date: 2016-01-18 22:40:33
tags: 技术流
---

---
{:toc}

## 术语

**宿主**：Host Apk，打包运行的总项目，用于集成各个子工程，相当于`android-nova壳`，只有一个

**插件**：Plugin，子工程，由各个业务线独立开发，可以依赖**宿主**提供的资源，一个宿主能够拥有多个插件

---

## Andoroid Build 流程

![][android-build-2.png]

- Resource Files：项目`res`路径下的文件，包含`anim`、`drawable`、`layout`、`raw`等
- Source Files：项目`src`路径下(对于`android-nova`是`src/main/java`)的文件
- aapt: 将`.xml`文件转换为java类型
- Generated Source Files：由`aapt`工具生成，存储在`/gen`目录
- javac：java编译器，生成`.class`文件
- dx：将`.class`文件转换为DVM可识别的`dex`文件
- apkbuilder：将所有文件打包进`apk`中
- zipalign：将`apk`中未经压缩过的文件进行4字节对齐，以减少运行时`RAM`消耗

---

## 协同开发

#### 与`引用aar提供的公用UI组件`之间的联系与区别

- 本质上都是`classes.dex`与`res/`等文件的集合
- 依赖关系不同
- 是否包含业务逻辑
- 最重要的一点，`Activity`注册

#### 插件引用宿主资源

- 在`java`代码中直接用

``` java
public class MainActivity extends Activity { // demo2

    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.demo2_activity_main);
        TextView textView=(TextView)findViewById(R.id.demo2_textView3);
        textView.setText(R.string.sample_text); // 宿主资源
        ImageView imageView=(ImageView)findViewById(R.id.demo2_imageView2);
        imageView.setImageResource(R.drawable.sample); // 宿主资源
    }

}
```

- 在布局文件中引用宿主资源，编译时会报错

``` bash
/Users/leili/Documents/idea_workspace/BeQuietLee/DynamicAPK/demo2/res/layout/demo2_activity_main.xml:40: error: 
Error: No resource found that matches the given name (at 'src' with value '@drawable/sample').
```

#### 宿主调起插件

- 宿主`AndroidMainifest.xml`需要对所有插件`Activity`进行注册

``` xml
<?xml version="1.0" encoding="utf-8"?>
<manifest xmlns:android="http://schemas.android.com/apk/res/android"
    package="ctrip.android.sample" >

    <application
        android:name="ctrip.android.sample.BundleBaseApplication"
        android:allowBackup="true"
        android:icon="@mipmap/ic_launcher"
        android:label="@string/app_name"
         >
        <activity
            android:name=".MainActivity"
            android:label="@string/app_name" >
            <intent-filter>
                <action android:name="android.intent.action.MAIN" />

                <category android:name="android.intent.category.LAUNCHER" />
            </intent-filter>
        </activity>

        <activity
            android:name="ctrip.android.demo1.MainActivity"
             >

        </activity>
        <activity
            android:name="ctrip.android.demo2.MainActivity"
             >

        </activity>
    </application>

</manifest>
```

- startActivity时需要写明类的完整路径

``` java
startActivity(new Intent(getApplicationContext(), Class.forName("ctrip.android.demo1.MainActivity")));
```

#### 对`aapt`工具的改造

为`aapt`增加`--apk-module`参数

>如前所述，资源ID其实有一个PackageID的内部字段。我们为每个插件工程指定独特的PackageID字段，这样根据资源ID就很容易判明，此资源需要从哪个插件apk中去查找并加载了。在后文的资源加载部分会有进一步阐述。

通常项目中生成的`R.java`，会包含由`aapt`生成的所有资源的`id`，以`0x7f`开头

```java
package com.dianping.hui;

public final class R {
    public static final class anim {
        public static int activity_exit=0x7f040000;
        public static int back_enter=0x7f040001;
        public static int back_exit=0x7f040002;
        public static int booking_push_up_out=0x7f040003;
        public static int fade_light_out=0x7f040004;
        public static int gradient_enter=0x7f040005;
        public static int grow_from_bottom=0x7f040006;
        // ...
    }
    public static final class color {
        public static int actionbar_title_color=0x7f0b0000;
        // ...
    }
    public static final class dimen {
        public static int action_button_height=0x7f0700a6;
        public static int action_button_margin_between=0x7f0700a7;
        // ...
    }

    // ...

}
```

`0x7f`是怎么来的呢，看一下原汁原味的`aapt`中的逻辑

```cpp
ResourceTable::ResourceTable(Bundle* bundle, const String16& assetsPackage, ResourceTable::PackageType type)
ssize_t packageId = -1;
switch (mPackageType) {
    case App:
    case AppFeature:
        packageId = 0x7f;
        break;

    case System:
        packageId = 0x01;
        break;

    case SharedLibrary:
        packageId = 0x00;
        break;

    default:
        assert(0);
        break;
}
```

DynamicAPK中改动后的生成`packageId`的逻辑，位于`ResourceTable.cpp`

```
ResourceTable::ResourceTable(Bundle* bundle, const String16& assetsPackage, ResourceTable::PackageType type)
    : mAssetsPackage(assetsPackage)
    , mPackageType(type)
    , mTypeIdOffset(0)
    , mNumLocal(0)
    , mBundle(bundle)
{
   
    
    ssize_t packageId = -1;
    
    switch (mPackageType) {
        case App:
        case AppFeature:
            packageId = 0x7f;
            break;
        case System:
            packageId = 0x01;
            break;
        case SharedLibrary:
            packageId = 0x00;
            break;
            
//        case Voice:
//            packageId = 0x34;
//            break;
//        case Call:
//            packageId = 0x35;
//            break;
//        case Search:
//            packageId = 0x36;
//            break;
//        case Schedule:
//            packageId = 0x37;
//            break;
//        case Train:
//            packageId = 0x38;
//            break;
//        case Destination:
//            packageId = 0x44;
//            break;
//        case Chat:
//            packageId = 0x46;
//            break;
//        case Flight:
//            packageId = 0x52;
//            break;
//        case MyCtrip:
//            packageId = 0x54;
//            break;
//        case Pay:
//            packageId = 0x55;
//            break;
//        case Foundation:
//            packageId = 0x56;
//            break;
//        case Hotel:
//            packageId = 0x58;
//            break;
//        case Container:
//            packageId = 0x61;
//            break;
//        case CustomerService:
//            packageId = 0x62;
//            break;
//        case ThirdParty:
//            packageId = 0x63;
//            break;
//        case Extend1:
//            packageId = 0x64;
//            break;
//        case Extend2:
//            packageId = 0x65;
//            break;
//        case Extend3:
//            packageId = 0x66;
//            break;
//        case Extend4:
//            packageId = 0x67;
//            break;
//        case Extend5:
//            packageId = 0x68;
//            break;
//        case Extend6:
//            packageId = 0x69;
//            break;
        default:
            assert(0);
            break;
    }
    if(!bundle->getApkModule().isEmpty()){
        android::String8 apkmoduleVal=bundle->getApkModule();
        packageId=apkStringToInt(apkmoduleVal);
    }
    
    sp<Package> package = new Package(mAssetsPackage, packageId);
    mPackages.add(assetsPackage, package);
    mOrderedPackages.add(package);

    // Every resource table always has one first entry, the bag attributes.
    const SourcePos unknown(String8("????"), 0);
    getType(mAssetsPackage, String16("attr"), unknown);
}
```

如此一来，不同业务的资源被赋予了不同的id，在加载时，便进入到各个业务打出的插件包里寻找资源.


在`sub-project-build.gradle`中，组装`--apk-module`参数

```bash
...
argv << '--apk-module'
argv << "$resourceId"
...
```

## 动态加载

运行时加载资源，需要知道资源从哪个插件中获取。所有插件位于压缩包的`assets/baseres/`路径下。在运行时会在`/data/data/ctrip.android.sample/files/storage/{num}`生成对应的文件夹。

![][ctrip_folders]

- `1`、`2`分别对应了`demo1`、`demo2`
- `meta`记录了下一个可用id，即`3`
- `1/meta`中保存包名`ctrip.android.demo1`
- `1/version_1/bundle.zip`就是`assets/baseres/ctrip_android_demo1.so`，可以解压缩观察
- `1/version_1/bundle.dex`其实与`bundle.zip`解压缩拿到的`dex`是一样的
- `1/version_1/meta`无用（内容为`file:`）

这部分目录与文件的管理，是通过`Bundle`、`Archive`接口完成的，`version_1`、`version_2`...`version_n`分别对应一个`BundleArchiveRevision`，它们统一由`BundleAchive`管理。`BundleArchive`由`BundleImpl`管理。图中有`demo1`、`demo2`，对应着就有2个`BundleImpl`。各个`Bundle`的启动，更新，卸载，由`Framework.java`管理。

对应关系如下图所示：

![][ctrip_bundle]

## 热修复

`DynamicAPK`中，仅仅提供了分包、资源加载的demo，未包含hotfix功能。虽然可以看到`HotPatchManager.java`，但并没有真正运行。这里我们自己来山寨一次hotfix的过程。

首先了解一下apk安装以及运行时会操作的几个目录。下载好的`apk`在进行安装时，会对系统中以下几个目录进行操作

- `/data/app`：apk安装时会被复制到该目录
- `/data/dalvik-cache`：安装dex文件的真正位置，后续app启动均从此处进行load
- `/data/data`：新建以`packageName`命名的文件夹，只有app自己能访问，用于管理数据

在没有root过的设备上，应用程序有权限操作的目录仅仅是`/data/data/com.foo.foo`，想更改`/data/dalvik-cache`中的`dex`文件是不可能的。但是，结合上面完成的动态加载工作，我们就可以在运行时更新`demo1`、`demo2`的dex文件，从而达到热修复的目的。

这里我们演示一个更换文案的demo，将`demo2`里面`textView`预设的文案由`This is sample  resource:`换成`下图来自于宿主资源`。首先需要把改好的项目进行编译，取出`bundle.dex`与`bundle.zip`（这两个文件的来源，前文已经提过）。然后把它们上传到`/data/data/com.ctrip.sample/files/storage/2/version_2`。上传完成后，下次启动就会加载我们修改后的dex文件，展示新的内容。

这里直接用**monitor**来模拟hotfix文件下载过程。

```java
public class MainActivity extends Activity {

    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.demo2_activity_main);
        TextView textView=(TextView)findViewById(R.id.demo2_textView3);
//       textView.setText(R.string.sample_text); // 宿主资源
        textView.setText("下图来自于宿主资源");
        ImageView imageView=(ImageView)findViewById(R.id.demo2_imageView2);
        imageView.setImageResource(R.drawable.sample); // 宿主资源
    }

}
```

hotfix之前

![][hotfix_before]

hotfix之后

![][hotfix_after]

---

#### 附一：更完整的编译流程图

![][android_build_process.png]

---

#### 附二：推荐阅读

- [Android动态加载技术 系列索引](http://segmentfault.com/a/1190000004086213)

---

[android_application_build_process_diagram.jpg]: ../img/160118_dynamic_apk/android_application_build_process_diagram.jpg
[android_build_process.png]: ../img/160118_dynamic_apk/android_build_process.png
[android-build-2.png]: ../img/160118_dynamic_apk/android-build-2.png
[ctrip_folders]: ../img/160118_dynamic_apk/ctrip_folders.png
[ctrip_bundle]: ../img/160118_dynamic_apk/ctrip_bundle.jpg
[hotfix_before]: ../img/160118_dynamic_apk/hotfix_before.png
[hotfix_after]: ../img/160118_dynamic_apk/hotfix_after.png
