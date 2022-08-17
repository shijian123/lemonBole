
### 关于Flutter

Flutter 诞生于 Chrome 团队的一场内部实验， 谷歌的前端团队在把前端一些“乱七八糟“的规范去掉后，发现在基准测试里性能居然提高了 20 倍，机缘巧合下 Flutter 就这么被立项

Flutter 尽管支持移动端、Web 端和 PC 端，但是作为 UI 框架，它主要帮助我们解决的是 UI 和部分业务逻辑的“跨平台”， 而和平台相关的诸如蓝牙、平台交互、数据存储、打包构建等等都离不开原生的支持。

为什么选择Flutter？

因为 Flutter 作为 UI 框架，它是真的跨平台！ 和 react-native 、 weex 不同，Flutter 的控件不是通过原生控件去实现的渲染，而是由 Flutter Engine 提供的平台无关的渲染能力，也就是 Flutter 的控件和平台没关系。简单来说，原生平台提供一个 Surface 作为画板，之后剩下的只需要由 Flutter 来渲染出对应的控件，而这个过程最终是打包成 AOT 的二进制完成。



### Flutter&iOS混编

开发过程中，我们最想要的是原生代码和Flutter共存：既不影响项目工程的原生开发，又能使用Flutter去统一iOS/Android技术栈。

