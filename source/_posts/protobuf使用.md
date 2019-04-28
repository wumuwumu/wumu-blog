---
title: protobuf使用
date: 2019-04-10 10:31:04
tags:
- go
---

# 安装

```
wget https://github.com/protocolbuffers/protobuf/releases/download/v3.6.1/protobuf-all-3.6.1.zip
unzop protobuf-all-3.6.1.zip
cd protobuf-all-3.6.1
./configure && make && make install
```

# 语法规则

```
// 声明版本，默认是proto2
syntax = "proto3";

// 声明包名
package tutorial
option java_package = "com.example.tutorial";
// java类名
option java_outer_classname = "AddressBookProtos";

message Person {
    required string name =1;
    required int32 id = 2;
    optional string email = 3;
    
    enum PhoneType {
        MOBILE = 0;
        HOME = 1;
        WORK = 2;
    }
    
    message PhoneNumber {
        required string number = 1;
        optional PhoneType type = 2[default = HOME]; 
    }
    repeated PhoneNumber phones = 4;
}

message AddressBook {
    repreated Person people = 1;
}

// 保留字段，编程过程中某些功能没有想好，可以先把该tag 进行保留，以备以后使用。
message Foo {
  reserved 2, 15, 9 to 11;
  reserved "foo", "bar";
}
```

# 编码

> https://blog.csdn.net/zxhoo/article/details/53228303

# 方法

1. Standard Message Methods

- `isInitialized()`: checks if all the required fields have been set.
- `toString()`: returns a human-readable representation of the message, particularly useful for debugging.
- `mergeFrom(Message other)`: (builder only) merges the contents of `other` into this message, overwriting singular scalar fields, merging composite fields, and concatenating repeated fields.
- `clear()`: (builder only) clears all the fields back to the empty state.

1. Parsing and Serialization

- `byte[] toByteArray();`: serializes the message and returns a byte array containing its raw bytes.
- `static Person parseFrom(byte[] data);`: parses a message from the given byte array.
- `void writeTo(OutputStream output);`: serializes the message and writes it to an `OutputStream`.
- `static Person parseFrom(InputStream input);`: reads and parses a message from an `InputStream`.

# 编译

# 注意

1. 升级协议

- you *must not* change the tag numbers of any existing fields.
- you *must not* add or delete any required fields.
- you *may* delete optional or repeated fields.
- you *may* add new optional or repeated fields but you must use fresh tag numbers (i.e. tag numbers that were never used in this protocol buffer, not even by deleted fields).

1. protobuf对repeated压缩不够好，所以尽量在后面加上[packed = true]。
2. 不要让protobuf对象成为全局变量或者类成员，因为其clear方法只会把占用的内存空间清零，而不会释放，使得进程空间越来越大，可参考[《Protobuf使用不当导致的程序内存上涨问题》](http://www.kuqin.com/shuoit/20141117/343247.html)。

> https://www.jianshu.com/p/27fdf44dd63b