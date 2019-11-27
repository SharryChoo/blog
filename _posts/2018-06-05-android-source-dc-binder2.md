---
title: Android 系统架构 —— AIDL 与 Binder
permalink: android-source/dc-binder2
key: android-source-dc-binder2
tags: AndroidFramework
---

## 一. 什么是 AIDL?
AIDL 即 Android interface definition language, Android 接口定义语言

## 二. AIDL 的作用
AIDL **是 Android 提供用于快速构建支持 Binder 跨进程通信接口的一种方式**, 帮助开发者快速构建 Binder 通信模型

<!--more-->

## 三. AIDL 的组成
### 一) 支持的形参
- 支持写入 Parcel 的, AIDL 都支持

### 二) 定向 TAG
定向 TAG 用于修饰方法参数
- in: 输入型参数, **基本数据类型和 String 只能为输入型参数**
- out: 输出型参数, **实现了 Parcelable 对象可以为输出型参数**
  - 调用之后, 会为这个参数赋值 
- inout: 输入输出型参数, **实现了 Parcelable 对象可以为输出型参数**
  - 先将这个对象传入目标端, 返回时会给这个参数赋值

#### 1. AIDL 代码
```
// IMyAidlInterface.aidl
package com.sharry.scompressor;
import android.graphics.Rect;

interface IMyAidlInterface {

    void basicTypes(in String aStr, out Rect rect, inout Rect ioRect);

}
```
#### 2. 生成文件分析
```
public interface IMyAidlInterface extends android.os.IInterface {
    
    public static abstract class Stub extends android.os.Binder implements com.sharry.scompressor.IMyAidlInterface {
        
        @Override
        public boolean onTransact(int code, android.os.Parcel data, android.os.Parcel reply, int flags) throws android.os.RemoteException {
            java.lang.String descriptor = DESCRIPTOR;
            switch (code) {
                case INTERFACE_TRANSACTION: {
                    reply.writeString(descriptor);
                    return true;
                }
                case TRANSACTION_basicTypes: {
                    data.enforceInterface(descriptor);
                    // 1. 从 Parcel 中读取入参
                    // 1.1 定向 TAG 为 in, 直接读取 String arg0
                    java.lang.String _arg0;
                    _arg0 = data.readString();
                    // 1.2 定向 TAG 为 out, 客户端不会传递这个参数, 直接 new 一个对象
                    android.graphics.Rect _arg1;
                    _arg1 = new android.graphics.Rect();
                    // 1.3 定向 TAG 为 inout, 客户端可能会传递这个参数, 不为 null 时读取
                    android.graphics.Rect _arg2;
                    if ((0 != data.readInt())) {
                        _arg2 = android.graphics.Rect.CREATOR.createFromParcel(data);
                    } else {
                        _arg2 = null;
                    }
                    // 2. 调用实现方法
                    this.basicTypes(_arg0, _arg1, _arg2);
                    // 3. 写入回执参数
                    reply.writeNoException();
                    ......// 3.1 定向 TAG 为 in, 不写入回执参数
                    // 3.2 定向 TAG 为 out, 写入回执参数
                    if ((_arg1 != null)) {
                        reply.writeInt(1);
                        _arg1.writeToParcel(reply, android.os.Parcelable.PARCELABLE_WRITE_RETURN_VALUE);
                    } else {
                        reply.writeInt(0);
                    }
                    // 3.3 定向 TAG 为 inout, 写入回执参数
                    if ((_arg2 != null)) {
                        reply.writeInt(1);
                        _arg2.writeToParcel(reply, android.os.Parcelable.PARCELABLE_WRITE_RETURN_VALUE);
                    } else {
                        reply.writeInt(0);
                    }
                    return true;
                }
                default: {
                    return super.onTransact(code, data, reply, flags);
                }
            }
        }
        
        private static class Proxy implements com.sharry.scompressor.IMyAidlInterface {
            
            @Override
            public void basicTypes(java.lang.String arg0, android.graphics.Rect arg1, android.graphics.Rect arg2) throws android.os.RemoteException {
                android.os.Parcel _data = android.os.Parcel.obtain();
                android.os.Parcel _reply = android.os.Parcel.obtain();
                try {
                    // 1. 将请求参数封装到 Parcel
                    _data.writeInterfaceToken(DESCRIPTOR);// 写入接口描述符
                    // 1.1 定向 TAG 为 in, 直接写入 Parcel
                    _data.writeString(arg0);
                    ......// 1.2 定向 TAG 为 out, 不会写入 Parcel
                    // 1.3 定向 TAG 为 inout, 不为 null 时, 会写入 Parcel
                    if ((arg2 != null)) {
                        _data.writeInt(1);
                        arg2.writeToParcel(_data, 0);
                    } else {
                        _data.writeInt(0);
                    }
                    // 2. 通过 Binder 通信
                    mRemote.transact(Stub.TRANSACTION_basicTypes, _data, _reply, 0);
                    _reply.readException();
                    // 3. 读取数据
                     ...... // 3.1 定向 TAG 为 in 的参数, 什么都不做
                    // 3.2 定向 TAG 为 out 的参数, 通过 Parcelable.readFromParcel 赋值
                    if ((0 != _reply.readInt())) {
                        arg1.readFromParcel(_reply);
                    }
                    // 3.3 定向 TAG 为 inout 的参数, 通过 Parcelable.readFromParcel 赋值
                    if ((0 != _reply.readInt())) {
                        arg2.readFromParcel(_reply);
                    }
                } finally {
                    _reply.recycle();
                    _data.recycle();
                }
            }
            
        }
    }
}
```
好的, 可以看到关于定向 TAG, 他们的作用分别如下
- **in: 封装到 Parcel 发送给服务端, 通信结束时不做处理**
- **out: 不会将这个对象发送给服务端, 通信结束时通过 reply 重新赋值**
- **inout: 不为 null 时, 封装到 Parcel 发送给服务端, 返回时通过 replay 重新赋值**

