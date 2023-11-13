
---
title: ProtoBuffer 自定义 option 扩展
date: 2019-08-03 12:11:42
tags:  ProtoBuffer
categories: Program
---


相信使用过 ProtoBuffer 的同学都不陌生以下的定义, 一个 Kid ProtoBuffer 对象，有一个 `age` 字段，且默认值是7。

```
message Kid
{
   optional uint32 age = 1 [default=7];
   optional uint32 score = 2 ;
}
```

运用 PB 的 默认 option 扩展，我们可以指定一些行为，比如上面的默认初始值。

我们想实现自定义的选项，比如，我想指定一种`上报`的 option, 对于此类 option, 我就将这种字段上报给老师。那么定义的 ProtoBuffer 大概会长这样:

```
message Kid
{
   optional uint32 age = 1 [default=7];
   optional uint32 score = 2 [action = Report];
}
```

是否可行呢？幸运的是 PB 有自定义 option 的机制，使得我们的设想变成可能。


#### 如何实现自定义扩展

1:  注册 Fieldoption , `action.proto` 就是定义好了 option 的 Proto 文件

```
package action;
import "google/protobuf/descriptor.proto";

enum ActionType {
    REPORT = 1;
}
extend google.protobuf.Fieldoptions {
    optional ActionType rule = 12345 [default = REPORT];
}
```

2： 应用 option
import 定义的 Proto 文件, 就可以轻易用上了。

```
package Kid                                                                          
import "action.proto";                                                                    
message person 
{
   optional uint32 age = 1;
   optional int32 age = 2 [(action.rule) = REPORT];
}
```


#### 怎么通过反射获取 option 的值呢

通过 descriptor && reflection 我们可以得到 option 的值 

```
#include <iostream>
#include <google/protobuf/message.h>
#include <google/protobuf/descriptor.h>
#include "kid.pb.h"
using namespace std;

int main()
{
    Kid obj;
    obj.set_age(12);

    auto descriptor = obj.GetDescriptor();
    auto reflection = obj.GetReflection();
	// 通过 FieldName 获取字段
    auto  idField = descriptor->FindFieldByName("age");
    reflection->SetInt32(&obj, idField, 18);
    for (unsigned int i = 0; i < descriptor->field_count(); ++i) 
    {
        auto field = descriptor->field(i);

        cout << "field_is_repeated:" << field->is_repeated() << endl;
        // 获取 field_num
        cout << "field_num:" << field->number() << endl;
        // 获取 option cout << "optiona value:" << field->options().GetExtension(action::rule) << endl;

        switch (field->type()) 
        {
            case ::google::protobuf::FieldDescriptor::TYPE_INT32:
                cout << "int type, value:" << reflection->GetInt32(obj, field);
                break;
            case ::google::protobuf::FieldDescriptor::TYPE_FLOAT:
                cout << "float type, value:" << reflection->GetFloat(obj, field);
                break;
            default:
                cout << "not found" << endl;
        }
    }
    return 0;
}
```


以上的例子，通过反射我们也可以在不知道字段的名字情况下，得到它的值和类型等信息。