![](https://upload-images.jianshu.io/upload_images/1426173-a87a962662cad323.png)

一般推荐使用三端分离方案：iOS工程、Android工程、Flutter工程是三个单独的项目工程，将Flutter工程的编译产物作为iOS工程和Android工程的依赖模块，原有工程的管理模式不变，对原生工程没有侵入性，无需额外配置工作。

这种方案需要单独创建Flutter项目，然后通过iOS（CocoaPods）和安卓的依赖管理工具将Flutter项目build出来的framework、资源包等放入Native工程以供使用。这种方式可以将iOS、Android和Flutter项目放在一个目录下面作为一个项目来管理，也可以不在同一目录下。






### 以iOS混编为例

首先按照 [官方文档](https://flutter.cn/docs/get-started/install) 配置flutter的环境

一、创建Flutter项目模块：

1、首先进入到项目目录
  /Users/zhangchaoyang/Desktop/Flutter/iOS\&Flutter/FlutterDome
 
2、利用终端命令行创建Flutter模块：(flutter_module自己命名的)

```
flutter create -t module flutter_module 

```

二、Pod引入Flutter模块：

1、如果项目中没有用到cocoaPods，先初始化pod，终端运行命令：

```
pod init

```
2、查看项目目录中会多一个 Podfile 文件，我们在该文件最后面添加命令如下：

```
platform :ios, '10.0'

flutter_application_path = 'flutter_module'
load File.join(flutter_application_path, '.ios', 'Flutter', 'podhelper.rb')
#其中flutter_application_path为flutter模块相对于podfile文件的位置。

target 'FlutterDome' do
  use_frameworks!
  
  install_all_flutter_pods(flutter_application_path)


end
```
3、运行 pod install，可以看到引入了Flutter模块

![](https://upload-images.jianshu.io/upload_images/1426173-5d0f1ec3be1f2520.png)


上面就是把Flutter引入到了iOS项目中，接下来我们简单了解下Flutter的结构

![](https://upload-images.jianshu.io/upload_images/1426173-6b7ae0559c0a63e5.png)

|	文件夹		|	作用					|
---------		|	-------------			|
|	android	|	android平台相关代码	|
|	ios			|	ios平台相关代码			|
| 	build		|	是运行flutter项目的时候生成的编译文件，即Android和iOS的构建产物|
|	lib			|	编写的flutter相关代码都放在这个文件夹里|
| 	test		|	用于存放测试代码		|
|	pubspec.lock|	记录当前项目实际依赖信息的文件|
|	pubspec.yaml|	配置文件，一般存放一些第三方的依赖。|

需注意的是：当在 `flutter_module/pubspec.yaml` 改变了 `Flutter plugin` 依赖，需要在 `Flutter module `目录运行 `flutter pub get`来更新会被 `podhelper.rb `脚本用到的` plugin `列表，然后再次在原生应用目录` FlutterDome ` 运行` pod install`进行更新.

如果`flutter pub get`超时，可以把默认的 `package` 获取地址改为访问没有问题的镜像。

Linux 或 Mac：

```
export PUB_HOSTED_URL=https://pub.flutter-io.cn
export FLUTTER_STORAGE_BASE_URL=https://storage.flutter-io.cn
```
Windows：

```
PUB_HOSTED_URL ===== https://pub.flutter-io.cn
FLUTTER_STORAGE_BASE_URL ===== https://storage.flutter-io.cn
```

### flutter 与 Native的通讯

Flutter跟Native相互通信Platform Channels，如下图所示

![](https://upload-images.jianshu.io/upload_images/7786086-07420e916c026088.png?imageMogr2/auto-orient/strip|imageView2/2/w/580)

上图中用到了MethodChannel，其实Flutter是有三种通信类型，分别是

```
* BasicMessageChannel：用于传递字符串和半结构化的信息,这个用的比较少

* MethodChannel：用于传递方法调用（method invocation）通常用来调用native中某个方法

* EventChannel: 用于数据流（event streams）的通信。有监听功能，比如电量变化之后直接推送数据给flutter端。

```
三种Channel之间互相独立，各有用途，但它们在设计上却非常相近。每种Channel均有三个重要成员变量：

```
* name: String类型，代表Channel的名字，也是其唯一标识符。

* messager：BinaryMessenger类型，代表消息信使，是消息的发送与接收的工具。

* codec: MessageCodec类型或MethodCodec类型，代表消息的编解码器。

```

channel通信的数据类型

![](https://upload-images.jianshu.io/upload_images/1975877-bcb7f6a45c5a41b4.png?imageMogr2/auto-orient/strip|imageView2/2/w/1200)



#### MethodChannel方式


下面的例子，Flutter调用原生网络请求，获取结果后返回给Flutter，其中CYHttpTool为原生封装的网络请求，网络请求失败时需注意返回的类型为FlutterError


**Flutter端**


```
import 'package:flutter/services.dart';

const platform = MethodChannel('com.flutterToNative');
String nativeBackString = 'Not Return';

 Widget build(BuildContext context) {
 ...
 Container(
              padding: EdgeInsets.all(10),
              child: Text(
                'result: ${nativeBackString}',
                style: TextStyle(
                  fontSize: 16,
                  fontWeight: FontWeight.normal,
                  color: Color(0xff333333),
                ),
              ),
            ),
 // 添加按钮
 RaisedButton(
            child: new Text('到Native网络请求'),
            onPressed: invokeNativeGetResult,
            ),
 

 }
 
 Future<void> invokeNativeGetResult() async {
    String backString;
    try {
      // 调用原生方法并传参，以及等待原生返回结果数据
      var result = await platform.invokeMethod(
          'testRequestParams', {"order": "new", "skip": "30","limit":"30"});
      backString = 'Native return: $result';
    } on PlatformException catch (err) {
      backString = "Native Failed: '${err.message}'.";
    }
    setState(() {
      nativeBackString = backString;
    });
  }


```

这里的await关键字是异步的，testRequestParams为传递的方法名字，后面的字典为传递的参数


**Native端**

```
FlutterMethodChannel *methodChannel = [FlutterMethodChannel
                                           methodChannelWithName:@"com.flutterToNative"
                                           binaryMessenger:flutterVC];
    [methodChannel setMethodCallHandler:^(FlutterMethodCall * _Nonnull call, FlutterResult  _Nonnull result) {
        //通过call.method来获取方法名称
        if ([@"testRequestParams" isEqualToString:call.method]) {
            
            [CYHttpTool get:[NSString stringWithFormat:@"%@%@", CYBASEURL, @"vertical"] params:[CYHttpTool assembleDicWithObject:call.arguments] success:^(id json) {
                if ([json[@"msg"] isEqualToString:@"success"]) {
                    result(json[@"res"][@"vertical"]);
                }else {
                    result(@"请求异常");
                }
            } failure:^(NSError *error) {
                
                FlutterError *err = [FlutterError errorWithCode:[NSString stringWithFormat:@"%ld", (long)error.code] message:error.domain details:nil];
                result(err);
            }];
        }
    }];
    
```

**Native&Flutter的互调**

上面我们说的是Flutter调用Native方法，但从最上面的图我们知道其实所有的调用都是双向的，即Native也可以调用Flutte。

例如，我们想从Native调用Flutter的getFlutterString方法获取一个字符串

**Flutter端**

```
@override
  void initState() {
    // TODO: implement initState
    super.initState();
    platform.setMethodCallHandler(platformCallHandler);
  }

Future<dynamic> platformCallHandler(MethodCall call) async {
    switch (call.method) {
      case "getFlutterString":
        return "Flutter Test flutter";
        break;
    }
  }
```

**Native端**

```
    [methodChannel invokeMethod:@"getFlutterString" arguments:nil result:^(id  _Nullable result) {
        [MBProgressHUD showText:result];
//        NSLog(@"result:%@", result);
    }];
```

#### EventChannel方式

EventChannel的使用以获取电池电量的demo为例，手机的电池状态是不停变化的。我们要把这样的电池状态变化由Native及时通过EventChannel来告诉Flutter。这种情况用之前讲的MethodChannel办法是不行的，这意味着Flutter需要不停调用getBatteryLevel来获取当前电量，显然是不正确的做法。而用EventChannel的方式，则是将当前电池状态"推送"给Flutter.

**Native端**

```
注册eventChannel
FlutterEventChannel *eventChannel = [FlutterEventChannel eventChannelWithName:@"com.phoneBatteryEvent" binaryMessenger:flutterVC];
    [eventChannel setStreamHandler:self];
    
实现eventChannel的代理
- (FlutterError *)onListenWithArguments:(id)arguments eventSink:(FlutterEventSink)events {
    if (events) {
        self.eventSink = events;
        self.eventSink([NSString stringWithFormat:@"当前电量:%.f%%", [UIDevice currentDevice].batteryLevel*100]);
    }
    return nil;
}

- (FlutterError *)onCancelWithArguments:(id)arguments {
    return nil;
}

电量发生改变时推送给flutter
[UIDevice currentDevice].batteryMonitoringEnabled = YES;
        
[[NSNotificationCenter defaultCenter]
         addObserverForName:UIDeviceBatteryLevelDidChangeNotification
         object:nil queue:[NSOperationQueue mainQueue]
         usingBlock:^(NSNotification *noti) {
            if (self.eventSink) {
                self.eventSink([NSString stringWithFormat:@"电量:%.f%%", [UIDevice currentDevice].batteryLevel*100]);
            }
         }];

```

**Flutter端**

```
const eventChannel = EventChannel('com.phoneBatteryEvent');
String batteryString = 'Battery: ';


@override
  void initState() {
    // TODO: implement initState
    super.initState();
   // 获取事件
   eventChannel.receiveBroadcastStream().listen(_getData, onError: _getError);
  }
  // 获取消息
  void _getData(dynamic data) {
    var str = data.toString();
    batteryString = "Battery: ${str}.";
    setState(() {
      batteryString = batteryString;
    });
  }
  // 获取错误
  void _getError(Object err) {
    batteryString = 'Battery: unknown.';
  }
  
  @override
  Widget build(BuildContext context) {
  ...
  Container(
            padding: EdgeInsets.all(10),
            child: Text(
              '${batteryString}',
              maxLines: 3,
              style: TextStyle(
                fontSize: 16,
                fontWeight: FontWeight.normal,
                color: Color(0xff333333),
              ),
            ),
          ),
  }
  
```


### Flutter_boost的使用

![](https://upload-images.jianshu.io/upload_images/1426173-b9045d89cd4c3a3d.png)


**Native端**

1、在设置widow时，引入YXPlatformRouter

```
#import "YXPlatformRouter.h"
#import <flutter_boost/FlutterBoost.h>

UIWindowScene *windowScene = (UIWindowScene *)scene;
    UINavigationController *nav = [[UINavigationController alloc] initWithRootViewController:[[ViewController alloc] init]];

    self.window = [[UIWindow alloc] initWithWindowScene:windowScene];
    
    YXPlatformRouter *router = [YXPlatformRouter sharedRouter];
    router.navController = nav;
    
    //使用FLBPlatform初始化FlutterBoost
    [FlutterBoostPlugin.sharedInstance startFlutterWithPlatform:router onStart:^(FlutterEngine * _Nonnull engine) {
    }];
    
    self.window.rootViewController = nav;
    [self.window makeKeyAndVisible];

```


2、创建 YXPlatformRouter实现FLBPlatform的类， FLBPlatform协议里面实现的是原生跳转flutter页面、flutter跳转原生页面、flutter跳转flutter页面 、以及退出flutter页面的方法

```
+ (YXPlatformRouter *)sharedRouter {
    static id instance = nil;
    static dispatch_once_t onceToken;
    dispatch_once(&onceToken, ^{
        instance = [[self alloc] init];
    });
    return instance;
}

- (void)openPage:(NSString *)name
          params:(NSDictionary *)params
        animated:(BOOL)animated
      completion:(void (^)(BOOL))completion {
    
    //原生跳转flutter页面的方法
    if([params[@"present"] boolValue]){
        FLBFlutterViewContainer *vc = FLBFlutterViewContainer.new;
        [vc setName:name params:params];
        vc.edgesForExtendedLayout = UIRectEdgeNone;
        [self.navController presentViewController:vc animated:animated completion:^{}];
    }else{
        
        FLBFlutterViewContainer *vc = [[FLBFlutterViewContainer alloc] init];
        [vc setName:name params:params];
        vc.edgesForExtendedLayout = UIRectEdgeNone;
        if (params[@"navTitle"]) {
            vc.title = params[@"navTitle"];
        }
        [self.navController pushViewController:vc animated:animated];
    }
}
 
- (BOOL)accessibilityEnable {
    return YES;
}

- (void)closePage:(NSString *)uid animated:(BOOL)animated params:(NSDictionary *)params completion:(void (^)(BOOL))completion {
    FLBFlutterViewContainer *vc = (id)self.navController.presentedViewController;
    if([vc isKindOfClass:FLBFlutterViewContainer.class] && [vc.uniqueIDString isEqual: uid]){
        [vc dismissViewControllerAnimated:animated completion:^{}];
    }else{
        [self.navController popViewControllerAnimated:animated];
    }
}

- (void)openNativeVC:(NSString *)name
           urlParams:(NSDictionary *)params
                exts:(NSDictionary *)exts{
    ...
}

#pragma mark - Boost

- (void)open:(NSString *)name
   urlParams:(NSDictionary *)params
        exts:(NSDictionary *)exts
  completion:(void (^)(BOOL))completion {
    ...
}

- (void)present:(NSString *)name
   urlParams:(NSDictionary *)params
        exts:(NSDictionary *)exts
  completion:(void (^)(BOOL))completion {
    ...
}

- (void)close:(NSString *)uid
       result:(NSDictionary *)result
         exts:(NSDictionary *)exts
   completion:(void (^)(BOOL))completion {
   ...
    }

```

Native 跳转 Flutter的三种方式

```
// push
[YXPlatformRouter.sharedRouter openPage:@"firstVC" params:@{@"navTitle":@"NewsList",} animated:YES completion:^(BOOL f){}];

// present
[YXPlatformRouter.sharedRouter openPage:@"secondVC" params:@{@"present":@(YES)} animated:YES completion:^(BOOL f){}];

// 带参数
[YXPlatformRouter.sharedRouter openPage:@"NewsCardDetail" params:@{@"navTitle":@"...", @"content":@"...", @"coverImgUrl":@"..."} animated:YES completion:^(BOOL f){}];

```



**Flutter端**

```
CYHomePage

@override
  void initState() {
    // TODO: implement initState
    super.initState();

    //其中pageName为原生代码传递过来的路由名称（firstVC，secondVC，NewsCardDetail）都是路由名称 params为原生传递过来的参数
    FlutterBoost.singleton.registerPageBuilders({
      'firstVC': (pageName, params, _) => PullDownRefreshList(),
      'secondVC':(pageName, params, _) => BasicWidgetsDemo(),
      'NewsCardDetail':(pageName, params,_) => NewsCardDetail(params: params,),

    });
    FlutterBoost.onPageStart();
  }
```


Flutter 内部跳转Flutter，可以使用两种方式，最好还是使用统一的FlutterBoost.singleton.open

```
// Flutter 跳转 Flutter
  void _clickItem(int index){
    NewsViewModel model = this.list[index];
  	//Navigator.of(context).push(MaterialPageRoute(builder: (context) => NewsCardDetail(model: model,)));
    FlutterBoost.singleton.open('NewsCardDetail',urlParams: {'navTitle':model.title, 'content':model.content, 'coverImgUrl':model.coverImgUrl});
  }
```


Flutter 跳转 Native

```
FlutterBoost.singleton.open('native')
                    .then((Map<dynamic, dynamic> value) {
                });
```


### iOS 注意事项

#### 1、打包报错
在archive的时候报
```
Bitcode bundle could not be generated because '/***/.ios/Flutter/engine/Flutter.framework/Flutter' was built without full bitcode. All frameworks and dylibs for bitcode must be generated from Xcode Archive or Install build file '/***/.ios/Flutter/engine/Flutter.framework/Flutter' for architecture armv7
```
方法一、只要将FlutterModule中的.ios工程中的Bitcode关掉就好了

方法二、若一方法无效，则

1、flutter_module文件夹中 终端执行`flutter clean`

2、终端 `Flutter build ios --release`

3、修改bundleid 和 证书及描述文件

4、然后对.ios工程执行`pod install`，然后bitcode修改为no，然后archive

5、.ios的工程archive没问题的话，再对native 项目执行pod install，bitcode修改为no，然后archive


#### 2、iOS14系统一启动app就闪退
原因:flutter适配问题,flutter为适配iOS14系统,导致真机启动闪退,模拟器没有问题

解决:将`scheme`改成`release`即可