## 四. AIDL 生成代码分析
```
/*
 * This file is auto-generated.  DO NOT MODIFY.
 * Original file: F:\\AndroidSpace\\Framework\\SCompressor\\app\\src\\main\\aidl\\com\\sharry\\scompressor\\IMyAidlInterface.aidl
 */
package com.sharry.scompressor;

public interface IMyAidlInterface extends android.os.IInterface {
    
    // Binder 实例对象 + 服务端接口
    public static abstract class Stub extends android.os.Binder implements com.sharry.scompressor.IMyAidlInterface {

        public Stub() {
            this.attachInterface(this, DESCRIPTOR);
        }

        // Binder 代理对象 convert2 代理接口
        public static com.sharry.scompressor.IMyAidlInterface asInterface(android.os.IBinder obj) {
           ......
        }

        @Override
        public android.os.IBinder asBinder() {
            return this;
        }

        @Override
        public boolean onTransact(int code, android.os.Parcel data, android.os.Parcel reply, int flags) throws android.os.RemoteException {
            ......
        }
        
        // 客户端代理接口
        private static class Proxy implements com.sharry.scompressor.IMyAidlInterface {
            
            // 内部持有 Binder 代理对象
            private android.os.IBinder mRemote;

            Proxy(android.os.IBinder remote) {
                mRemote = remote;
            }

            @Override
            public android.os.IBinder asBinder() {
                return mRemote;
            }
            
            @Override
            public void basicTypes(java.lang.String arg0, android.graphics.Rect arg1, android.graphics.Rect arg2) throws android.os.RemoteException {
                ......
            }
        }

    }
    
    // 由服务端实现的方法
    public void basicTypes(java.lang.String arg0, android.graphics.Rect arg1, android.graphics.Rect arg2) throws android.os.RemoteException;
    
}
```
好的, AIDL 生成类的结构如下
- **IMyAidlInterface.Stub: 既是 Binder 本地对象, 也是服务端定义的接口** 
  - 服务端需要继承这个类, 实现内部的 basicTypes 方法
- **IMyAidlInterface.Stub.Proxy: 客户端的代理方法, 内部持有 Binder 代理对象**

## 总结
AIDL 到这里就描述结束了, **其主要的作用是帮助开发者快速的构建 Binder 通信模型**, 减少 transact 和 onTransact 中参数打包解包的撰写, 帮助应用层开发者更加高效的开发
