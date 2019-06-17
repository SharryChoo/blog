---
layout: post
title: "Android 图片压缩框架 —— SCompressor"
date: 2019-06-17
categories: Android
tags: Framework
img: https://i.loli.net/2019/06/17/5d07593ad56ab66651.jpg
describe: 如何使用 SCompressor?
---
---
## 一. [关于](https://github.com/SharryChoo/SCompressor)
一款关于 Android 端图片压缩解决方案的框架 (Core is [libjpeg-turbo 2.0.2](https://github.com/libjpeg-turbo/libjpeg-turbo/releases/tag/2.0.2))

---
## 二. 最新版本 
[![](https://jitpack.io/v/SharryChoo/SCompressor.svg)](https://jitpack.io/#SharryChoo/SCompressor)

---
## 三. 如何集成
### Step 1
在工程的根 build.gradle 中添加 jitpack 的 maven 仓库
```
allprojects {
    repositories {
    	...
	    maven { url 'https://jitpack.io' }
    }
}
```

### Step 2
在工程的 base 库中的 build.gradle 添加如下依赖
```
dependencies {
    ...
    implementation 'com.github.SharryChoo:SCompressor:x.x.x'
}
```

---
## 四. 如何使用
### 一) 初始化
在 Application 创建时, 传入 Context 进行初始化操作
```
SCompressor.init(this);
```

### 二) 配置输入源
```
// 1. 支持压缩文件路径
SCompressor.create()
       // set file path.
       .setInputPath(inputPath)
       ......
});

// 2. 支持压缩 Bitmap
SCompressor.create()
       // set origin bitmap.
       .setInputBitmap(originBitmap)
       ......
});

// 3. 支持自定义数据源
SCompressor.create()
       // custom input source
       .setInputSource(new DataSource<Object>() {
           @NonNull
           @Override
           public Class<Object> getType() {
                return null;
           }

           @Nullable
           @Override
           public Object getSource() {
               return null;
           }
       })
       ......
});
```
上面三种方式, 使用文件路径能够获得最好的压缩速度

如果使用了自定义数据源, 则需要在后面实现对应的 InputAdapter, 并且添加到 SCompressor 中

### 三) 配置压缩项
```
SCompressor.create()
        .setInputPath(url)
        // 0 ~ 100
        .setQuality(70)
        // 设置期望压缩尺寸
        .setDesireSize(500, 1000)
        // 默认值为 true: 在没有设置期望尺寸的情况下, 进行自适应降采样
        .setAutoDownSample(true)
        ......
```

### 四) 配置输出数据源
```
// 1. 配置输出路径
SCompressor.create()
        .setInputPath(inputPath)
        .setQuality(70)
        .setOutputPath(outputPath)
        ......
        
// 2. 指定输出为 bitmap
SCompressor.create()
        .setInputPath(inputPath)
        .setQuality(70)
        .asBitmap()
        ......
        
// 3. 指定输出 byte[]
SCompressor.create()
        .setInputPath(inputPath)
        .setQuality(70)
        .asByteArray()
        ......
        
// 3. 自定义输出源
SCompressor.create()
        .setInputPath(inputPath)
        .setQuality(70)
        // custom output source
        .setOutputSource(new DataSource<Object>() {
           @NonNull
           @Override
           public Class<Object> getType() {
                return null;
           }

           @Nullable
           @Override
           public Object getSource() {
               return null;
           }
       })
        ......        
```
没有设置输出源, 则默认使用文件路径输出

如果使用了自定义输出源, 则需要在后面实现对应的 OutputAdapter, 并且添加到 SCompressor 中

### 五) 异步发起
```
// normal async call.
SCompressor.create()
        .setInputPath(inputPath)
        .setQuality(70)
        .asBitmap()
        .asyncCall(new CompressCallback<Bitmap>() {
            @Override
            public void onCompressSuccess(@NonNull Bitmap compressedData) {
                // ......
            }

            @Override
            public void onCompressFailed(@NonNull Throwable e) {
                // ......
            }
        });
        
// lambda async call.
SCompressor.create()
        .setInputPath(inputPath)
        .setQuality(70)
        .asBitmap()
        .asyncCall(new CompressCallbackLambda<Bitmap>() {
            @Override
            public void onCompressComplete(boolean isSuccess, @Nullable Bitmap compressedData) {
                 if (isSuccess) {
                      ......// if isSuccess, the compressedData non null.
                 }
            }
        });
```

### 六) 同步发起
```
Bitmap bitmap = SCompressor.create()
        .setInputPath(inputPath)
        .setQuality(70)
        .asBitmap()
        .syncCall()
```

### 七) 其他 
如果使用了自定义输入输出源, 则需要实现对应的 Adapter, 并且添加到 SCompressor

#### 1. InputAdapter
```
// Define Input Adapter
InputAdapter<Object> myInputAdapter = new InputAdapter<Object>() {
     @Override
     public String adapt(@NonNull Request request, @NonNull Object inputData) throws Throwable {
         // Request: u can fetch everything from request.
         // inputData: u need adapt this data 2 input file.
         return null;
     }

     @Override
     public boolean isAdapter(@NonNull Class adaptedType) {
          // Ensure this Adapter field.
          return Object.class.getName().equals(adaptedType.getName());
     }
};

// Add to scompressor.
SCompressor.addInputAdapter(myInputAdapter);
```

#### 2. OutputAdapter
```
OutputAdapter<Object> myOutputAdapter = new OutputAdapter<Object>() {
     @Override
     public Object adapt(@NonNull File compressedFile) {
         // compressedFile: this file is compressed output file, u need adapt it to u desire obj.
         return null;
     }

     @Override
     public boolean isAdapter(@NonNull Class adaptedType) {
          // Ensure this Adapter field.
          return Object.class.getName().equals(adaptedType.getName());
     }
};

// Add to scompressor.
SCompressor.addOutputAdapter(myOutputAdapter);
```
---

## License
```
Copyright 2019 drakeet.

Licensed under the Apache License, Version 2.0 (the "License");
you may not use this file except in compliance with the License.
You may obtain a copy of the License at

   http://www.apache.org/licenses/LICENSE-2.0

Unless required by applicable law or agreed to in writing, software
distributed under the License is distributed on an "AS IS" BASIS,
WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
See the License for the specific language governing permissions and
limitations under the License.
```
