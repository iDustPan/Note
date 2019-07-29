这篇文章是讲如何把protobuf文件的编译工作集成到Xcode中，达到在Xcode中就像添加一般的OC文件一样不进行任何多余的操作直接编译运行.proto文件的目的。是的，就是这么智能！

笔者的公司现在前后端都在统一使用一套protobuf数据结构，效率很高，非常值得推荐。并且Xcode 10进行了一些小优化来增加了对Protobuf的支持，相信不久以后，Xcode对Protobuf的支持将更加智能！

如果你想了解更多的Protobuf实战应用，可以朝我[扔简历]([https://www.lagou.com/jobs/6098585.html?show=43d138b665b8451cac981d8a874394e6](https://www.lagou.com/jobs/6098585.html?show=43d138b665b8451cac981d8a874394e6)
)哦，我司iOS现在缺人，欢迎大神投递，进公司一起研究！～

至于什么是Protobuf和Protobuf语法教程，不是这篇文章的主题，请自行Google。

环境：Xcode 10+
语言：Objective-C

话不多说，正题开始：

首先，真正的企业级项目，并不只是网上很多教程里面演示的一两个.proto文件，而是一批.proto文件目录的集合，并且是多端共享的。你会发现按照那些教程里面的讲的去做写个demo或许可以，但是真正要达到企业级别的使用的时候，还远远不够，你会遇到各种各样的坑。别问我是怎么知道的，哎～～～

## 安装编译工具

首先，要能编译Protobuf文件，我们得安装官方的编译器。你可以选择下面任意一种你喜欢的安装方式：

1. 源码编译安装；https://github.com/protocolbuffers/protobuf/tree/master/objectivec
2. 直接下载编译好的对应语言版本的二进制文件；https://github.com/protocolbuffers/protobuf/releases
3. 使用brew；`brew install protobuf`;

安装好后，在terminal中输入`which protoc`检测是否安装成功,如安装成功会返回文件路径: `/usr/local/bin/protoc`

如有问题，请自行google，不在本教程范围内。

## 在xcode项目中集成Protobuf库

没什么好说的，新建一个Xcode工程。使用Cocoapods引入Protobuf的库：

`Pod search Protobuf`

选择最稳定的版本即可。

> 坑点一：到这里，需要注意的是编译器和Pod引入的Protobuf Framework的版本需要对应。比如你的编译工具是3.9.0版本，那么Protobuf版本最好也是3.9.0。如果后期升级Pod的Protobuf库，那么编译工具也需要跟随升级。版本不一致，可能会导致项目在运行时出现编译出错哦！

## 在Xcode中创建.proto文件

1. 在新工程中创建一个Protos目录；

真实的企业级项目，并不会像网上很多教程里一样只是单纯的一两个.proto文件。而是根据使用模块的划分，会有不同的文件夹，甚至整个存放.proto文件的根目录会作为git submodule来存放到远端达到多端共享的目的。Proto源文件的目录层级，对编译结果有很大的影响，直接关系到在Xcode中的使用，这是最大的坑点，我们稍后再讲；

2. 在该Protos根目录下再新建两个子目录，代表实际项目中不同的模块。为方便记忆一个为a目录，一个为b目录；

3. 在a目录下创建A.proto源文件。在b目录下创建B.proto文件；

这里有两种创建.proto文件的方式：
1) 通过命令行创建的话，创建好之后需要拖到Xcode项目下；
2) 直接在Xcode中通过右键A目录，选择`New File`，然后依次选择`iOS --> Other --> Empty`, 文件名加上.proto后缀即可。

> 坑点二：.proto的文件名格式一定是大驼峰写法。即一定要以大写字母开头。因为即使文件名全是小写，最终编译出来的是结果也是大驼峰格式命名的文件。比如test.proto编译出来的是Test.pbobjc.h和Test.pbobjc.m文件；

至于文件内容，如果你熟悉protobuf语法，那随便写几行即可，如果不熟悉，那么可以copy我的测试内容：

A.proto文件内容：

```
syntax = "proto3";

import "b/b.proto"; // 在A.proto文件中引入b/b.proto文件，一定要指明路径哦～

option objc_class_prefix = "PXL";

package a; 

message TestA {
    string name = 1;
    b.TestB test = 2;
}
```

B.proto文件内容：

```
syntax = "proto3";

option objc_class_prefix = "PXL";

package b;

message TestB {
    string name = 1;
}

```

> 坑点三：注意，无论以上面哪种方式创建。在Xcode10以前的版本，创建好文件后，需要到`Project --> Build Phases --> Compile Sources` 中，把刚才新建的a.proto和b.proto文件添加进去。什么意思呢？就是说要把这两个文件添加到可编译文件里面。只有可编译文件，我们才能对其进行后续的自定义编译；Xcode10不用，Xcode10已经针对Protobuf进行了一些专门的优化。

