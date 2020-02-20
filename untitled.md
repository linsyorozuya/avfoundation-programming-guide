# 使用 Assets

Assets 可以从文件, 用户的 iPod 库或 Photo 库中的媒体中创建. 当你创建了一个 asset 对象时, 你可以想立即就能获取到资源的相关信息, 但实际上这些信息并不能立即获取. 当你拥有了一个代表电影资源的 asset 对象时, 你可以使用这个对象来截取电影中的静态图片, 将资源转码成另外一种格式, 或者对电影内容进行删减.

## 创建Asset对象 <a id="&#x521B;&#x5EFA;asset&#x5BF9;&#x8C61;"></a>

可以根据 URL 创建一个 asset 对象, 一个从文件中创建 asset 对象的简单示例如下:

```text
NSURL *url = <#A URL that identifies an audiovisual asset such as a movie file#>;
AVURLAsset *anAsset = [[AVURLAsset alloc] initWithURL:url options:nil];
```

### 初始化 Asset 时的可选设置

AVURLAsset 的初始化方法的第二个参数是一个可选的字典. 这个字典中唯一可用的 key 是 AVURLAssetPreferPreciseDurationAndTimingKey. 这个 key 对应的 value 是一个布尔值, 用来表明资源是否需要为时长的精确展示, 以及随机时间内容的读取进行提前准备.

获取资源的精确时长可能会需要额外的处理开销. 因此使用近似的资源时长是一个很划算的选择, 对于资源的回放而言已经绰绰有余.

