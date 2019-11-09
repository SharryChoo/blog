---
layout: article
title: "Android 版本适配 —— Android 10 外部存储的适配"
key: "Android 版本适配 —— Android 10 外部存储的适配" 
tags: PerformanceOptimization
aside:
  toc: true
---
## 前言
最近项目上开展了对 Android 10 的适配工作, 因为自己维护了图像处理相关的基础组件, 需要在外部存储卡进行读写, 因此笔者主要探究了 AndroidQ 的存储适配

这里做个简单的记录

## 存储变更
Android 系统的磁盘分区有如下几种
- **/system 分区**: 存放所有 Google 提供的 Android 组件, 这个分区只能够以只读的方式 mount, 即使突然断电, 也依然要保证 /system 分区的内容不受损害
  - 系统升级也主要是更新该分区
- **/data 分区**: 存放所有用户数据的地方
  - 恢复出厂设置 即格式化该分区
- **/vendor**: 存放厂商特殊系统修改的分区
- **/storage**: 外部存储卡

**Android 10 的 存储变更主要是体现在 storage 外部存储的文件系统中, Android Q 将其划分为两个部分**
- [Private files](https://developer.android.google.cn/training/data-storage/files/external#PrivateFiles)
- [Public files](https://developer.android.google.cn/training/data-storage/files/external#PublicFiles)

## Private files
通过 Context.getExternalFilesDir(xxx) 获取, 其路径如下
```
context.getExternalFilesDir(Environment.DIRECTORY_PICTURES)

/storage/emulated/0/Android/data/${package_name}/files/Pictures/{file_name}
```

### 一) 特征
- Android Q 以后无需申请外部存储卡也能够进行读写
- 应用卸载后会随之删除

### 二) 适配方案
无需特殊适配, 之前的文件 API 均可使用

## Public files
除了私有文件之外, 其他的区域即为共有文件区域, 主要分为两个部分
- 多媒体文件
- Other File

### 一) 特征
- 应用卸载之后, 仍可以保留在磁盘中
- **Android Q 之后, File API 全部失效**, 只能够通过 URI 进行读写

### 二) 适配方案
这里主要探究一下多媒体文件的读写, 以 Android Q 之前和之后为例, 看看他们的异同

#### 1. Android Q 之后
```
// 创建拍照目标文件
String fileName = "camera_" + DateFormat.format("yyyyMMdd_HH_mm_ss",
        Calendar.getInstance(Locale.CHINA)) + ".jpg";
        
// 1. 使用 ContentValues 描述这个要创建的媒体资源信息
ContentValues values = new ContentValues();
// 创建将文件生成在 "/storage/emulated/0/Pictures/${Environment.DIRECTORY_PICTURES}}/${relativePath}" 目录下
// 注意: 无法在非 Environment.xxx 的路径下创建文件 
values.put(MediaStore.Images.Media.RELATIVE_PATH, Environment.DIRECTORY_PICTURES + "/" + relativePath);
values.put(MediaStore.Images.Media.MIME_TYPE, "image/jpg");
values.put(MediaStore.Images.Media.DISPLAY_NAME, fileName);
// 表示延时发布到媒体库
values.put(MediaStore.Images.Media.IS_PENDING, 1); 

// 2. 插入一条媒体文件的数据, 获取文件 URI
ContentResolver resolver = context.getContentResolver();
Uri uri;
if (Environment.MEDIA_MOUNTED.equals(Environment.getExternalStorageState())) {
    uri =  resolver.insert(MediaStore.Images.Media.EXTERNAL_CONTENT_URI, values);
} else {
    uri = resolver.insert(MediaStore.Images.Media.INTERNAL_CONTENT_URI, values);
}

// 3.打开文件描述符进行读写
// 打开一个写端文件描述符
ParcelFileDescriptor pfd = mContext.getContentResolver().openFileDescriptor(uri, "w");
FileDescriptor fd = pfd.getFileDescriptor();
FileOutputStream fos = new FileOutputStream(fd);
......// 进行文件读写
fos.flush();
fos.close();

// 4. 将这个文件发布到媒体库
ContentValues values = new ContentValues();
values.put(MediaStore.Images.Media.IS_PENDING, 0);
context.getContentResolver().update(item, values, null, null);

// 5. 删除媒体文件
context.getContentResolver().delete(uri, null, null);
```
如上所示, 将数据写到外部存储卡共需要如下步骤
- 创建 ContentValues
- 插入一条媒体文件的数据, 获取文件 URI
- 打开文件描述符进行读写
- 发布到媒体库
  - 若不执行这步操作, 相册的更新则会有延迟

#### 2. Android Q 之前
```
// 创建文件名
String fileName = "camera_" + DateFormat.format("yyyyMMdd_HH_mm_ss",
        Calendar.getInstance(Locale.CHINA)) + ".jpg";
        
// 创建文夹目录
File dir = Environment.getExternalStoragePublicDirectory(Environment.DIRECTORY_PICTURES);
if (!dir.exists()) {
    dir.mkdirs();
}

// 创建文件
File file = new File(dir, fileName);
if (file.exists()) {
    file.delete();
}
file.createNewFile();

......//进行文件读写

// 更新到媒体库
if (Build.VERSION.SDK_INT >= Build.VERSION_CODES.KITKAT) {
    MediaScannerConnection.scanFile(context.getApplicationContext(), new String[]{filePath}, null, null);
} else {
    context.sendBroadcast(new Intent(Intent.ACTION_MEDIA_MOUNTED, Uri.parse("file://" + Environment.getExternalStorageDirectory())));
}
```
Android Q 之前可以使用标准的文件 API, 这里就不再赘述了, 不过在操作媒体文件之后(添加/删除), 同样要执行更新媒体库的操作, 否则其他用户无法检测到文件更新

## 总结
这里只探究了媒体文件的读写, 可以看到 Android Q 将数据写入到外部磁盘的难度比起之前要复杂不少
- 从 Google 视角
  - Android 资源管理器中文件的冗杂是长期的诟病了, Google 在意识到这个问题之后在 Android Q 上狠下心来对外部存储也进行了沙箱化的处理, 在开放与约束直接找到了一个更加微妙的平衡点
- 从开发者的视角
  - 增加了开发难度, 增添了适配成本
- 从用户的视角
  - 通过这样的操作, 能够带来一个更加清爽的文件管理器, 用户可以更加方便的定位到自己想要的数据, 无需在冗杂的目录下查找这真是非常令人愉悦的事情

笔者维护的 [SAlbum](https://github.com/SharryChoo/SAlbum) 现已经支持 Android 10 的版本, 可以作为参考

## 参考文献
- https://developer.android.google.cn/about/versions/10/privacy/
- https://developer.android.google.cn/training/data-storage/files/external
- https://developer.android.google.cn/training/data-storage/files/media