## 为工程添加自定义编译脚本

Xcode自己并不认识proto文件，所以并不会为自动编译.proto文件，我们需要把.proto编译器自己集成到项目当中，集成的方式如下：

1. 依次进入到以下目录：

`Project --> Build Rules --> 点击+号`，生成一个特定文件类型编译脚本。

2. 在`Process`中选择`Protobuf source files`；(注意，如果是Xcode10之前的版本并没有这个选项，你需要选择`Source files with names matching`, 然后在后面的输入框中输入`*.proto`);

3. 按照[官方教程](https://developers.google.com/protocol-buffers/docs/reference/objective-c-generated)，添加编译脚本：

```
/usr/local/bin/protoc --proto_path=${SRCROOT}/<你的工程目录名称>/protos/ --objc_out=${DERIVED_FILE_DIR} $INPUT_FILE_PATH 

```

比如：

```
/usr/local/bin/protoc --proto_path=${SRCROOT}/ProtoTests/protos/ --objc_out=${DERIVED_FILE_DIR} $INPUT_FILE_PATH
```

到此处，我们有几个注意事项：

 1. protoc命令尽量指明绝对路径，以防脚本编译时找不到命令的情况。即`/usr/local/bin/protoc` 而不是`protoc`。 该点官方文档倒是没提到，是我们自己遇到的一个坑；

 2. 这里需要用到几个环境变量：
    ${SRCROOT} 是Xcode自带环境变量，代表工程根目录; 

    ${INPUT_FILE_PATH} 代表脚本执行文件的绝对输入路径，包含文件名本身，并且带文件格式；

    ${INPUT_FILE_BASE} 代表脚本执行文件的文件名，不包含后缀格式；

    ${INPUT_FILE_NAME} 代表脚本执行文件的文件名，包含后缀格式；

    ${DERIVED_FILE_DIR} 代表Xcode的文件输出目录；

    其他Xcode自带环境变量https://gist.github.com/gdavis/6670468。当然，你也可以在项目*build log* 中查看。

 3. 如文档所言，`--proto_path`对应的路径是proto源文件的*绝对根目录*。`--objc_out`是编译产生文件的存放目录。


### 为什么`--proto_path` 需要是绝对根目录呢？

 我们试试把 `--proto_path` 换成相对路径，看会发生什么，也就是把脚本换成

 ```
 cd ${SRCROOT}/ProtoTests/protos/
 /usr/local/bin/protoc --proto_path=./ --objc_out=${DERIVED_FILE_DIR} $INPUT_FILE_PATH
 ```

编译运行，咦～报错了。查看日志，我们可以看到这么一条log信息：

```
File does not reside within any path specified using --proto_path (or -I).  You must specify a --proto_path which encompasses this file.  Note that the proto_path must be an exact prefix of the .proto file names -- protoc is too dumb to figure out when two paths (e.g. absolute and relative) are equivalent (it's harder than you think).
```

翻译过来就是在--proto_path这个参数中你必须指定.proto源文件的精确路径，`protoc`太笨了，它无法搞清楚这个相对路径是不是我们要的绝对路径。google的工程师说这太他么难了。所以这里很明确了，`--proto_path` 的参数值，只能是proto文件根目录的绝对路径。


 ### 那我们为什么要用`$INPUT_FILE_PATH`? 

 我们上面说了，${INPUT_FILE_PATH} 是代表编译输入源文件的绝对路径。

文档里面给的demo是: 
`protoc --proto_path=src --objc_out=build/gen src/foo.proto src/bar/baz.proto`

什么意思呢？

它说，最终编译器会把`src/foo.proto`文件编译成：`build/gen/Foo.pbobjc.h` 和 `build/gen/Foo.pbobjc.m` 文件。
而会把 `src/bar/baz.proto` 文件编译成 `build/gen/bar/Baz.pbobjc.h` 和 `build/gen/bar/Baz.pbobjc.m`。
而不是`build/gen/Baz.pbobjc.h` 和  `build/gen/Baz.pbobjc.m`

也就是说protobuf编译器最终生成的文件会自动按照文件源目录结构存放。

特别强调 *并不会* 自动创建 `build/gen` 目录，这个目录需要你提前建好。

并且，查看最终编译生成的.m文件，你会发现一些有趣的事情；比如我在A.proto中引入了B.proto文件，你会看到Protobuf最终编译出来的A.pbobjc.m文件导入文件的格式是包含文件路径的，例如：

 ```
 import "a/A.pbobjc.h"
 import "b/B.pbobjc.h"
 ```


## 设置编译文件输出路径

我们注意到，上面设置的proto文件的编译输出路径是 `$DERIVED_FILE_DIR`， 这是为何呢？

答案是为了方便Xcode的集成。

*对于自定义的编译脚本，都需要设置一个文件的输出路径.*

我们点脚本框下面的Output Files下面的`+`号, 指定文件输出路径。
因为OC文件分为.h和.m文件，所以我们指定2个。

点了之后，你会发现，xcode默认给出的是 `$(DERIVED_FILE_DIR)/newOutputFile`, 
我们将其改为`$(DERIVED_FILE_DIR)/${INPUT_FILE_BASE}.pbobjc.h` 和 `$(DERIVED_FILE_DIR)/${INPUT_FILE_BASE}.pbobjc.m`，并且在.m文件的`Compiler Flags`中指定为`-fno-objc-arc`代表该.m文件采用mrc编译。

编译运行，大功告成，是不可能的！！！！

你会发现又报错了：

```
clang: error: no such file or directory: '~/Library/Developer/Xcode/DerivedData/ProtoTests-dpojqcqwplnmyzbgdvjiqjfefgky/Build/Intermediates.noindex/ProtoTests.build/Debug-iphonesimulator/ProtoTests.build/DerivedSources/A.pbobjc.m'
```

什么意思呢？ 其实就是在 `DerivedSources` 下找不到 `A.pbobjc.m` 文件。因为我们指定这个编译的输出路径在这个目录下，所以Xcode在进行OC文件的编译时会去这个目录下找，但是它找不到。为什么找不到呢？我们去这个目录下看，这个目录下确实没有 `A.pbobjc.m` 这个文件，但是确发现有 `a/A.pbobjc.m`。原因我们已经说了，protoc最终的编译文件会自动加上目录前缀。


有人可能会说，能不能把输出文件改成 `$(DERIVED_FILE_DIR)/*/${INPUT_FILE_BASE}.pbobjc.h` 呢？那我们就来试下。

编译运行

what the hell?

```
clang: error: no such file or directory: '~/Library/Developer/Xcode/DerivedData/ProtoTests-dpojqcqwplnmyzbgdvjiqjfefgky/Build/Intermediates.noindex/ProtoTests.build/Debug-iphonesimulator/ProtoTests.build/DerivedSources/*/A.pbobjc.m'
```

原来，Xcode的Output Files特别蠢，它不支持类似这种通配符写法: `$(DERIVED_FILE_DIR)/*/${INPUT_FILE_BASE}.pbobjc.h`。
也不支持传入任何的自定义变量。

只能是明确的文件路径和Xcode自带的环境变量，但是实际项目中，可能不只一层路径，有可能是文件夹下嵌套文件夹。

靠，那这怎么办呢？

实在没办法了，就在打算放弃的时候，咨询了我们的脚本大神，我们尝试了以下在脚本末尾再加了两行：

```
# cd ${DERIVED_FILE_DIR}
# find . -mindepth 2 -name ${INPUT_FILE_BASE}.pbobjc.m -o -name ${INPUT_FILE_BASE}.pbobjc.h | xargs -I{} cp "{}" .
```

是不是很机智？

什么意思呢？就是说我们cd到该目录，然后找到该文件对应生成的oc文件，将其copy一份儿到根目录。怀着求神拜佛的意志，运行了以下，Perfect，终于不再报错了，到目录中查看，也正是我们想要的，所有文件都被copy出来了。

下一步，就是正常的在项目中import和使用了。

## Use it

你以为到此就没有坑了吗？到此还有坑。有2点需要注意：

1. 当我们在import这些生成的OC文件的时候，如果你用的是Xcode的 *新编译系统*，你在import的时候应该使用 `#import <B.pbobjc.h>` ,你会发现 #import "B.pbobjc.h" 也可以，但是Xcode不会给你提示。怎么办呢？将Xcode设置为老编译系统就可以了。设置方式：`File --> Workspace Settings`,将 `New Build System` 改为 `Legacy Build System` ；悄悄地告诉你，这个设置可以解决Xcode在import其他非Protobuf编译产生的文件时也不提示的问题哦～

2. import的方式是选择 `#import "B.pbobjc.h"` 还是 `#import "b/B.pbobjc.h"` 。看你喜欢，并且要统一，不过建议采用带目录的这种方式，一来是Protobuf自己产生的文件是这样做的，二来以后xcode的输出文件目录变得更智能时，一定是会支持这种方式的。

好了，就讲到这里吧～

如果大家喜欢，有时间再讲讲怎么改改AFNetworking，能直接请求后端给的Protobuf格式的数据～





