* 如果只是单纯的播放资源, 可以对第二个参数传递一个 nil, 或者将 AVURLAssetPreferPreciseDurationAndTimingKey 设为 NO.
* 如果需要将资源添加到一个组合器 \([AVMutableComposition](https://developer.apple.com/reference/avfoundation/avmutablecomposition)\) 中, 那就需要对资源进行随机时间读取, 将 AVURLAssetPreferPreciseDurationAndTimingKey 设为 YES.

示例代码:

```text
NSURL *url = <#A URL that identifies an audiovisual asset such as a movie file#>;
NSDictionary *options = @{ AVURLAssetPreferPreciseDurationAndTimingKey : @YES };
AVURLAsset *anAssetToUseInAComposition = [[AVURLAsset alloc] initWithURL:url options:options];
```

### 访问用户的资源库

要访问用户 iPod 库或者 Photos 库中的资源, 需要先获取对应资源的 URL.

* 访问 iPod 库, 需要创建一个 [MPMediaQuery](https://developer.apple.com/reference/mediaplayer/mpmediaquery) 对象来找到需要的资源, 然后通过 [MPMediaItemPropertyAssetURL](https://developer.apple.com/reference/mediaplayer/mpmediaitempropertyasseturl) 获取到对应资源的 URL. 更多媒体库相关的信息, 参见 [_Multimedia Programming Guide._](https://developer.apple.com/library/prerelease/content/documentation/AudioVideo/Conceptual/MultimediaPG/Introduction/Introduction.html#//apple_ref/doc/uid/TP40009767)
* 访问 Photos 库, 需要使用 [ALAssetsLibrary](https://developer.apple.com/reference/assetslibrary/alassetslibrary)

下面的是获取用户相册中第一个视频的示例代码:

```text
ALAssetsLibrary *library = [[ALAssetsLibrary alloc] init];

// Enumerate just the photos and videos group by using ALAssetsGroupSavedPhotos.
[library enumerateGroupsWithTypes:ALAssetsGroupSavedPhotos usingBlock:^(ALAssetsGroup *group, BOOL *stop) {

// Within the group enumeration block, filter to enumerate just videos.
[group setAssetsFilter:[ALAssetsFilter allVideos]];

// For this example, we're only interested in the first item.
[group enumerateAssetsAtIndexes:[NSIndexSet indexSetWithIndex:0]
                        options:0
                     usingBlock:^(ALAsset *alAsset, NSUInteger index, BOOL *innerStop) {

                         // The end of the enumeration is signaled by asset == nil.
                         if (alAsset) {
                             ALAssetRepresentation *representation = [alAsset defaultRepresentation];
                             NSURL *url = [representation url];
                             AVAsset *avAsset = [AVURLAsset URLAssetWithURL:url options:nil];
                             // Do something interesting with the AV asset.
                         }
                     }];
                 }
                 failureBlock: ^(NSError *error) {
                     // Typically you should handle an error more gracefully than this.
                     NSLog(@"No groups");
                 }];
```

## 准备使用Asset <a id="&#x51C6;&#x5907;&#x4F7F;&#x7528;asset"></a>

初始化一个 asset\(或者 track\)并不意味着资源的所有信息都可以获取使用了. 即使只是资源 \(比如 MP3\) 的时长信息, 框架都可能需要进行一段时间的计算. 为了不阻塞主线程, 最好使用 [AVAsynchronousKeyValueLoading](https://developer.apple.com/reference/avfoundation/avasynchronouskeyvalueloading) 协议来获取资源的信息, 并通过 block 回调使用结果. AVAsset 和 AVAssetTrack 都遵循 AVAsynchronousKeyValueLoading 协议.

可以使用 [statusOfValueForKey:error:](https://developer.apple.com/reference/avfoundation/avasynchronouskeyvalueloading/1386816-statusofvalueforkey) 方法来判断一个资源属性是否加载完成. 当首次加载一个资源时, 资源的大部分或者所有属性的值都是 [AVKeyValueStatusUnknown](https://developer.apple.com/reference/avfoundation/1612428-enumerations/avkeyvaluestatus/1388764). 要加载资源的一个或多个属性, 可以使用 [loadValuesAsynchronouslyForKeys:completionHandler:](https://developer.apple.com/reference/avfoundation/avasynchronouskeyvalueloading/1387321-loadvaluesasynchronouslyforkeys) 方法. 在 completion handler 中, 需要根据属性的状态进行对应的操作, 因为可能因为某些原因导致加载失败, 比如一个无效的网络 URL, 或者加载被取消.

```text
NSURL *url = <#A URL that identifies an audiovisual asset such as a movie file#>;
AVURLAsset *anAsset = [[AVURLAsset alloc] initWithURL:url options:nil];
NSArray *keys = @[@"duration"];

[asset loadValuesAsynchronouslyForKeys:keys completionHandler:^() {

    NSError *error = nil;
    AVKeyValueStatus tracksStatus = [asset statusOfValueForKey:@"duration" error:&error];
    switch (tracksStatus) {
        case AVKeyValueStatusLoaded:
            [self updateUserInterfaceForDuration];
            break;
        case AVKeyValueStatusFailed:
            [self reportError:error forAsset:asset];
            break;
        case AVKeyValueStatusCancelled:
            // Do whatever is appropriate for cancelation.
            break;
   }
}];
If you want to prepare an asset for playback, you should load its tracks property. For more about playing assets, see Playback.
```

如果你需要加载一个资源进行回放, 应该加载资源的 tracks 属性. 更多播放资源的方法, 参见 [Playback](https://developer.apple.com/library/prerelease/content/documentation/AudioVideo/Conceptual/AVFoundationPG/Articles/02_Playback.html#//apple_ref/doc/uid/TP40010188-CH3-SW1).

## 从视频中获取静态图片 <a id="&#x4ECE;&#x89C6;&#x9891;&#x4E2D;&#x83B7;&#x53D6;&#x9759;&#x6001;&#x56FE;&#x7247;"></a>

从视频中获取静态图片 \(比如某个时间点的视频预览缩略图\), 可以使用 [AVAssetImageGenerator](https://developer.apple.com/reference/avfoundation/avassetimagegenerator).

AVAssetImageGenerator 的初始化方法需要传递对应的 asset 对象, 即使创建时 asset 对象中没有任何可视轨道 \(track\), 初始化也可能成功. 如果需要判断一个 asset 是否拥有可视化的 track, 可以使用 [tracksWithMediaCharacteristic:](https://developer.apple.com/reference/avfoundation/avasset/1389554).

```text
AVAsset anAsset = <#Get an asset#>;
if ([[anAsset tracksWithMediaType:AVMediaTypeVideo] count] > 0) {
    AVAssetImageGenerator *imageGenerator =
        [AVAssetImageGenerator assetImageGeneratorWithAsset:anAsset];
    // Implementation continues...
}
```

可以设置 imagegenerator 的属性进行设置, 比如可以使用 maximumSize 属性设置图片的最大尺寸, 使用 apertureMode 属性设置图片的光栅模式. 可以根据给定的时间点生成单张或者一系列的图片. 生成过程中必须确保 imagegenerator 的强引用.

### 生成单张图片

使用 [copyCGImageAtTime:actualTime:error:](https://developer.apple.com/reference/avfoundation/avassetimagegenerator/1387303) 可以从一个指定的时间点生成单张图片. AVFoundation 可以无法精确的根据传入的时间生成图片, 所以可以在第二个参数传入一个 CMTime 指针, 用来接收实际的生成时间.

```text
AVAsset *myAsset = <#An asset#>];
AVAssetImageGenerator *imageGenerator = [[AVAssetImageGenerator alloc] initWithAsset:myAsset];

Float64 durationSeconds = CMTimeGetSeconds([myAsset duration]);
CMTime midpoint = CMTimeMakeWithSeconds(durationSeconds/2.0, 600);
NSError *error;
CMTime actualTime;

CGImageRef halfWayImage = [imageGenerator copyCGImageAtTime:midpoint actualTime:&actualTime error:&error];

if (halfWayImage != NULL) {

    NSString *actualTimeString = (NSString *)CMTimeCopyDescription(NULL, actualTime);
    NSString *requestedTimeString = (NSString *)CMTimeCopyDescription(NULL, midpoint);
    NSLog(@"Got halfWayImage: Asked for %@, got %@", requestedTimeString, actualTimeString);

    // Do something interesting with the image.
    CGImageRelease(halfWayImage);
}
```

### 生成多张图片

要根据多个时间点生成一系列的图片, 可以使用 [generateCGImagesAsynchronouslyForTimes:completionHandler:](https://developer.apple.com/reference/avfoundation/avassetimagegenerator/1388100) 方法. 第一个参数是 NSValue 的数组, 包含多个 CMTime 结构体对象. 第二个参数是每张图片生成后的 block 回调, blcok 的参数中包含一个参数用来表明图片生成的结果, 成功, 失败, 或者被取消. 根据不同情况, 可能包含以下的参数:

* 生成的图片
* 请求生成图片的时间和实际生成图片的时间
* 生成失败的原因

在 block 的实现中, 应当检查图片生成的结果. 另外, 在所有的图片都生成完毕之前, 必须保持对 image generator 的强引用.

```text
AVAsset *myAsset = <#An asset#>];
// Assume: @property (strong) AVAssetImageGenerator *imageGenerator;
self.imageGenerator = [AVAssetImageGenerator assetImageGeneratorWithAsset:myAsset];

Float64 durationSeconds = CMTimeGetSeconds([myAsset duration]);
CMTime firstThird = CMTimeMakeWithSeconds(durationSeconds/3.0, 600);
CMTime secondThird = CMTimeMakeWithSeconds(durationSeconds*2.0/3.0, 600);
CMTime end = CMTimeMakeWithSeconds(durationSeconds, 600);
NSArray *times = @[NSValue valueWithCMTime:kCMTimeZero],
                  [NSValue valueWithCMTime:firstThird], [NSValue valueWithCMTime:secondThird],
                  [NSValue valueWithCMTime:end]];

[imageGenerator generateCGImagesAsynchronouslyForTimes:times
                completionHandler:^(CMTime requestedTime, CGImageRef image, CMTime actualTime,
                                    AVAssetImageGeneratorResult result, NSError *error) {

                NSString *requestedTimeString = (NSString *)
                    CFBridgingRelease(CMTimeCopyDescription(NULL, requestedTime));
                NSString *actualTimeString = (NSString *)
                    CFBridgingRelease(CMTimeCopyDescription(NULL, actualTime));
                NSLog(@"Requested: %@; actual %@", requestedTimeString, actualTimeString);

                if (result == AVAssetImageGeneratorSucceeded) {
                    // Do something interesting with the image.
                }

                if (result == AVAssetImageGeneratorFailed) {
                    NSLog(@"Failed with error: %@", [error localizedDescription]);
                }
                if (result == AVAssetImageGeneratorCancelled) {
                    NSLog(@"Canceled");
                }
  }];
```

可以使用 [cancelAllCGImageGeneration](https://developer.apple.com/reference/avfoundation/avassetimagegenerator/1385859) 取消图片生成.

## 视频的剪辑和转码 <a id="&#x89C6;&#x9891;&#x7684;&#x526A;&#x8F91;&#x548C;&#x8F6C;&#x7801;"></a>

[AVAssetExportSession](https://developer.apple.com/reference/avfoundation/avassetexportsession) 对象可以剪辑视频或者对视频进行格式转换. 流程图如下:![](https://developer.apple.com/library/prerelease/content/documentation/AudioVideo/Conceptual/AVFoundationPG/Art/export_2x.png)

一个导出会话 \(export session\) 用来管理资源的异步导出. 传入一个 asset 来初始化 export session. Export Preset 用来表明导出会话的配置信息, 参见 [allExportPresets](https://developer.apple.com/reference/avfoundation/avassetexportsession/1387150-allexportpresets). 然后配置 export session 指定导出的 URL 和文件格式以及其他信息 \(比如是否因为网络使用而对元数据进行优化\).

可以使用 [exportPresetsCompatibleWithAsset:](https://developer.apple.com/reference/avfoundation/avassetexportsession/1390567-exportpresetscompatiblewithasset) 方法检查是否可以使用某个 Preset.

```text
AVAsset *anAsset = <#Get an asset#>;
NSArray *compatiblePresets = [AVAssetExportSession exportPresetsCompatibleWithAsset:anAsset];
if ([compatiblePresets containsObject:AVAssetExportPresetLowQuality]) {
    AVAssetExportSession *exportSession = [[AVAssetExportSession alloc]
        initWithAsset:anAsset presetName:AVAssetExportPresetLowQuality];
    // Implementation continues.
}
```

需要给 export session 配置一个 output URL\(必须是文件 URL\) 来完成配置. AVAssetExportSession 可以根据 output URL 后缀推断出要导出的文件格式, 也可以通过 [outputFileType](https://developer.apple.com/reference/avfoundation/avassetexportsession/1387110-outputfiletype) 属性直接设置导出格式. 除此之外, 还可以设置一些其他的属性, 比如导出时长, 导出的文件大小, 或者是否需要为网络使用进行优化. 下面的代码根据 [timeRange](https://developer.apple.com/reference/avfoundation/avassetexportsession/1388728) 属性来对视频进行剪辑:

```text
exportSession.outputURL = <#A file URL#>;
exportSession.outputFileType = AVFileTypeQuickTimeMovie;

CMTime start = CMTimeMakeWithSeconds(1.0, 600);
CMTime duration = CMTimeMakeWithSeconds(3.0, 600);
CMTimeRange range = CMTimeRangeMake(start, duration);
exportSession.timeRange = range;
```

使用 [exportAsynchronouslyWithCompletionHandler:](https://developer.apple.com/reference/avfoundation/avassetexportsession/1388005) 方法开始导出文件, 导出完成后会调用 completion handler. 在 completion handler 的实现中, 需要根据 [status](https://developer.apple.com/reference/avfoundation/avassetexportsession/1390528) 属性判断导出是否成功.

```text
[exportSession exportAsynchronouslyWithCompletionHandler:^{

    switch ([exportSession status]) {
        case AVAssetExportSessionStatusFailed:
            NSLog(@"Export failed: %@", [[exportSession error] localizedDescription]);
            break;
        case AVAssetExportSessionStatusCancelled:
            NSLog(@"Export canceled");
            break;
        default:
            break;
    }
}];
```

使用 [cancelExport](https://developer.apple.com/reference/avfoundation/avassetexportsession/1387794-cancelexport) 可以取消导出.

导出到一个已存在的文件或者导出到应用程序沙盒目录外将会导致导出失败. 其他可能导致失败的情况包括:

* 导出过程中接收到电话呼叫
* 程序进入后台, 有其他程序开始使用播放功能

在这些情况下, 要告知用户导出失败, 并允许用户重新开始导出.

> 翻译自 Apple 官方文档: [AVFoundation Programming Guide](https://developer.apple.com/library/prerelease/content/documentation/AudioVideo/Conceptual/AVFoundationPG/Articles/00_Introduction.html#//apple_ref/doc/uid/TP40010188-CH1-SW3)
>
> 翻译人: [www.devzhang.cn](http://www.devzhang.cn/)

