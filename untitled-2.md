# 编辑 Assets

AVFoundation 框架为音视频编辑提供了功能丰富的类集. 这些 API 的核心称为组件 \(compositions\). composition 是一个或多个媒体资源的 track 的集合. [AVMutableComposition](https://developer.apple.com/reference/avfoundation/avmutablecomposition) 类提供了插入和删除 track, 以及管理其时间顺序的的接口. 下图展示了如何通过已存在的 assets 组合成为一个 composition. 如果你需要顺序合并多个 asset 到一个文件中, 这就刚刚够用. 但是如果要对 track 执行任何自定义的音视频处理操作, 那么你需要分别对音频和视频进行合并.

![](https://developer.apple.com/library/prerelease/content/documentation/AudioVideo/Conceptual/AVFoundationPG/Art/avmutablecomposition_2x.png)

如下图中所示, 使用 [AVMutableAudioMix](https://developer.apple.com/reference/avfoundation/avmutableaudiomix) 类可以对 composition 中的 audio track 进行自定义操作. 你还可以指定 audio track 的最大音量以及为其设置渐变效果.

![](https://developer.apple.com/library/prerelease/content/documentation/AudioVideo/Conceptual/AVFoundationPG/Art/avmutableaudiomix_2x.png)

如下图所示, 使用 [AVMutableVideoComposition](https://developer.apple.com/reference/avfoundation/avmutablevideocomposition) 类可以直接处理视频 track. 从一个 video composition 输出视频时, 还可以指定输出的尺寸, 缩放比例, 以及帧率. 通过 video composition 的指令 \(instructions,[AVMutableVideoCompositionInstruction](https://developer.apple.com/reference/avfoundation/avmutablevideocompositioninstruction)\), 可以修改视频背景色, 以及设置 layer 的 instructions. Layer 的 instructions\([AVMutableVideoCompositionLayerInstruction](https://developer.apple.com/reference/avfoundation/avmutablevideocompositionlayerinstruction)\) 可以对 video track 实现渐变, 渐变变换, 透明度, 透明度变换等效果. Video composition 还允许通过`animationTool`属性在视频中应用 Core Animation 框架的一些效果.

![](https://developer.apple.com/library/prerelease/content/documentation/AudioVideo/Conceptual/AVFoundationPG/Art/avmutablevideocomposition_2x.png)

如下图所示, 要对音视频进行组合, 可以使用 [AVAssetExportSession](https://developer.apple.com/reference/avfoundation/avassetexportsession). 使用 composition 初始化一个 export session, 然后分别其设置`audioMix`和`videoComposition`属性.

![](https://developer.apple.com/library/prerelease/content/documentation/AudioVideo/Conceptual/AVFoundationPG/Art/puttingitalltogether_2x.png)

## 创建Composition <a id="&#x521B;&#x5EFA;composition"></a>

可以使用 [AVMutableComposition](https://developer.apple.com/reference/avfoundation/avmutablecomposition) 类创建一个自定义的 Composition. 可以使用 [AVMutableCompositionTrack](https://developer.apple.com/reference/avfoundation/avmutablecompositiontrack) 类在自定义的 Composition 中添加一个或多个 composition tracks. 下面是一个通过 video track 和 audio track 创建 composition 的例子:

```text
AVMutableComposition *mutableComposition = [AVMutableComposition composition];
// Create the video composition track.
AVMutableCompositionTrack *mutableCompositionVideoTrack = [mutableComposition addMutableTrackWithMediaType:AVMediaTypeVideo preferredTrackID:kCMPersistentTrackID_Invalid];
// Create the audio composition track.
AVMutableCompositionTrack *mutableCompositionAudioTrack = [mutableComposition addMutableTrackWithMediaType:AVMediaTypeAudio preferredTrackID:kCMPersistentTrackID_Invalid];
```

### 初始化 Composition 的选项

当在 composition 中添加一个新的 track 时, 必须同时提供媒体类型 \(media type\) 和 track ID. 除了最常用的音频和视频类型, 还有其他的媒体类型可以选择, 比如 [AVMediaTypeSubtitle](https://developer.apple.com/reference/avfoundation/avmediatypesubtitle), [AVMediaTypeText](https://developer.apple.com/reference/avfoundation/avmediatypetext).

每个 track 都会有一个唯一的标识符 track ID. 如果指定 [kCMPersistentTrackID\_Invalid](https://changjianfeishui.gitbooks.io/avfoundation-programming-guide/kCMPersistentTrackID_Invalid) 作为 track ID, 会为关联的 track 自动生成一个唯一的 ID.

## 为Composition增加音视频数据 <a id="&#x4E3A;composition&#x589E;&#x52A0;&#x97F3;&#x89C6;&#x9891;&#x6570;&#x636E;"></a>

要将媒体数据添加到 composition track, 需要访问媒体数据所在的`AVAsset`对象. 可以使用 mutable composition track 的接口将具有相同媒体类型的多个 track 添加到同一个 mutable composition track 中. 下面的例子说明了如何将两个不同的 video asset tracks 顺序添加到一个 composition track 中:

```text
// You can retrieve AVAssets from a number of places, like the camera roll for example.
AVAsset *videoAsset = <#AVAsset with at least one video track#>;
AVAsset *anotherVideoAsset = <#another AVAsset with at least one video track#>;
// Get the first video track from each asset.
AVAssetTrack *videoAssetTrack = [[videoAsset tracksWithMediaType:AVMediaTypeVideo] objectAtIndex:0];
AVAssetTrack *anotherVideoAssetTrack = [[anotherVideoAsset tracksWithMediaType:AVMediaTypeVideo] objectAtIndex:0];
// Add them both to the composition.
[mutableCompositionVideoTrack insertTimeRange:CMTimeRangeMake(kCMTimeZero,videoAssetTrack.timeRange.duration) ofTrack:videoAssetTrack atTime:kCMTimeZero error:nil];
[mutableCompositionVideoTrack insertTimeRange:CMTimeRangeMake(kCMTimeZero,anotherVideoAssetTrack.timeRange.duration) ofTrack:anotherVideoAssetTrack atTime:videoAssetTrack.timeRange.duration error:nil];
```

### 检索兼容的 Composition Tracks

可能的情况下, 每种媒体类型都应当只有一个对之对应的 composition track, 这样会减少资源的使用量. 当连续呈现媒体数据时, 应当将相同类型的媒体数据放到同一个 composition track 中. 通过查询一个 mutable composition, 找出是否有与 asset track 对应的 composition track.

```text
AVMutableCompositionTrack *compatibleCompositionTrack = [mutableComposition mutableTrackCompatibleWithTrack:<#the AVAssetTrack you want to insert#>];
if (compatibleCompositionTrack) {
    // Implementation continues.
}
```

> 注意: 在同一个 composition track 添加多个视频段, 在视频段之间进行切换时可能会掉帧, 嵌入式设备尤其明显. 如何为 composition track 选择合适数量的视频段取决于 App 的设计以及其目标设备.

## 生成音量渐变 <a id="&#x751F;&#x6210;&#x97F3;&#x91CF;&#x6E10;&#x53D8;"></a>

使用一个`AVMutableAudioMix`对象就可以为 composition 中的每一个 audio tracks 单独执行自定义的音频处理操作.

通过`AVMutableAudioMix`的类方法 [audioMix](https://developer.apple.com/reference/avfoundation/avmutableaudiomix/1560973-audiomix) 创建一个 audio mix, 然后使用 [AVMutableAudioMixInputParameters](https://developer.apple.com/reference/avfoundation/avmutableaudiomixinputparameters) 类的接口将 audio mix 与 composition 中特定的 track 关联起来. audio mix 可以用来修改 audio track 的音量. 下面的例子展示了如何给一个 audio track 设置音量渐变让声音有一个缓慢淡出结束的效果:

```text
AVMutableAudioMix *mutableAudioMix = [AVMutableAudioMix audioMix];
// Create the audio mix input parameters object.
AVMutableAudioMixInputParameters *mixParameters = [AVMutableAudioMixInputParameters audioMixInputParametersWithTrack:mutableCompositionAudioTrack];
// Set the volume ramp to slowly fade the audio out over the duration of the composition.
[mixParameters setVolumeRampFromStartVolume:1.f toEndVolume:0.f timeRange:CMTimeRangeMake(kCMTimeZero, mutableComposition.duration)];
// Attach the input parameters to the audio mix.
mutableAudioMix.inputParameters = @[mixParameters];
```

## 自定义视频处理 <a id="&#x81EA;&#x5B9A;&#x4E49;&#x89C6;&#x9891;&#x5904;&#x7406;"></a>

使用`AVMutableVideoComposition`对象可以对 composition 中的 video tracks 执行自定义处理操作. 使用 video composition, 还可以为 video tracks 指定尺寸, 缩放比例, 以及帧率.

### 设置 Composition 的背景色

Video compositions 必须包含一个 [AVVideoCompositionInstruction](https://developer.apple.com/reference/avfoundation/avvideocompositioninstruction) 对象的数组, 其中至少包含一个 video composition instruction. 使用 [AVMutableVideoCompositionInstruction](https://developer.apple.com/reference/avfoundation/avmutablevideocompositioninstruction) 可以创建自定义的 video composition instructions. 使用 video composition instructions 可以修改 composition 的背景色:

```text
AVMutableVideoCompositionInstruction *mutableVideoCompositionInstruction = [AVMutableVideoCompositionInstruction videoCompositionInstruction];
mutableVideoCompositionInstruction.timeRange = CMTimeRangeMake(kCMTimeZero, mutableComposition.duration);
mutableVideoCompositionInstruction.backgroundColor = [[UIColor redColor] CGColor];
```

### 透明度渐变

Video composition instructions 也可以用来设置 video composition layer instructions. [AVMutableVideoCompositionLayerInstruction](https://developer.apple.com/reference/avfoundation/avmutablevideocompositionlayerinstruction) 可以用来设置 video track 的 transforms, transforms 渐变, opacity, opacity 渐变. Video composition instruction 的属性数组 [layerInstructions](https://developer.apple.com/reference/avfoundation/avmutablevideocompositioninstruction/1388912-layerinstructions) 中的 layer instructions 的顺序, 决定了 tracks 中的视频帧是如何被放置和组合的. 下面的代码片段展示了如何在第二个视频出现之前为第一个视频增加一个透明度淡出效果:

```text
AVAsset *firstVideoAssetTrack = <#AVAssetTrack representing the first video segment played in the composition#>;
AVAsset *secondVideoAssetTrack = <#AVAssetTrack representing the second video segment played in the composition#>;
// Create the first video composition instruction.
AVMutableVideoCompositionInstruction *firstVideoCompositionInstruction = [AVMutableVideoCompositionInstruction videoCompositionInstruction];
// Set its time range to span the duration of the first video track.
firstVideoCompositionInstruction.timeRange = CMTimeRangeMake(kCMTimeZero, firstVideoAssetTrack.timeRange.duration);
// Create the layer instruction and associate it with the composition video track.
AVMutableVideoCompositionLayerInstruction *firstVideoLayerInstruction = [AVMutableVideoCompositionLayerInstruction videoCompositionLayerInstructionWithAssetTrack:mutableCompositionVideoTrack];
// Create the opacity ramp to fade out the first video track over its entire duration.
[firstVideoLayerInstruction setOpacityRampFromStartOpacity:1.f toEndOpacity:0.f timeRange:CMTimeRangeMake(kCMTimeZero, firstVideoAssetTrack.timeRange.duration)];
// Create the second video composition instruction so that the second video track isn't transparent.
AVMutableVideoCompositionInstruction *secondVideoCompositionInstruction = [AVMutableVideoCompositionInstruction videoCompositionInstruction];
// Set its time range to span the duration of the second video track.
secondVideoCompositionInstruction.timeRange = CMTimeRangeMake(firstVideoAssetTrack.timeRange.duration, CMTimeAdd(firstVideoAssetTrack.timeRange.duration, secondVideoAssetTrack.timeRange.duration));
// Create the second layer instruction and associate it with the composition video track.
AVMutableVideoCompositionLayerInstruction *secondVideoLayerInstruction = [AVMutableVideoCompositionLayerInstruction videoCompositionLayerInstructionWithAssetTrack:mutableCompositionVideoTrack];
// Attach the first layer instruction to the first video composition instruction.
firstVideoCompositionInstruction.layerInstructions = @[firstVideoLayerInstruction];
// Attach the second layer instruction to the second video composition instruction.
secondVideoCompositionInstruction.layerInstructions = @[secondVideoLayerInstruction];
// Attach both of the video composition instructions to the video composition.
AVMutableVideoComposition *mutableVideoComposition = [AVMutableVideoComposition videoComposition];
mutableVideoComposition.instructions = @[firstVideoCompositionInstruction, secondVideoCompositionInstruction];
```

### 结合 Core Animation

Video composition 的 [animationTool](https://developer.apple.com/reference/avfoundation/avmutablevideocomposition/1390395-animationtool) 属性可以在 composition 中展示 Core Animation 框架的强大能力, 例如视频水印, 视频标题和动画遮罩等. 在 Video compositions 中 Core Animatio 有两种不同的使用方式: 添加一个 Core Animation layer 作为独立的 composition track, 或者直接使用 Core Animation layer 在视频帧中渲染动画效果. 下面的代码展示了后面一种使用方式, 在视频区域的中心添加水印:

```text
CALayer *watermarkLayer = <#CALayer representing your desired watermark image#>;
CALayer *parentLayer = [CALayer layer];
CALayer *videoLayer = [CALayer layer];
parentLayer.frame = CGRectMake(0, 0, mutableVideoComposition.renderSize.width, mutableVideoComposition.renderSize.height);
videoLayer.frame = CGRectMake(0, 0, mutableVideoComposition.renderSize.width, mutableVideoComposition.renderSize.height);
[parentLayer addSublayer:videoLayer];
watermarkLayer.position = CGPointMake(mutableVideoComposition.renderSize.width/2, mutableVideoComposition.renderSize.height/4);
[parentLayer addSublayer:watermarkLayer];
mutableVideoComposition.animationTool = [AVVideoCompositionCoreAnimationTool videoCompositionCoreAnimationToolWithPostProcessingAsVideoLayer:videoLayer inLayer:parentLayer];
```

## 最终示例 <a id="&#x6700;&#x7EC8;&#x793A;&#x4F8B;"></a>

下面的代码简要的展示了如何合并两个 video asset tracks 和一个 audio asset track 为一个视频文件. 包括:

* 创建 [AVMutableComposition](https://developer.apple.com/reference/avfoundation/avmutablecomposition) 对象, 并添加多个 [AVMutableCompositionTrack](https://developer.apple.com/reference/avfoundation/avmutablecompositiontrack) 对象
* 在 composition tracks 中添加 [AVAssetTrack](https://developer.apple.com/reference/avfoundation/avassettrack) 对象的时间范围
* 检查 video asset track 的 [preferredTransform](https://developer.apple.com/reference/avfoundation/avassettrack/1389837-preferredtransform) 属性, 判断视频方向
* 使用 [AVMutableVideoCompositionLayerInstruction](https://developer.apple.com/reference/avfoundation/avmutablevideocompositionlayerinstruction) 对象进行 transform 变换
* 设置 video composition 的 [renderSize](https://developer.apple.com/reference/avfoundation/avmutablevideocomposition/1386365-rendersize) 和 [frameDuration](https://developer.apple.com/reference/avfoundation/avmutablevideocomposition/1390059-frameduration) 属性
* 导出视频文件
* 保存视频文件到相册

> 提示: 为了展示核心代码, 这份示例省略了某些内容, 比如内存管理和通知的移除等. 使用 AV Foundation 之前, 你最好已经拥有 Cocoa 框架的使用经验.

### 创建 Composition

使用`AVMutableComposition`对象组合多个 assets 中的 tracks. 下面的代码创建了一个 composition, 并向其添加了一个 audio track 和一个 video track.

```text
AVMutableComposition *mutableComposition = [AVMutableComposition composition];
AVMutableCompositionTrack *videoCompositionTrack = [mutableComposition addMutableTrackWithMediaType:AVMediaTypeVideo preferredTrackID:kCMPersistentTrackID_Invalid];
AVMutableCompositionTrack *audioCompositionTrack = [mutableComposition addMutableTrackWithMediaType:AVMediaTypeAudio preferredTrackID:kCMPersistentTrackID_Invalid];
```

### 添加 Assets

向 composition 添加两个 video asset tracks 和一个 audio asset track.

```text
AVAssetTrack *firstVideoAssetTrack = [[firstVideoAsset tracksWithMediaType:AVMediaTypeVideo] objectAtIndex:0];
AVAssetTrack *secondVideoAssetTrack = [[secondVideoAsset tracksWithMediaType:AVMediaTypeVideo] objectAtIndex:0];
[videoCompositionTrack insertTimeRange:CMTimeRangeMake(kCMTimeZero, firstVideoAssetTrack.timeRange.duration) ofTrack:firstVideoAssetTrack atTime:kCMTimeZero error:nil];
[videoCompositionTrack insertTimeRange:CMTimeRangeMake(kCMTimeZero, secondVideoAssetTrack.timeRange.duration) ofTrack:secondVideoAssetTrack atTime:firstVideoAssetTrack.timeRange.duration error:nil];
[audioCompositionTrack insertTimeRange:CMTimeRangeMake(kCMTimeZero, CMTimeAdd(firstVideoAssetTrack.timeRange.duration, secondVideoAssetTrack.timeRange.duration)) ofTrack:[[audioAsset tracksWithMediaType:AVMediaTypeAudio] objectAtIndex:0] atTime:kCMTimeZero error:nil];
```

### 判断视频方向

一旦在 composition 中添加了 audio tracks 和 videotracks, 必须确保其中所有的 video tracks 的视频方向都是正确的. 默认情况下, video tracks 默认为横屏模式. 如果 video track 是在竖屏模式下采集的, 那么导出视频时会出现方向错误. 同理, 也不能将一个横向的视频和一个纵向的视频进行合并后导出.

```text
BOOL isFirstVideoPortrait = NO;
CGAffineTransform firstTransform = firstVideoAssetTrack.preferredTransform;
// Check the first video track's preferred transform to determine if it was recorded in portrait mode.
if (firstTransform.a == 0 && firstTransform.d == 0 && (firstTransform.b == 1.0 || firstTransform.b == -1.0) && (firstTransform.c == 1.0 || firstTransform.c == -1.0)) {
    isFirstVideoPortrait = YES;
}
BOOL isSecondVideoPortrait = NO;
CGAffineTransform secondTransform = secondVideoAssetTrack.preferredTransform;
// Check the second video track's preferred transform to determine if it was recorded in portrait mode.
if (secondTransform.a == 0 && secondTransform.d == 0 && (secondTransform.b == 1.0 || secondTransform.b == -1.0) && (secondTransform.c == 1.0 || secondTransform.c == -1.0)) {
    isSecondVideoPortrait = YES;
}
if ((isFirstVideoAssetPortrait && !isSecondVideoAssetPortrait) || (!isFirstVideoAssetPortrait && isSecondVideoAssetPortrait)) {
    UIAlertView *incompatibleVideoOrientationAlert = [[UIAlertView alloc] initWithTitle:@"Error!" message:@"Cannot combine a video shot in portrait mode with a video shot in landscape mode." delegate:self cancelButtonTitle:@"Dismiss" otherButtonTitles:nil];
    [incompatibleVideoOrientationAlert show];
    return;
}
```

### 设置 Video Composition Layer Instructions

一旦确认了视频方向, 就可以对每个视频应用必要的 layer instructions, 并将这些 layer instructions 添加到 video composition 中去.

```text
AVMutableVideoCompositionInstruction *firstVideoCompositionInstruction = [AVMutableVideoCompositionInstruction videoCompositionInstruction];
// Set the time range of the first instruction to span the duration of the first video track.
firstVideoCompositionInstruction.timeRange = CMTimeRangeMake(kCMTimeZero, firstVideoAssetTrack.timeRange.duration);
AVMutableVideoCompositionInstruction * secondVideoCompositionInstruction = [AVMutableVideoCompositionInstruction videoCompositionInstruction];
// Set the time range of the second instruction to span the duration of the second video track.
secondVideoCompositionInstruction.timeRange = CMTimeRangeMake(firstVideoAssetTrack.timeRange.duration, CMTimeAdd(firstVideoAssetTrack.timeRange.duration, secondVideoAssetTrack.timeRange.duration));
AVMutableVideoCompositionLayerInstruction *firstVideoLayerInstruction = [AVMutableVideoCompositionLayerInstruction videoCompositionLayerInstructionWithAssetTrack:videoCompositionTrack];
// Set the transform of the first layer instruction to the preferred transform of the first video track.
[firstVideoLayerInstruction setTransform:firstTransform atTime:kCMTimeZero];
AVMutableVideoCompositionLayerInstruction *secondVideoLayerInstruction = [AVMutableVideoCompositionLayerInstruction videoCompositionLayerInstructionWithAssetTrack:videoCompositionTrack];
// Set the transform of the second layer instruction to the preferred transform of the second video track.
[secondVideoLayerInstruction setTransform:secondTransform atTime:firstVideoAssetTrack.timeRange.duration];
firstVideoCompositionInstruction.layerInstructions = @[firstVideoLayerInstruction];
secondVideoCompositionInstruction.layerInstructions = @[secondVideoLayerInstruction];
AVMutableVideoComposition *mutableVideoComposition = [AVMutableVideoComposition videoComposition];
mutableVideoComposition.instructions = @[firstVideoCompositionInstruction, secondVideoCompositionInstruction];
```

所有的`AVAssetTrack`对象都有一个`preferredTransform`属性, 包含了 asset track 的方向信息. 这个 transform 会在 asset track 在屏幕上展示时被应用. 在上面的代码中, layer instruction 的 transform 被设置为 asset track 的 transform, 便于在你修改了视频尺寸时, 新的 composition 中的视频也能正确的进行展示.

### 设置渲染尺寸和帧率

要修正视频方向, 还必须对 [renderSize](https://developer.apple.com/reference/avfoundation/avmutablevideocomposition/1386365-rendersize) 属性进行调整. 同时也需要设置一个合理的帧率 [frameDuration](https://developer.apple.com/reference/avfoundation/avmutablevideocomposition/1390059-frameduration), 比如 30FPS. 默认情况下, [renderScale](https://developer.apple.com/reference/avfoundation/avmutablevideocomposition/1615787-renderscale) 值为 1.0.

```text
CGSize naturalSizeFirst, naturalSizeSecond;
// If the first video asset was shot in portrait mode, then so was the second one if we made it here.
if (isFirstVideoAssetPortrait) {
// Invert the width and height for the video tracks to ensure that they display properly.
    naturalSizeFirst = CGSizeMake(firstVideoAssetTrack.naturalSize.height, firstVideoAssetTrack.naturalSize.width);
    naturalSizeSecond = CGSizeMake(secondVideoAssetTrack.naturalSize.height, secondVideoAssetTrack.naturalSize.width);
}
else {
// If the videos weren't shot in portrait mode, we can just use their natural sizes.
    naturalSizeFirst = firstVideoAssetTrack.naturalSize;
    naturalSizeSecond = secondVideoAssetTrack.naturalSize;
}
float renderWidth, renderHeight;
// Set the renderWidth and renderHeight to the max of the two videos widths and heights.
if (naturalSizeFirst.width > naturalSizeSecond.width) {
    renderWidth = naturalSizeFirst.width;
}
else {
    renderWidth = naturalSizeSecond.width;
}
if (naturalSizeFirst.height > naturalSizeSecond.height) {
    renderHeight = naturalSizeFirst.height;
}
else {
    renderHeight = naturalSizeSecond.height;
}
mutableVideoComposition.renderSize = CGSizeMake(renderWidth, renderHeight);
// Set the frame duration to an appropriate value (i.e. 30 frames per second for video).
mutableVideoComposition.frameDuration = CMTimeMake(1,30);
```

### 导出 Composition

最后一步是导出 composition 到一个视频文件中, 并将视频文件保存到用户相册中. 使用 [AVAssetExportSession](https://developer.apple.com/reference/avfoundation/avassetexportsession) 创建一个新的视频文件, 并指定要输出的文件目录的 URL. 使用 [ALAssetsLibrary](https://developer.apple.com/reference/assetslibrary/alassetslibrary) 可以将生成的视频文件保存到用户相册中.

```text
// Create a static date formatter so we only have to initialize it once.
static NSDateFormatter *kDateFormatter;
if (!kDateFormatter) {
    kDateFormatter = [[NSDateFormatter alloc] init];
    kDateFormatter.dateStyle = NSDateFormatterMediumStyle;
    kDateFormatter.timeStyle = NSDateFormatterShortStyle;
}
// Create the export session with the composition and set the preset to the highest quality.
AVAssetExportSession *exporter = [[AVAssetExportSession alloc] initWithAsset:mutableComposition presetName:AVAssetExportPresetHighestQuality];
// Set the desired output URL for the file created by the export process.
exporter.outputURL = [[[[NSFileManager defaultManager] URLForDirectory:NSDocumentDirectory inDomain:NSUserDomainMask appropriateForURL:nil create:@YES error:nil] URLByAppendingPathComponent:[kDateFormatter stringFromDate:[NSDate date]]] URLByAppendingPathExtension:CFBridgingRelease(UTTypeCopyPreferredTagWithClass((CFStringRef)AVFileTypeQuickTimeMovie, kUTTagClassFilenameExtension))];
// Set the output file type to be a QuickTime movie.
exporter.outputFileType = AVFileTypeQuickTimeMovie;
exporter.shouldOptimizeForNetworkUse = YES;
exporter.videoComposition = mutableVideoComposition;
// Asynchronously export the composition to a video file and save this file to the camera roll once export completes.
[exporter exportAsynchronouslyWithCompletionHandler:^{
    dispatch_async(dispatch_get_main_queue(), ^{
        if (exporter.status == AVAssetExportSessionStatusCompleted) {
            ALAssetsLibrary *assetsLibrary = [[ALAssetsLibrary alloc] init];
            if ([assetsLibrary videoAtPathIsCompatibleWithSavedPhotosAlbum:exporter.outputURL]) {
                [assetsLibrary writeVideoAtPathToSavedPhotosAlbum:exporter.outputURL completionBlock:NULL];
            }
        }
    });
}];
```

