# 静态图片和视频捕捉

通过输入 \(inputs\) 和输出 \(outputs\) 对象来对采集设备 \(比如摄像头或麦克风\) 进行管理. 使用 [AVCaptureSession](https://developer.apple.com/reference/avfoundation/avcapturesession) 对象协调 inputs 和 outputs 之间的数据.

* [AVCaptureDevice](https://developer.apple.com/reference/avfoundation/avcapturedevice) 代表输入设备, 比如摄像头和麦克风
* [AVCaptureInput](https://developer.apple.com/reference/avfoundation/avcaptureinput) 的子类用来对输入设备进行配置
* [AVCaptureOutput](https://developer.apple.com/reference/avfoundation/avcaptureoutput) 的子类用来设置输出结果为图片或者视频
* [AVCaptureSession](https://developer.apple.com/reference/avfoundation/avcapturesession) 用来协调 inputs 和 outputs 之间的数据

使用 CALayer 的子类 [AVCaptureVideoPreviewLayer](https://developer.apple.com/reference/avfoundation/avcapturevideopreviewlayer), 可以展示摄像头采集的画面预览.

对于一个 session, 可以配置多个 inputs 和 outputs, 如图所示:

![](https://developer.apple.com/library/prerelease/content/documentation/AudioVideo/Conceptual/AVFoundationPG/Art/captureOverview_2x.png)

对于大部分的应用而言, 这已经足够了. 但是有些情况下, 会涉及到如何表示一个 inputs 的多个端口 \(ports\), 以及这些 ports 如何连接到 outputs.

Capture session 中使用 [AVCaptureConnection](https://developer.apple.com/reference/avfoundation/avcaptureconnection) 表示 inputs 和 outputs 之间的连接. 一个 Inputs 包含一个或多个 input ports\([AVCaptureInputPort](https://developer.apple.com/reference/avfoundation/avcaptureinputport)\). Outputs 可以从一个或多个来源接收数据, 比如 [AVCaptureMovieFileOutput](https://developer.apple.com/reference/avfoundation/avcapturemoviefileoutput) 可以同时接收视频和音频数据.

如下图所示, 当在 session 中添加一个 input 或 output 时, session 会为所有可匹配的 inputs 和 outputs 之前生成 connections\([AVCaptureConnection](https://developer.apple.com/reference/avfoundation/avcaptureconnection)\).

![](https://developer.apple.com/library/prerelease/content/documentation/AudioVideo/Conceptual/AVFoundationPG/Art/captureDetail_2x.png)

可以使用一个 connection 来开启或关闭一个 input 或 output 数据流. 也可以使用 connection 监控一个 audio 频道的码率平均值和峰值.

> 注意: 媒体捕捉不支持模拟器, 也不能同时使用 iOS 设备上的前置摄像头和后置摄像头进行捕捉

## 使用Capture Session协调数据流 <a id="&#x4F7F;&#x7528;capture-session&#x534F;&#x8C03;&#x6570;&#x636E;&#x6D41;"></a>

数据采集管理的核心是 [AVCaptureSession](https://developer.apple.com/reference/avfoundation/avcapturesession). 在 session 中添加采集设备并对 output 进行配置之后, 可以向 session 发送 [startRunning](https://developer.apple.com/reference/avfoundation/avcapturesession/1388185-startrunning) 消息开始采集, 发送 [stopRunning](https://developer.apple.com/reference/avfoundation/avcapturesession/1385661-stoprunning) 消息停止采集.

```text
AVCaptureSession *session = [[AVCaptureSession alloc] init];
// Add inputs and outputs.
[session startRunning];
```

### 配置 Capture Session

使用 session 的`sessionPreset`属性指定图片质量和分辨率:

* AVCaptureSessionPresetHigh: 高分辨率, 最终效果根据设备不同有所差异
* AVCaptureSessionPresetMedium: 中等分辨率, 适合 Wi-Fi 分享. 最终效果根据设备不同有所差异
* AVCaptureSessionPresetLow: 低分辨率, 适合 3G 分享, 最终效果根据设备不同有所差异
* AVCaptureSessionPreset640x480: 640x480, VGA
* AVCaptureSessionPreset1280x720: 1280x720, 720p HD
* AVCaptureSessionPresetPhoto: 全屏照片, 不能用来作为输出视频

在设置一个 preset 之前, 需要判断设备是否支持该 preset 值:

```text
if ([session canSetSessionPreset:AVCaptureSessionPreset1280x720]) {
    session.sessionPreset = AVCaptureSessionPreset1280x720;
}
else {
    // Handle the failure.
}
```

如果需要设置一个更高分辨率的 preset, 或者在 session 运行时修改一些配置, 需要在 [beginConfiguration](https://developer.apple.com/reference/avfoundation/avcapturesession/1389174-beginconfiguration) 和 [commitConfiguration](https://developer.apple.com/reference/avfoundation/avcapturesession/1388173-commitconfiguration) 之间完成修改. [beginConfiguration](https://developer.apple.com/reference/avfoundation/avcapturesession/1389174-beginconfiguration) 和 [commitConfiguration](https://developer.apple.com/reference/avfoundation/avcapturesession/1388173-commitconfiguration) 方法确保所有的修改被整体应用, 减少对预览状态的影响. 在调用 [beginConfiguration](https://developer.apple.com/reference/avfoundation/avcapturesession/1389174-beginconfiguration) 之后, 可以添加或移除一个 output, 修改`sessionPreset`属性, 或者单独配置 input 和 output 的属性. 只有调用了 [commitConfiguration](https://developer.apple.com/reference/avfoundation/avcapturesession/1388173-commitconfiguration) 方法, 改变才会生效.

```text
[session beginConfiguration];
// Remove an existing capture device.
// Add a new capture device.
// Reset the preset.
[session commitConfiguration];
```

### 监测 Capture Session 的状态

可以使用通知 \(NSNotification\) 监测 session 的状态, 并且所有的通知都在主线程中发送. 注册监听 [AVCaptureSessionRuntimeError](https://developer.apple.com/reference/foundation/nsnotification.name/1389738-avcapturesessionruntimeerror) 通知可以捕捉运行时发生的错误. 也可以使用 session 的`running`属性判断当前的运行状态, `interrupted`属性则可以判断当前是否中断. 此外, `running`和`interrupted`属性都可以通过 KVO 进行监听.

## 使用AVCaptureDevice表示输入设备 <a id="&#x4F7F;&#x7528;avcapturedevice&#x8868;&#x793A;&#x8F93;&#x5165;&#x8BBE;&#x5907;"></a>

[AVCaptureDevice](https://developer.apple.com/reference/avfoundation/avcapturedevice) 是对实际的物理捕捉设备的抽象, 物体捕捉设备向`AVCaptureSession`提供数据. 每个`AVCaptureDevice`对象代表一个实际的输入设备, 例如前摄像头或后摄像头, 或麦克风.

使用`AVCaptureDevice`类的 [devices](https://developer.apple.com/reference/avfoundation/avcapturedevice/1386237-devices) 和 [devicesWithMediaType:](https://developer.apple.com/reference/avfoundation/avcapturedevice/1390520-deviceswithmediatype) 方法可以获取当前可用的捕捉设备. 而且可以获取捕捉设备的设备特性 \(参见 [Device Capture Settings](https://developer.apple.com/library/prerelease/content/documentation/AudioVideo/Conceptual/AVFoundationPG/Articles/04_MediaCapture.html#//apple_ref/doc/uid/TP40010188-CH5-SW18)\). 当前的可用设备的状态可能会发生改变, 当前使用的输入设备可能会变为不可用状态 \(如果设备被另外一个应用使用\), 也可能会有新的设备变为可用状态 \(被其他应用释放\). 注册接收`AVCaptureDeviceWasConnectedNotification`和`AVCaptureDeviceWasDisconnectedNotification`通知可以得知可用设备列表的变化.

### 设备特性

可以获取一个设备的设备特性. 也可以通过 [hasMediaType:](https://developer.apple.com/reference/avfoundation/avcapturedevice/1389487-hasmediatype) 方法判断设备是否支持特定媒体类型的捕捉, 通过 [supportsAVCaptureSessionPreset:](https://developer.apple.com/reference/avfoundation/avcapturedevice/1386263-supportsavcapturesessionpreset) 判断设备是否支持特定的分辨率. 当要提供一个可用的捕捉设备列表给用户进行选择时, 获取展示出设备的位置以及名称 \(比如前摄像头或后摄像头\) 拥有更好的用户体验.

下图展示了前摄像头 \(`AVCaptureDevicePositionFront`\) 和后摄像头 \(`AVCaptureDevicePositionBack`\):

![](https://developer.apple.com/library/prerelease/content/documentation/AudioVideo/Conceptual/AVFoundationPG/Art/cameras_2x.png)

下面的代码遍历了所有的可用设备并打印其名称, 如果是视频设备, 则打印其位置:

```text
NSArray *devices = [AVCaptureDevice devices];

for (AVCaptureDevice *device in devices) {

    NSLog(@"Device name: %@", [device localizedName]);

    if ([device hasMediaType:AVMediaTypeVideo]) {

        if ([device position] == AVCaptureDevicePositionBack) {
            NSLog(@"Device position : back");
        }
        else {
            NSLog(@"Device position : front");
        }
    }
}
```

此外, 还可以获取设备的 model ID 以及 unique ID.

### 捕捉设置

不同的设备之间存在性能差异, 比如一些设备支持特殊的对焦或闪光灯模式, 某些设备还支持兴趣点对焦.

下面的代码示例了如何找出一个支持手电筒模式和特定 preset 的设备:

```text
NSArray *devices = [AVCaptureDevice devicesWithMediaType:AVMediaTypeVideo];
NSMutableArray *torchDevices = [[NSMutableArray alloc] init];

for (AVCaptureDevice *device in devices) {
    [if ([device hasTorch] &&
         [device supportsAVCaptureSessionPreset:AVCaptureSessionPreset640x480]) {
        [torchDevices addObject:device];
    }
}
```

如果找到了多个符合要求的设备, 你可能需要让用户选择其中的某一个设备, 这时可以使用 [localizedName](https://developer.apple.com/reference/avfoundation/avcapturedevice/1388222-localizedname) 属性获取设备的描述信息.

可以用类似的方式实现各种不同的捕捉设置. 框架预定义了一些常量用来代表特定的捕捉模式, 你可以使用这些常量以便于判断设备是否支持特定的模式. 在大部分情况下, 可以通过属性监听获取设备特性的变化状态. 任何情况下, 在改变设备的捕捉设置之前, 都应该先锁定设备, 详见下节`设备的配置`.

> 兴趣点对焦模式和兴趣点曝光模式是互斥的, 正如对焦模式和曝光模式也是互斥的一样

#### 对焦模式

有三种对焦模式:

* `AVCaptureFocusModeLocked`: 固定焦点
* `AVCaptureFocusModeAutoFocus`: 自动对焦然后锁定焦点
* `AVCaptureFocusModeContinuousAutoFocus`: 连续自动对焦

使用 [isFocusModeSupported:](https://developer.apple.com/reference/avfoundation/avcapturedevice/1390215-isfocusmodesupported) 方法判断设备是否支持给定的对焦模式, 然后设置属性 [focusMode](https://developer.apple.com/reference/avfoundation/avcapturedevice/1389191-focusmode) 改变对焦模式.

此外, 一些设备还支持兴趣点对焦模式. 通过方法 [focusPointOfInterestSupported](https://developer.apple.com/reference/avfoundation/avcapturedevice/1390436-isfocuspointofinterestsupported) 判断是否支持该模式, 然后使用属性 [focusPointOfInterest](https://developer.apple.com/reference/avfoundation/avcapturedevice/1385853-focuspointofinterest) 设置焦点. 无论设备是横屏 \(Home 键靠右\) 或竖屏模式, CGPoint{0,0}代表设备左上角, CGPoint{1,1}代表设备右下角.

属性 [adjustingFocus](https://developer.apple.com/reference/avfoundation/avcapturedevice/1390577-adjustingfocus) 可以用来判断当前设备是否正在对焦中. 可以使用 KVO 监听该属性获取对焦开始与结束的通知.

设置对焦模式的示例代码如下:

```text
if ([currentDevice isFocusModeSupported:AVCaptureFocusModeContinuousAutoFocus]) {
    CGPoint autofocusPoint = CGPointMake(0.5f, 0.5f);
    [currentDevice setFocusPointOfInterest:autofocusPoint];
    [currentDevice setFocusMode:AVCaptureFocusModeContinuousAutoFocus];
}
```

#### 曝光模式

有两种曝光模式:

* `AVCaptureExposureModeContinuousAutoExposure`: 自动调整曝光等级
* `AVCaptureExposureModeLocked`: 固定曝光等级

使用 [isExposureModeSupported:](https://developer.apple.com/reference/avfoundation/avcapturedevice/1389048-isexposuremodesupported) 方法判断设备是否支持给定的曝光模式, 然后设置属性 [exposureMode](https://developer.apple.com/reference/avfoundation/avcapturedevice/1388858-exposuremode) 改变曝光模式.  
此外, 一些设备还支持兴趣点曝光模式. 通过方法 [exposurePointOfInterestSupported](https://developer.apple.com/reference/avfoundation/avcapturedevice/1387263-exposurepointofinterestsupported) 判断是否支持该模式, 然后使用属性 [exposurePointOfInterest](https://developer.apple.com/reference/avfoundation/avcapturedevice/1388777-exposurepointofinterest) 设置曝光点. 无论设备是横屏 \(Home 键靠右\) 或竖屏模式, CGPoint{0,0}代表设备左上角, CGPoint{1,1}代表设备右下角.

属性 [adjustingExposure](https://developer.apple.com/reference/avfoundation/avcapturedevice/1386253-adjustingexposure) 可以用来判断当前设备是否正在改变曝光设置中. 可以使用 KVO 监听该属性获取开始设置曝光模式与结束设置曝光模式的通知.

设置曝光模式的示例代码如下:

```text
if ([currentDevice isExposureModeSupported:AVCaptureExposureModeContinuousAutoExposure]) {
    CGPoint exposurePoint = CGPointMake(0.5f, 0.5f);
    [currentDevice setExposurePointOfInterest:exposurePoint];
    [currentDevice setExposureMode:AVCaptureExposureModeContinuousAutoExposure];
}
```

#### 闪光模式

有三种闪光模式:

* `AVCaptureFlashModeOff`: 关闭
* `AVCaptureFlashModeOn`: 打开
* `AVCaptureFlashModeAuto`: 根据环境亮度自动开启或关闭

使用方法 [hasFlash](https://developer.apple.com/reference/avfoundation/avcapturedevice/1388988-hasflash) 判断一个设备是否有闪光灯. 使用方法 [isFlashModeSupported:](https://developer.apple.com/reference/avfoundation/avcapturedevice/1386434-isflashmodesupported) 判断是否支持某个闪光模式, 使用属性 [flashMode](https://developer.apple.com/reference/avfoundation/avcapturedevice/1388116-flashmode) 设置闪光灯模式.

#### 手电筒模式

手电筒模式下, 闪光灯会一直处于开启状态, 用于视频捕捉. 有三种手电筒模式:

* `AVCaptureTorchModeOff`: 关闭
* `AVCaptureTorchModeOn`: 打开
* `AVCaptureTorchModeAuto`: 根据需要自动开启或关闭

使用方法 [hasTorch](https://developer.apple.com/reference/avfoundation/avcapturedevice/1387674-hastorch) 判断一个设备是否有闪光灯. 使用方法 [isTorchModeSupported:](https://developer.apple.com/reference/avfoundation/avcapturedevice/1388822-istorchmodesupported) 判断是否支持某个手电筒模式, 使用属性 [torchMode](https://developer.apple.com/reference/avfoundation/avcapturedevice/1386035-torchmode) 设置手电筒模式.

对于一个有手电筒的设备, 手电筒只有在设备与一个运行中的 capture session 进行了关联后才可以设置为开启.

#### 视频稳定性

依赖于某些特殊的硬件设备, 视频会有更好设置达到电影级别的稳定性. 但并不支持所有的视频格式和分辨率.

开启电影级别的视频稳定性特性在捕捉视频时可能会增加延迟. 使用属性 [videoStabilizationEnabled](https://developer.apple.com/reference/avfoundation/avcaptureconnection/1620485-isvideostabilizationenabled) 可以判断当前是否使用了视频稳定性特性. 属性 [enablesVideoStabilizationWhenAvailable](https://developer.apple.com/reference/avfoundation/avcaptureconnection/1620482-enablesvideostabilizationwhenava) 可以在设备支持的情况下自动开启视频稳定性特性, 该属性默认为关闭状态.

#### 白平衡

有两种白平衡模式:

* `AVCaptureWhiteBalanceModeLocked`: 固定参数的白平衡
* `AVCaptureWhiteBalanceModeContinuousAutoWhiteBalance`: 由相机自动调整白平衡参数

使用方法 [isWhiteBalanceModeSupported:](https://developer.apple.com/reference/avfoundation/avcapturedevice/1388587-iswhitebalancemodesupported) 判断设备是否支持给定的白平衡模式, 然后通过属性 [whiteBalanceMode](https://developer.apple.com/reference/avfoundation/avcapturedevice/1386369-whitebalancemode) 设置白平衡模式.

使用属性 [adjustingWhiteBalance](https://developer.apple.com/reference/avfoundation/avcapturedevice/1386544-adjustingwhitebalance) 判断当前是否正在修改白平衡模式. 可以使用 KVO 监听该属性获取开始设置白平衡模式与结束设置白平衡模式的通知.

#### 设置设备方向

可以在`AVCaptureConnection`上指定期望的设备方向, 用来设置输出时`AVCaptureOutput`\(`AVCaptureMovieFileOutput`, `AVCaptureStillImageOutput`和`AVCaptureVideoDataOutput`\) 的设备方向.

使用属性`AVCaptureConnectionsupportsVideoOrientation`判断设备是否支持修改视频方向, 使用属性`videoOrientation`指定一个方向. 下面的代码将`AVCaptureConnection`的方向设置为`AVCaptureVideoOrientationLandscapeLeft`:

```text
AVCaptureConnection *captureConnection = <#A capture connection#>;
if ([captureConnection isVideoOrientationSupported])
{
    AVCaptureVideoOrientation orientation = AVCaptureVideoOrientationLandscapeLeft;
    [captureConnection setVideoOrientation:orientation];
}
```

### 设备配置

要修改设备的捕捉属性, 首先需要使用方法 [lockForConfiguration:](https://developer.apple.com/reference/avfoundation/avcapturedevice/1387810-lockforconfiguration) 锁定设备, 这样可以避免与其他应用的设置产生冲突.

```text
if ([device isFocusModeSupported:AVCaptureFocusModeLocked]) {
    NSError *error = nil;
    if ([device lockForConfiguration:&error]) {
        device.focusMode = AVCaptureFocusModeLocked;
        [device unlockForConfiguration];
    }
    else {
        // Respond to the failure as appropriate.

    }
```

### 切换设备

某些场景下可能需要允许用户切换输入设备, 比如前后摄像头. 为了避免卡顿, 可以重新配置正在运行的 session, 但是嵌套使用 [beginConfiguration](https://developer.apple.com/reference/avfoundation/avcapturesession/1389174-beginconfiguration) 和 [commitConfiguration](https://developer.apple.com/reference/avfoundation/avcapturesession/1388173-commitconfiguration) 方法.

```text
AVCaptureSession *session = <#A capture session#>;
[session beginConfiguration];

[session removeInput:frontFacingCameraDeviceInput];
[session addInput:backFacingCameraDeviceInput];

[session commitConfiguration];
```

当最后的`commitConfiguration`方法被调用时, 所有的设置变化会一起执行, 确保了切换的流畅性.

## 使用AVCaptureInput添加输入设备 <a id="&#x4F7F;&#x7528;avcaptureinput&#x6DFB;&#x52A0;&#x8F93;&#x5165;&#x8BBE;&#x5907;"></a>

要把一个 capture device 添加到 capture session 中, 需要使用 [AVCaptureDeviceInput](https://developer.apple.com/library/ios/documentation/AVFoundation/Reference/AVCaptureDeviceInput_Class/index.html#//apple_ref/occ/cl/AVCaptureDeviceInput)\(抽象类`AVCaptureInput`的子类\). Capture device input 管理设备的端口.

```text
NSError *error;
AVCaptureDeviceInput *input =
        [AVCaptureDeviceInput deviceInputWithDevice:device error:&error];
if (!input) {
    // Handle the error appropriately.
}
```

使用 [addInput:](https://developer.apple.com/library/ios/documentation/AVFoundation/Reference/AVCaptureSession_Class/index.html#//apple_ref/occ/instm/AVCaptureSession/addInput:) 添加输入. 使用 [canAddInput:](https://developer.apple.com/library/ios/documentation/AVFoundation/Reference/AVCaptureSession_Class/index.html#//apple_ref/occ/instm/AVCaptureSession/canAddInput:) 判断该设备是否可以被添加到 session 中.

```text
AVCaptureSession *captureSession = <#Get a capture session#>;
AVCaptureDeviceInput *captureDeviceInput = <#Get a capture device input#>;
if ([captureSession canAddInput:captureDeviceInput]) {
    [captureSession addInput:captureDeviceInput];
}
else {
    // Handle the failure.
}
```

一个`AVCaptureInput`对象包含一个或多个数据流. 例如, 输入设备可能同时提供音频和视频数据. 每个 [AVCaptureInputPort](https://developer.apple.com/library/ios/documentation/AVFoundation/Reference/AVCaptureInputPort_Class/index.html#//apple_ref/occ/cl/AVCaptureInputPort) 对象代表一个媒体数据流. Capture session 使用一个`AVCaptureConnection`对象定义一组`AVCaptureInputPort`和一个`AVCaptureOutput`之间的映射关系.

## 使用AVCaptureOutput输出数据 <a id="&#x4F7F;&#x7528;avcaptureoutput&#x8F93;&#x51FA;&#x6570;&#x636E;"></a>

要从 capture session 中输出数据, 可以向其添加一个或多个 outputs\([AVCaptureOutput](https://developer.apple.com/library/ios/documentation/AVFoundation/Reference/AVCaptureOutput_Class/index.html#//apple_ref/occ/cl/AVCaptureOutput) 的子类\), 你可以使用:

* [AVCaptureMovieFileOutput](https://developer.apple.com/library/ios/documentation/AVFoundation/Reference/AVCaptureMovieFileOutput_Class/index.html#//apple_ref/occ/cl/AVCaptureMovieFileOutput): 输出为电影文件
* [AVCaptureVideoDataOutput](https://developer.apple.com/library/ios/documentation/AVFoundation/Reference/AVCaptureVideoDataOutput_Class/index.html#//apple_ref/occ/cl/AVCaptureVideoDataOutput): 可以逐帧处理捕捉到的视频
* [AVCaptureAudioDataOutput](https://developer.apple.com/library/ios/documentation/AVFoundation/Reference/AVCaptureAudioDataOutput_Class/index.html#//apple_ref/occ/cl/AVCaptureAudioDataOutput): 可以处理捕捉到的音频数据
* [AVCaptureStillImageOutput](https://developer.apple.com/library/ios/documentation/AVFoundation/Reference/AVCaptureStillImageOutput_Class/index.html#//apple_ref/occ/cl/AVCaptureStillImageOutput): 输出为静态图片

使用方法 [addOutput:](https://developer.apple.com/library/ios/documentation/AVFoundation/Reference/AVCaptureSession_Class/index.html#//apple_ref/occ/instm/AVCaptureSession/addOutput:) 在 capture session 中添加 outputs. 使用方法 [canAddOutput:](https://developer.apple.com/library/ios/documentation/AVFoundation/Reference/AVCaptureSession_Class/index.html#//apple_ref/occ/instm/AVCaptureSession/canAddOutput:) 判断是否可以添加一个给定的 output. 可以根据需要在 session 运行过程中添加或移除一个 output.

```text
AVCaptureSession *captureSession = <#Get a capture session#>;
AVCaptureMovieFileOutput *movieOutput = <#Create and configure a movie output#>;
if ([captureSession canAddOutput:movieOutput]) {
    [captureSession addOutput:movieOutput];
}
else {
    // Handle the failure.
}
```

### 输出为视频文件

使用 [AVCaptureMovieFileOutput](https://developer.apple.com/reference/avfoundation/avcapturemoviefileoutput) 将视频数据保存为一个本地文件. 可以对 movie file output 的参数进行配置, 比如最大的录制时长, 最大的录制文件大小, 如果设备磁盘空间不足的话, 还可以阻止用户进行视频录制.

```text
AVCaptureMovieFileOutput *aMovieFileOutput = [[AVCaptureMovieFileOutput alloc] init];
CMTime maxDuration = <#Create a CMTime to represent the maximum duration#>;
aMovieFileOutput.maxRecordedDuration = maxDuration;
aMovieFileOutput.minFreeDiskSpaceLimit = <#An appropriate minimum given the quality of the movie format and the duration#>;
```

输出的分辨率和码率依赖于 capture session 的`sessionPreset` 属性, 常用的视频编码格式是 H.264, 音频编码格式是 AAC. 实际的编码格式可能由于设备不同有所差异.

#### 开始录制

使用方法 [startRecordingToOutputFileURL:recordingDelegate:](https://developer.apple.com/reference/avfoundation/avcapturefileoutput/1387224-startrecordingtooutputfileurl) 开始录制一段 QuickTime 视频, 方法中需要传入一个本地文件的 URL 和一个录制的 delegate. 传入的本地 URL 不能是已经存在的文件, 因为 movie file output 不会对已存在的文件进行重写, 而且对传入的文件路径, 程序必须有写入权限. 传入的 delegate 必须遵循 [AVCaptureFileOutputRecordingDelegate](https://developer.apple.com/reference/avfoundation/avcapturefileoutputrecordingdelegate) 协议, 且必须实现 [captureOutput:didFinishRecordingToOutputFileAtURL:fromConnections:error:](https://developer.apple.com/reference/avfoundation/avcapturefileoutputrecordingdelegate/1390612-captureoutput) 方法. 在这个代理方法中, delegate 可能会向相册写入数据, 需要对错误进行检测.

```text
AVCaptureMovieFileOutput *aMovieFileOutput = <#Get a movie file output#>;
NSURL *fileURL = <#A file URL that identifies the output location#>;
[aMovieFileOutput startRecordingToOutputFileURL:fileURL recordingDelegate:<#The delegate#>];
```

#### 确保文件写入成功

要判断文件是否写入成功, 在 [captureOutput:didFinishRecordingToOutputFileAtURL:fromConnections:error:](https://developer.apple.com/reference/avfoundation/avcapturefileoutputrecordingdelegate/1390612-captureoutput) 方法中不仅需要检测 error, 还需要对 error 中的 user info 字典中的 [AVErrorRecordingSuccessfullyFinishedKey](https://developer.apple.com/reference/avfoundation/averrorrecordingsuccessfullyfinishedkey) 进行判断.

```text
- (void)captureOutput:(AVCaptureFileOutput *)captureOutput
        didFinishRecordingToOutputFileAtURL:(NSURL *)outputFileURL
        fromConnections:(NSArray *)connections
        error:(NSError *)error {

    BOOL recordedSuccessfully = YES;
    if ([error code] != noErr) {
        // A problem occurred: Find out if the recording was successful.
        id value = [[error userInfo] objectForKey:AVErrorRecordingSuccessfullyFinishedKey];
        if (value) {
            recordedSuccessfully = [value boolValue];
        }
    }
    // Continue as appropriate...
```

需要对`AVErrorRecordingSuccessfullyFinishedKey`进行判断是因为即使写入过程中抛出了一个 error, 文件也可能被成功写入了. 抛出的 error 可能是因为达到了一些设置的限制约束条件, 比如 [AVErrorMaximumDurationReached](https://developer.apple.com/reference/avfoundation/averror.code/1390284-maximumdurationreached), 以及 [AVErrorMaximumFileSizeReached](https://developer.apple.com/reference/avfoundation/averror/averrormaximumfilesizereached). 其他可能导致录制中断的情况如下:

* 磁盘已满 - [AVErrorDiskFull](https://developer.apple.com/reference/avfoundation/averror/averrordiskfull)
* 与录制的设备的连接断开 - [AVErrorDeviceWasDisconnected](https://developer.apple.com/reference/avfoundation/averror/averrordevicewasdisconnected)
* session 中断 \(比如有电话接入\) - [AVErrorSessionWasInterrupted](https://developer.apple.com/reference/avfoundation/averror/averrorsessionwasinterrupted)

#### 在文件中添加元数据

可在任何时刻对文件的元数据 \(metadata\) 进行设置, 哪怕是在录制过程中. 一个 file output 的 metadata 由一个 [AVMetadataItem](https://developer.apple.com/reference/avfoundation/avmetadataitem) 对象的数组来表示. 可以使用其可变子类 [AVMutableMetadataItem](https://developer.apple.com/reference/avfoundation/avmutablemetadataitem) 创建自定义的 metadata.

```text
AVCaptureMovieFileOutput *aMovieFileOutput = <#Get a movie file output#>;
NSArray *existingMetadataArray = aMovieFileOutput.metadata;
NSMutableArray *newMetadataArray = nil;
if (existingMetadataArray) {
    newMetadataArray = [existingMetadataArray mutableCopy];
}
else {
    newMetadataArray = [[NSMutableArray alloc] init];
}

AVMutableMetadataItem *item = [[AVMutableMetadataItem alloc] init];
item.keySpace = AVMetadataKeySpaceCommon;
item.key = AVMetadataCommonKeyLocation;

CLLocation *location - <#The location to set#>;
item.value = [NSString stringWithFormat:@"%+08.4lf%+09.4lf/"
    location.coordinate.latitude, location.coordinate.longitude];

[newMetadataArray addObject:item];

aMovieFileOutput.metadata = newMetadataArray;
```

### 处理视频帧

[AVCaptureVideoDataOutput](https://developer.apple.com/library/ios/documentation/AVFoundation/Reference/AVCaptureVideoDataOutput_Class/index.html#//apple_ref/occ/cl/AVCaptureVideoDataOutput) 使用代理模式来对视频帧进行处理. 使用方法 [setSampleBufferDelegate:queue:](https://developer.apple.com/reference/avfoundation/avcapturevideodataoutput/1389008-setsamplebufferdelegate) 设置代理, 此外还需要传入代理方法被调用的队列. 必须使用同步队列确保视频帧按照录制顺序被传递到代理方法中. 可以使用队列修改视频帧传递处理的优先级, 参见示例 [SquareCam](https://developer.apple.com/library/content/samplecode/SquareCam/Introduction/Intro.html#//apple_ref/doc/uid/DTS40011190).

在代理方法 [captureOutput:didOutputSampleBuffer:fromConnection:](https://developer.apple.com/reference/avfoundation/avcapturevideodataoutputsamplebufferdelegate/1385775-captureoutput) 中, 视频帧由 [CMSampleBufferRef](https://developer.apple.com/reference/coremedia/cmsamplebuffer) 类型表示. 默认情况下, buffers 被设置为当前设备相机效率最高的格式. 也可以使用属性 [videoSettings](https://developer.apple.com/reference/avfoundation/avcapturevideodataoutput/1389945-videosettings) 自定义输出格式. `videoSettings`属性是一个字典类型, 目前只支持`kCVPixelBufferPixelFormatTypeKey`. 系统建议的视频格式可以通过属性 [availableVideoCVPixelFormatTypes](https://developer.apple.com/reference/avfoundation/avcapturevideodataoutput/1387050-availablevideocvpixelformattypes) 获取, 属性 [availableVideoCodecTypes](https://developer.apple.com/reference/avfoundation/avcapturevideodataoutput/1389227-availablevideocodectypes) 返回支持的编码格式. Core Graphics 和 OpenGL 都很好的兼容了`BGRA`格式.

```text
AVCaptureVideoDataOutput *videoDataOutput = [AVCaptureVideoDataOutput new];
NSDictionary *newSettings =
                @{ (NSString *)kCVPixelBufferPixelFormatTypeKey : @(kCVPixelFormatType_32BGRA) };
videoDataOutput.videoSettings = newSettings;

 // discard if the data output queue is blocked (as we process the still image
[videoDataOutput setAlwaysDiscardsLateVideoFrames:YES];)

// create a serial dispatch queue used for the sample buffer delegate as well as when a still image is captured
// a serial dispatch queue must be used to guarantee that video frames will be delivered in order
// see the header doc for setSampleBufferDelegate:queue: for more information
videoDataOutputQueue = dispatch_queue_create("VideoDataOutputQueue", DISPATCH_QUEUE_SERIAL);
[videoDataOutput setSampleBufferDelegate:self queue:videoDataOutputQueue];

AVCaptureSession *captureSession = <#The Capture Session#>;

if ( [captureSession canAddOutput:videoDataOutput] )
     [captureSession addOutput:videoDataOutput];
```

#### 视频处理时的性能考虑

导出视频应当尽可能的使用低分辨率, 高分辨率会消耗额外的 CPU 和电量.

确保在代理方法`captureOutput:didOutputSampleBuffer:fromConnection:`中处理 sample buffer 时不要使用耗时操作, 如果处理占用时间过长, AV Foundation 会停止向代理方法中传递视频帧, 而且会停止其他的输出, 比如 preview layer 上的预览.

可以设置 capture video data 的属性 [minFrameDuration](https://developer.apple.com/reference/avfoundation/avcapturevideodataoutput/1616296-minframeduration) 通过降低帧率来确保有足够的时间对视频帧进行处理. 将属性 [alwaysDiscardsLateVideoFrames](https://developer.apple.com/reference/avfoundation/avcapturevideodataoutput/1385780-alwaysdiscardslatevideoframes) 设置为`YES`\(默认值\) 的话, 后面的视频帧将会被丢弃, 而不是排队等待处理. 如果你并不介意延迟, 而且需要处理所有的视频帧, 也可以将`alwaysDiscardsLateVideoFrames`设置为`NO`\(即使如此, 也可能会出现掉帧的情况\).

### 捕捉静态图像

使用 [AVCaptureStillImageOutput](https://developer.apple.com/reference/avfoundation/avcapturestillimageoutput) 捕捉带元数据的静态图像. 图片的分辨率依赖于 session 的 preset 设置和具体的硬件设备.

#### 像素和编码格式

不同的设备支持不同的图片格式. 可以使用 [availableImageDataCVPixelFormatTypes](https://developer.apple.com/reference/avfoundation/avcapturestillimageoutput/1388622-availableimagedatacvpixelformatt) 获取设备支持的所有像素格式, 使用 [availableImageDataCodecTypes](https://developer.apple.com/reference/avfoundation/avcapturestillimageoutput/1388312-availableimagedatacodectypes) 可以获取设备支持的图像编码类型. 设置属性字典 [outputSettings](https://developer.apple.com/reference/avfoundation/avcapturestillimageoutput/1389306-outputsettings) 可以指定需要的图片格式:\(\)

```text
AVCaptureStillImageOutput *stillImageOutput = [[AVCaptureStillImageOutput alloc] init];
NSDictionary *outputSettings = @{ AVVideoCodecKey : AVVideoCodecJPEG};
[stillImageOutput setOutputSettings:outputSettings];
```

如果需要的是 JPEG 图片, 则不要指定压缩格式. 相反, 应该让 still image output 进行压缩 \(硬件加速\). 可以使用 [jpegStillImageNSDataRepresentation:](https://developer.apple.com/reference/avfoundation/avcapturestillimageoutput/1388131-jpegstillimagensdatarepresentati) 将图片转换为无压缩的`NSData`对象.

#### 捕捉图片

使用方法 [captureStillImageAsynchronouslyFromConnection:completionHandler:](https://developer.apple.com/reference/avfoundation/avcapturestillimageoutput/1387374-capturestillimageasynchronously) 捕捉图片. 第一个参数是需要捕捉的 connection, 需要判断当前的 connection 中哪个 input 正在采集视频.

```text
AVCaptureConnection *videoConnection = nil;
for (AVCaptureConnection *connection in stillImageOutput.connections) {
    for (AVCaptureInputPort *port in [connection inputPorts]) {
        if ([[port mediaType] isEqual:AVMediaTypeVideo] ) {
            videoConnection = connection;
            break;
        }
    }
    if (videoConnection) { break; }
}
```

方法的第二个参数是一个有两个参数的`block`: 一个包含图像数据的`CMSampleBuffer`类型, 另一个是 NSError 对象. Sample buffer 自身包含了元数据, 比如 EXIF 信息字典, 可以对这些元数据进行修改.

```text
[stillImageOutput captureStillImageAsynchronouslyFromConnection:videoConnection completionHandler:
    ^(CMSampleBufferRef imageSampleBuffer, NSError *error) {
        CFDictionaryRef exifAttachments =
            CMGetAttachment(imageSampleBuffer, kCGImagePropertyExifDictionary, NULL);
        if (exifAttachments) {
            // Do something with the attachments.
        }
        // Continue as appropriate.
    }];
```

## 录制预览 <a id="&#x5F55;&#x5236;&#x9884;&#x89C8;"></a>

可以提供给用户一个 preview, 用来展示正在通过摄像头录制的内容 \(使用 preview layer\), 或者正在通过麦克风记录的音频内容 \(通过监听 audio channel\).

### 视频预览

使用 [AVCaptureVideoPreviewLayer](https://developer.apple.com/reference/avfoundation/avcapturevideopreviewlayer) 可以进行视频预览. `AVCaptureVideoPreviewLayer`是`CALayer`的子类. 进行视频预览不需要设置任何的 output 对象.

使用 [AVCaptureVideoDataOutput](https://developer.apple.com/reference/avfoundation/avcapturevideodataoutput) 类可以在视频展示给用户预览之前对视频进行处理.

与 capture output 不同, 一个 video preview layer 会强引用与其相关联的 session. 这是为了确保在进行视频预览时 session 不会被销毁.

```text
AVCaptureSession *captureSession = <#Get a capture session#>;
CALayer *viewLayer = <#Get a layer from the view in which you want to present the preview#>;

AVCaptureVideoPreviewLayer *captureVideoPreviewLayer = [[AVCaptureVideoPreviewLayer alloc] initWithSession:captureSession];
[viewLayer addSublayer:captureVideoPreviewLayer];
```

大体上, video preview layer 的性质与`CALayer`类似. 你可以对图像进行缩放, 向操作其他任何 layer 一样进行 transformations, rotations 等操作. 一个不同点在于你可能需要设置 layer 的`orientation`属性指定如何对摄像头捕捉的图像方向进行旋转. 此外, 通过属性 [supportsVideoMirroring](https://developer.apple.com/reference/avfoundation/avcaptureconnection/1387424-supportsvideomirroring) 可以判断设备是否支持预览镜像. 属性 [automaticallyAdjustsVideoMirroring](https://developer.apple.com/reference/avfoundation/avcaptureconnection/1387082-automaticallyadjustsvideomirrori) 的默认值为`YES`, 但是仍然可以根据需要设置属性 [videoMirrored](https://developer.apple.com/reference/avfoundation/avcaptureconnection/1389172-isvideomirrored) 进行修改.

#### 视频重力模式

Preview layer 支持三种重力模式, 可以使用属性 [videoGravity](https://developer.apple.com/reference/avfoundation/avcapturevideopreviewlayer/1386708-videogravity) 进行设置:

* `AVLayerVideoGravityResizeAspect`: 保持视频款高比, 当视频内容不能铺满屏幕时, 不足的部分使用黑色背景进行填充.
* `AVLayerVideoGravityResizeAspectFill`: 保持视频款高比, 但是会铺满整个屏幕, 必要时会对视频内容进行裁剪.
* `AVLayerVideoGravityResize`: 拉伸视频内容铺满屏幕, 可能导致图像变形.

#### 预览时使用点击聚焦功能

在 preview layer 上实现点击聚焦功能时, 需要注意视频方向, 视频重力模式以及可能设置了视频镜像. 参见代码示例 [AVCam-iOS: Using AVFoundation to Capture Images and Movies](https://developer.apple.com/library/content/samplecode/AVCam/Introduction/Intro.html#//apple_ref/doc/uid/DTS40010112).

### 展示声音等级

要在 capture connection 中检测声音的均值和峰值, 可以使用 [AVCaptureAudioChannel](https://developer.apple.com/reference/avfoundation/avcaptureaudiochannel) 对象. 声音等级不能使用 KVO 的方式获取, 所以需要根据界面更新的需求定时进行轮询 \(比如每秒 10 次\).

```text
AVCaptureAudioDataOutput *audioDataOutput = <#Get the audio data output#>;
NSArray *connections = audioDataOutput.connections;
if ([connections count] > 0) {
    // There should be only one connection to an AVCaptureAudioDataOutput.
    AVCaptureConnection *connection = [connections objectAtIndex:0];

    NSArray *audioChannels = connection.audioChannels;

    for (AVCaptureAudioChannel *channel in audioChannels) {
        float avg = channel.averagePowerLevel;
        float peak = channel.peakHoldLevel;
        // Update the level meter user interface.
    }
}
```

## 示例: 捕捉视频帧为UIImage对象 <a id="&#x793A;&#x4F8B;-&#x6355;&#x6349;&#x89C6;&#x9891;&#x5E27;&#x4E3A;uiimage&#x5BF9;&#x8C61;"></a>

接下来的代码简单示例了如何捕捉视频, 并将捕捉到的视频帧转换为 UIImage 对象:

* 创建`AVCaptureSession`对象
* 找到合适类型的`AVCaptureDevice`对象进行输入
* 为设备创建`AVCaptureDeviceInput`对象
* 创建`AVCaptureVideoDataOutput`对象获取视频帧
* 实现`AVCaptureVideoDataOutput`的代理
* 实现一个方法将接收到的`CMSampleBuffer`转换为`UIImage`

> 提示: 为了展示核心代码, 这份示例省略了某些内容, 比如内存管理和通知的移除等. 使用 AV Foundation 之前, 你最好已经拥有 Cocoa 框架的使用经验.

### 创建和配置 Capture Session

`AVCaptureSession`用来协调 input 和 output 之间的数据流.

```text
AVCaptureSession *session = [[AVCaptureSession alloc] init];
session.sessionPreset = AVCaptureSessionPresetMedium;
```

### 创建和配置 Device 和 Device Input

`AVCaptureDevice`表示捕捉设备, `AVCaptureInput`用来配置捕捉设备的端口

```text
AVCaptureDevice *device =
        [AVCaptureDevice defaultDeviceWithMediaType:AVMediaTypeVideo];

NSError *error = nil;
AVCaptureDeviceInput *input =
        [AVCaptureDeviceInput deviceInputWithDevice:device error:&error];
if (!input) {
    // Handle the error appropriately.
}
[session addInput:input];
```

### 创建和配置 Video Data Output

使用`AVCaptureVideoDataOutput`处理未压缩的视频帧.

```text
AVCaptureVideoDataOutput *output = [[AVCaptureVideoDataOutput alloc] init];
[session addOutput:output];
output.videoSettings =
                @{ (NSString *)kCVPixelBufferPixelFormatTypeKey : @(kCVPixelFormatType_32BGRA) };
output.minFrameDuration = CMTimeMake(1, 15);

dispatch_queue_t queue = dispatch_queue_create("MyQueue", NULL);
[output setSampleBufferDelegate:self queue:queue];
dispatch_release(queue);
```

### 实现 Sample Buffer 代理方法

```text
- (void)captureOutput:(AVCaptureOutput *)captureOutput
         didOutputSampleBuffer:(CMSampleBufferRef)sampleBuffer
         fromConnection:(AVCaptureConnection *)connection {

    UIImage *image = imageFromSampleBuffer(sampleBuffer);
    // Add your code here that uses the image.
}
```

将转换为 UIImage 的操作代码参见 [Converting CMSampleBuffer to a UIImage Object](https://developer.apple.com/library/content/documentation/AudioVideo/Conceptual/AVFoundationPG/Articles/06_MediaRepresentations.html#//apple_ref/doc/uid/TP40010188-CH2-SW4).

### 开始和停止录制

配置 capture session 之后, 需要确保应用有访问相机的权限.

```text
NSString *mediaType = AVMediaTypeVideo;

[AVCaptureDevice requestAccessForMediaType:mediaType completionHandler:^(BOOL granted) {
    if (granted)
    {
        //Granted access to mediaType
        [self setDeviceAuthorized:YES];
    }
    else
    {
        //Not granted access to mediaType
        dispatch_async(dispatch_get_main_queue(), ^{
        [[[UIAlertView alloc] initWithTitle:@"AVCam!"
                                    message:@"AVCam doesn't have permission to use Camera, please change privacy settings"
                                   delegate:self
                          cancelButtonTitle:@"OK"
                          otherButtonTitles:nil] show];
                [self setDeviceAuthorized:NO];
        });
    }
}];
```

当获取到相应的访问权限之后, 可以使用`startRunning`方法开始录制. `startRunning`会阻塞线程, 所以需要异步调用, 以免阻塞主线程.

```text
[session startRunning];
```

类似的调用`stopRunning`可以停止录制.

