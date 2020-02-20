# 导出

使用 AVFoundation 提供的导出 \(export\)API 可以对音视频资源操作. [AVAssetExportSession](https://developer.apple.com/reference/avfoundation/avassetexportsession) 类提供的接口可以实现一些简单的 export 需求, 比如修改资源文件格式, 对资源进行删减. 对更复杂的需求, 需要使用 [AVAssetReader](https://developer.apple.com/reference/avfoundation/avassetreader) 和 [AVAssetWriter](https://developer.apple.com/reference/avfoundation/avassetwriter).

在需要对 asset 内容进行操作时使用`AVAssetReader`. 例如, 需要读取 audio track 绘制音频波形图. 在需要将媒体 \(比如 sample buffers 或者静态图像\) 转换为一个 asset 时, 使用`AVAssetWriter`.

> 注意: 不应实时处理时用到这两个类. `AVAssetReader`不能用来读取 HTTP 直播流这样的实时资源. 如果在实时数据处理 \(比如 [AVCaptureOutput](https://developer.apple.com/reference/avfoundation/avcaptureoutput)\) 中使用了`AVAssetWriter`, 需要将`AVAssetWriter`的属性 [expectsMediaDataInRealTime](https://developer.apple.com/reference/avfoundation/avassetwriterinput/1387827-expectsmediadatainrealtime) 设置为`YES`, 这样可以保证以正确的顺序写入文件.

## 读取Asset <a id="&#x8BFB;&#x53D6;asset"></a>

每个`AVAssetReader`对象只能被关联到一个 asset, 但是这个 asset 可能包含多个 track. 因此, 在开始读取之前, 需要配置一个 [AVAssetReaderOutput](https://developer.apple.com/reference/avfoundation/avassetreaderoutput) 的子类来设置媒体数据的读取方式. `AVAssetReaderOutput`有三个子类可以用来读取 asset: [AVAssetReaderTrackOutput](https://developer.apple.com/reference/avfoundation/avassetreadertrackoutput), [AVAssetReaderAudioMixOutput](https://developer.apple.com/reference/avfoundation/avassetreaderaudiomixoutput) 和 [AVAssetReaderVideoCompositionOutput](https://developer.apple.com/reference/avfoundation/avassetreadervideocompositionoutput).

### 创建 Asset Reader

创建`AVAssetReader`对象需要一个 asset 对象:

```text
NSError *outError;
AVAsset *someAsset = <#AVAsset that you want to read#>;
AVAssetReader *assetReader = [AVAssetReader assetReaderWithAsset:someAsset error:&outError];
BOOL success = (assetReader != nil);
```

> 需要检查 assetReader 是否创建成功, 如果失败, error 会包含相关的错误信息.

### 设置 Asset Reader Output

成功创建 assetReader 后, 至少需要设置一个 output 来接收读取的媒体数据. 确保 output 的属性 [alwaysCopiesSampleData](https://developer.apple.com/reference/avfoundation/avassetreaderoutput/1389189-alwayscopiessampledata) 被设置为`NO`, 这样能提升性能. 本章所有的实例代码中, 该属性都设置为`NO`.

如果只是需要从一个或多个 track 中读取数据并修改其格式, 那么可以使用`AVAssetReaderTrackOutput`. 要解压一个 audio track 为 Linear PCM, 需要进行如下设置:

```text
AVAsset *localAsset = assetReader.asset;
// Get the audio track to read.
AVAssetTrack *audioTrack = [[localAsset tracksWithMediaType:AVMediaTypeAudio] objectAtIndex:0];
// Decompression settings for Linear PCM
NSDictionary *decompressionAudioSettings = @{ AVFormatIDKey : [NSNumber numberWithUnsignedInt:kAudioFormatLinearPCM] };
// Create the output with the audio track and decompression settings.
AVAssetReaderOutput *trackOutput = [AVAssetReaderTrackOutput assetReaderTrackOutputWithTrack:audioTrack outputSettings:decompressionAudioSettings];
// Add the output to the reader if possible.
if ([assetReader canAddOutput:trackOutput])
    [assetReader addOutput:trackOutput];
```

> 要以存储时的格式读取数据, 将参数`outputSettings`设置为`nil`.

对于使用 [AVAudioMix](https://developer.apple.com/reference/avfoundation/avaudiomix) 和 [AVVideoComposition](https://developer.apple.com/reference/avfoundation/avvideocomposition) 处理过的 asset, 需要使用`AVAssetReaderAudioMixOutput` 和 `AVAssetReaderVideoCompositionOutput`进行读取. 通常, 当从 [AVComposition](https://developer.apple.com/reference/avfoundation/avcomposition) 对象中读取数据时, 会使用到这些 output 对象.

使用一个`AVAssetReaderAudioMixOutput`对象, 可以读取 asset 中的多个 audio track. 下面的代码展示了为 asset 中所有的 audio track 创建一个`AVAssetReaderAudioMixOutput`对象, 解压 audio track 为 Linear PCM, 并为 output 设置音频混合方式 \(audio mix\):

```text
AVAudioMix *audioMix = <#An AVAudioMix that specifies how the audio tracks from the AVAsset are mixed#>;
// Assumes that assetReader was initialized with an AVComposition object.
AVComposition *composition = (AVComposition *)assetReader.asset;
// Get the audio tracks to read.
NSArray *audioTracks = [composition tracksWithMediaType:AVMediaTypeAudio];
// Get the decompression settings for Linear PCM.
NSDictionary *decompressionAudioSettings = @{ AVFormatIDKey : [NSNumber numberWithUnsignedInt:kAudioFormatLinearPCM] };
// Create the audio mix output with the audio tracks and decompression setttings.
AVAssetReaderOutput *audioMixOutput = [AVAssetReaderAudioMixOutput assetReaderAudioMixOutputWithAudioTracks:audioTracks audioSettings:decompressionAudioSettings];
// Associate the audio mix used to mix the audio tracks being read with the output.
audioMixOutput.audioMix = audioMix;
// Add the output to the reader if possible.
if ([assetReader canAddOutput:audioMixOutput])
    [assetReader addOutput:audioMixOutput];
```

> 设置参数`audioSettings` 为 `nil`, 将返回未被压缩的样本数据. 对`AVAssetReaderVideoCompositionOutput`也一样.

`AVAssetReaderVideoCompositionOutput` 的使用方法大致与`AVAssetReaderAudioMixOutput` 相同, 可以从 asset 中读取多个 video track. 下面的代码示例了如何从多个 video track 中读取数据, 并解压为 ARGB:

```text
AVVideoComposition *videoComposition = <#An AVVideoComposition that specifies how the video tracks from the AVAsset are composited#>;
// Assumes assetReader was initialized with an AVComposition.
AVComposition *composition = (AVComposition *)assetReader.asset;
// Get the video tracks to read.
NSArray *videoTracks = [composition tracksWithMediaType:AVMediaTypeVideo];
// Decompression settings for ARGB.
NSDictionary *decompressionVideoSettings = @{ (id)kCVPixelBufferPixelFormatTypeKey : [NSNumber numberWithUnsignedInt:kCVPixelFormatType_32ARGB], (id)kCVPixelBufferIOSurfacePropertiesKey : [NSDictionary dictionary] };
// Create the video composition output with the video tracks and decompression setttings.
AVAssetReaderOutput *videoCompositionOutput = [AVAssetReaderVideoCompositionOutput assetReaderVideoCompositionOutputWithVideoTracks:videoTracks videoSettings:decompressionVideoSettings];
// Associate the video composition used to composite the video tracks being read with the output.
videoCompositionOutput.videoComposition = videoComposition;
// Add the output to the reader if possible.
if ([assetReader canAddOutput:videoCompositionOutput])
    [assetReader addOutput:videoCompositionOutput];
```

### 读取 Asset 中的媒体数据

按需设置 outputs 之后, 调用 asset reader 的方法 [startReading](https://developer.apple.com/reference/avfoundation/avassetreader/1390286-startreading) 开始读取数据. 然后使用方法 [copyNextSampleBuffer](https://developer.apple.com/reference/avfoundation/avassetreaderoutput/1385732-copynextsamplebuffer) 从 outputs 中获取媒体数据. 示例如下:

```text
// Start the asset reader up.
[self.assetReader startReading];
BOOL done = NO;
while (!done)
{
  // Copy the next sample buffer from the reader output.
  CMSampleBufferRef sampleBuffer = [self.assetReaderOutput copyNextSampleBuffer];
  if (sampleBuffer)
  {
    // Do something with sampleBuffer here.
    CFRelease(sampleBuffer);
    sampleBuffer = NULL;
  }
  else
  {
    // Find out why the asset reader output couldn't copy another sample buffer.
    if (self.assetReader.status == AVAssetReaderStatusFailed)
    {
      NSError *failureError = self.assetReader.error;
      // Handle the error here.
    }
    else
    {
      // The asset reader output has read all of its samples.
      done = YES;
    }
  }
}
```

## 写入Asset <a id="&#x5199;&#x5165;asset"></a>

[AVAssetWriter](https://developer.apple.com/reference/avfoundation/avassetwriter) 将多个来源的数据以指定格式写入到单个文件中. Asset writer 并不与一个特定的 asset 相关联, 但必须与要输出的文件相关联. 由于一个 asset writer 可以从多个来源获取数据, 所以需要为每个要写入的 track 创建对应的 [AVAssetWriterInput](https://developer.apple.com/reference/avfoundation/avassetwriterinput) 对象. 每个`AVAssetWriterInput`对象接收 [CMSampleBufferRef](https://developer.apple.com/reference/coremedia/cmsamplebuffer) 类型的数据, 如果想要添加 [CVPixelBufferRef](https://developer.apple.com/reference/corevideo/cvpixelbufferref) 类型的数据, 可以使用 [AVAssetWriterInputPixelBufferAdaptor](https://developer.apple.com/reference/avfoundation/avassetwriterinputpixelbufferadaptor).

### 创建 AVAssetWriter

创建 AVAssetWriter 对象需要指定一个文件 URL 和文件格式. 下面的代码示例了如何初始化一个 AVAssetWriter 用来创建 QuickTime 电影.

```text
NSError *outError;
NSURL *outputURL = <#NSURL object representing the URL where you want to save the video#>;
AVAssetWriter *assetWriter = [AVAssetWriter assetWriterWithURL:outputURL
                                                      fileType:AVFileTypeQuickTimeMovie
                                                         error:&outError];
BOOL success = (assetWriter != nil);
```

### 设置 Asset Writer Inputs

要让 AVAssetWriter 能写入媒体数据, 必须至少设置一个 asset writer input. 例如要写入`CMSampleBufferRef`类型的数据, 需要使用`AVAssetWriterInput`. 下面的代码示例了将压缩的音频数据写入为 128 kbps 的 AAC 格式:

```text
// Configure the channel layout as stereo.
AudioChannelLayout stereoChannelLayout = {
    .mChannelLayoutTag = kAudioChannelLayoutTag_Stereo,
    .mChannelBitmap = 0,
    .mNumberChannelDescriptions = 0
};

// Convert the channel layout object to an NSData object.
NSData *channelLayoutAsData = [NSData dataWithBytes:&stereoChannelLayout length:offsetof(AudioChannelLayout, mChannelDescriptions)];

// Get the compression settings for 128 kbps AAC.
NSDictionary *compressionAudioSettings = @{
    AVFormatIDKey         : [NSNumber numberWithUnsignedInt:kAudioFormatMPEG4AAC],
    AVEncoderBitRateKey   : [NSNumber numberWithInteger:128000],
    AVSampleRateKey       : [NSNumber numberWithInteger:44100],
    AVChannelLayoutKey    : channelLayoutAsData,
    AVNumberOfChannelsKey : [NSNumber numberWithUnsignedInteger:2]
};

// Create the asset writer input with the compression settings and specify the media type as audio.
AVAssetWriterInput *assetWriterInput = [AVAssetWriterInput assetWriterInputWithMediaType:AVMediaTypeAudio outputSettings:compressionAudioSettings];
// Add the input to the writer if possible.
if ([assetWriter canAddInput:assetWriterInput])
    [assetWriter addInput:assetWriterInput];
```

> 只有 asset writer 初始化时`fileType`为 [AVFileTypeQuickTimeMovie](https://developer.apple.com/reference/avfoundation/avfiletypequicktimemovie), 参数`outputSettings`才能为 nil, 意味着写入的文件格式为 QuickTime movie.

使用属性 [metadata](https://developer.apple.com/reference/avfoundation/avassetwriterinput/1386328-metadata) 和 [transform](https://developer.apple.com/reference/avfoundation/avassetwriterinput/1390183-transform) 可以为指定的 track 设置 metadata 和 transform. 当输入源为 video track 时, 可以通过如下方式持有 video track 的原始 transform:

```text
AVAsset *videoAsset = <#AVAsset with at least one video track#>;
AVAssetTrack *videoAssetTrack = [[videoAsset tracksWithMediaType:AVMediaTypeVideo] objectAtIndex:0];
assetWriterInput.transform = videoAssetTrack.preferredTransform;
```

> 注意, 需要在开始写入之前设置这两个属性才会生效.

在写入文件时, 有时候可能会需要分配一个 pixel buffer, 这时可以使用`AVAssetWriterInputPixelBufferAdaptor`类. 为了提高效率, 可以直接使用 pixel buffer adaptor 提供的 pixel buffer pool. 下面的代码示例了创建了一个 pixel buffer 对象处理 RGB 色域:

```text
NSDictionary *pixelBufferAttributes = @{
     kCVPixelBufferCGImageCompatibilityKey : [NSNumber numberWithBool:YES],
     kCVPixelBufferCGBitmapContextCompatibilityKey : [NSNumber numberWithBool:YES],
     kCVPixelBufferPixelFormatTypeKey : [NSNumber numberWithInt:kCVPixelFormatType_32ARGB]
};
AVAssetWriterInputPixelBufferAdaptor *inputPixelBufferAdaptor = [AVAssetWriterInputPixelBufferAdaptor assetWriterInputPixelBufferAdaptorWithAssetWriterInput:self.assetWriterInput sourcePixelBufferAttributes:pixelBufferAttributes];
```

> 注意, 所有的`AVAssetWriterInputPixelBufferAdaptor`对象都必须与一个 asset writer input 相关联 . 这个 asset writer input 对象必须接收`AVMediaTypeVideo`类型的数据.

### 写入媒体数据

当配置完 asset writer 之后, 就可以\) 开始写入数据了. 调用方法 [startWriting](https://developer.apple.com/reference/avfoundation/avassetwriter/1386724-startwriting) 初始化写入过程. 然后调用方法 [startSessionAtSourceTime:](https://developer.apple.com/reference/avfoundation/avassetwriter/1389908-startsessionatsourcetime) 开启一个写入会话 \(sample-writing session\). Asset writer 的所有写入过程都通过这个 session 完成, 并且 sesion 的时间范围决定了源媒体数据中哪个时间范围内的数据会被写入到文件中. 例如, 只写入源数据的后一半的示例代码如下:

```text
CMTime halfAssetDuration = CMTimeMultiplyByFloat64(self.asset.duration, 0.5);
[self.assetWriter startSessionAtSourceTime:halfAssetDuration];
//Implementation continues.
```

一般情况下, 方法 [endSessionAtSourceTime:](https://developer.apple.com/reference/avfoundation/avassetwriter/1389921-endsession) 用来结束写入会话. 但是如果文件已经写入完毕, 则可以方法 [finishWriting](https://developer.apple.com/reference/avfoundation/avassetwriter/1426644-finishwriting) 结束写入会话. 下面的代码示例了从一个输入源读取数据并写入所有读取到的数据:

```text
// Prepare the asset writer for writing.
[self.assetWriter startWriting];
// Start a sample-writing session.
[self.assetWriter startSessionAtSourceTime:kCMTimeZero];
// Specify the block to execute when the asset writer is ready for media data and the queue to call it on.
[self.assetWriterInput requestMediaDataWhenReadyOnQueue:myInputSerialQueue usingBlock:^{
     while ([self.assetWriterInput isReadyForMoreMediaData])
     {
          // Get the next sample buffer.
          CMSampleBufferRef nextSampleBuffer = [self copyNextSampleBufferToWrite];
          if (nextSampleBuffer)
          {
               // If it exists, append the next sample buffer to the output file.
               [self.assetWriterInput appendSampleBuffer:nextSampleBuffer];
               CFRelease(nextSampleBuffer);
               nextSampleBuffer = nil;
          }
          else
          {
               // Assume that lack of a next sample buffer means the sample buffer source is out of samples and mark the input as finished.
               [self.assetWriterInput markAsFinished];
               break;
          }
     }
}];
```

## 重编码Assets <a id="&#x91CD;&#x7F16;&#x7801;assets"></a>

上面代码中的`copyNextSampleBufferToWrite`方法仅仅是一个存根 \(stub\). 这个 stub 需要实现一些逻辑用来返回要写入的`CMSampleBufferRef`对象. Sample buffers 可能来源于一个 asset reader output.

可以搭配使用 asset reader 和 asset writer 进行 asset 之间的转换. 相比于使用`AVAssetExportSession`, 使用这些对象可以更好的控制转换细节. 例如, 可以选择导出哪个 track, 可以指定导出的文件格式, 还可以指定导出的时间范围. 下面的代码片段示例了如何从一个 asset reader output 读取数据, 并使用 asset writer input 写入这些数据.

```text
NSString *serializationQueueDescription = [NSString stringWithFormat:@"%@ serialization queue", self];

// Create a serialization queue for reading and writing.
dispatch_queue_t serializationQueue = dispatch_queue_create([serializationQueueDescription UTF8String], NULL);

// Specify the block to execute when the asset writer is ready for media data and the queue to call it on.
[self.assetWriterInput requestMediaDataWhenReadyOnQueue:serializationQueue usingBlock:^{
     while ([self.assetWriterInput isReadyForMoreMediaData])
     {
          // Get the asset reader output's next sample buffer.
          CMSampleBufferRef sampleBuffer = [self.assetReaderOutput copyNextSampleBuffer];
          if (sampleBuffer != NULL)
          {
               // If it exists, append this sample buffer to the output file.
               BOOL success = [self.assetWriterInput appendSampleBuffer:sampleBuffer];
               CFRelease(sampleBuffer);
               sampleBuffer = NULL;
               // Check for errors that may have occurred when appending the new sample buffer.
               if (!success && self.assetWriter.status == AVAssetWriterStatusFailed)
               {
                    NSError *failureError = self.assetWriter.error;
                    //Handle the error.
               }
          }
          else
          {
               // If the next sample buffer doesn't exist, find out why the asset reader output couldn't vend another one.
               if (self.assetReader.status == AVAssetReaderStatusFailed)
               {
                    NSError *failureError = self.assetReader.error;
                    //Handle the error here.
               }
               else
               {
                    // The asset reader output must have vended all of its samples. Mark the input as finished.
                    [self.assetWriterInput markAsFinished];
                    break;
               }
          }
     }
}];
```

## 最终示例: 使用Asset Reader 和 Writer 对 Asset 进行重编码 <a id="&#x6700;&#x7EC8;&#x793A;&#x4F8B;-&#x4F7F;&#x7528;asset-reader-&#x548C;-writer-&#x5BF9;-asset-&#x8FDB;&#x884C;&#x91CD;&#x7F16;&#x7801;"></a>

下面的代码简要示例了使用 asset reader 和 writer 对一个 asset 中的第一个 video 和 audio track 进行重新编码并将结果数据写入到一个新文件中.

> 提示: 为了将注意力集中在核心代码上, 这份示例省略了某些内容.

### 初始化设置

在创建和配置 asset reader 和 writer 之前, 需要进行一些初始化设置. 首先需要为读写过程创建三个串行队列.

```text
NSString *serializationQueueDescription = [NSString stringWithFormat:@"%@ serialization queue", self];

// Create the main serialization queue.
self.mainSerializationQueue = dispatch_queue_create([serializationQueueDescription UTF8String], NULL);
NSString *rwAudioSerializationQueueDescription = [NSString stringWithFormat:@"%@ rw audio serialization queue", self];

// Create the serialization queue to use for reading and writing the audio data.
self.rwAudioSerializationQueue = dispatch_queue_create([rwAudioSerializationQueueDescription UTF8String], NULL);
NSString *rwVideoSerializationQueueDescription = [NSString stringWithFormat:@"%@ rw video serialization queue", self];

// Create the serialization queue to use for reading and writing the video data.
self.rwVideoSerializationQueue = dispatch_queue_create([rwVideoSerializationQueueDescription UTF8String], NULL);
```

队列 mainSerializationQueue 用于 asset reader 和 writer 的启动, 停止和取消. 其他两个队列用于 output/input 的读取和写入.

### 接着, 加载 asset 中的 track, 并开始重编码.

```text
self.asset = <#AVAsset that you want to reencode#>;
self.cancelled = NO;
self.outputURL = <#NSURL representing desired output URL for file generated by asset writer#>;
// Asynchronously load the tracks of the asset you want to read.
[self.asset loadValuesAsynchronouslyForKeys:@[@"tracks"] completionHandler:^{
     // Once the tracks have finished loading, dispatch the work to the main serialization queue.
     dispatch_async(self.mainSerializationQueue, ^{
          // Due to asynchronous nature, check to see if user has already cancelled.
          if (self.cancelled)
               return;
          BOOL success = YES;
          NSError *localError = nil;
          // Check for success of loading the assets tracks.
          success = ([self.asset statusOfValueForKey:@"tracks" error:&localError] == AVKeyValueStatusLoaded);
          if (success)
          {
               // If the tracks loaded successfully, make sure that no file exists at the output path for the asset writer.
               NSFileManager *fm = [NSFileManager defaultManager];
               NSString *localOutputPath = [self.outputURL path];
               if ([fm fileExistsAtPath:localOutputPath])
                    success = [fm removeItemAtPath:localOutputPath error:&localError];
          }
          if (success)
               success = [self setupAssetReaderAndAssetWriter:&localError];
          if (success)
               success = [self startAssetReaderAndWriter:&localError];
          if (!success)
               [self readingAndWritingDidFinishSuccessfully:success withError:localError];
     });
}];
```

剩下的工作就是实现取消的处理, 并实现三个自定义方法.

### 初始化 Asset Reader 和 Writer

自定义方法`setupAssetReaderAndAssetWriter`实现了 asset Reader 和 writer 的初始化和配置. 在这个示例中, audio 先被 asset reader 解压为 Linear PCM, 然后被 asset write 压缩为 128 kbps AAC. video 被 asset reader 解压为 YUV, 然后被 asset writer 压缩为 H.264:

```text
 - (BOOL)setupAssetReaderAndAssetWriter:(NSError **)outError
 {
      // Create and initialize the asset reader.
      self.assetReader = [[AVAssetReader alloc] initWithAsset:self.asset error:outError];
      BOOL success = (self.assetReader != nil);
      if (success)
      {
           // If the asset reader was successfully initialized, do the same for the asset writer.
           self.assetWriter = [[AVAssetWriter alloc] initWithURL:self.outputURL fileType:AVFileTypeQuickTimeMovie error:outError];
           success = (self.assetWriter != nil);
      }

      if (success)
      {
           // If the reader and writer were successfully initialized, grab the audio and video asset tracks that will be used.
           AVAssetTrack *assetAudioTrack = nil, *assetVideoTrack = nil;
           NSArray *audioTracks = [self.asset tracksWithMediaType:AVMediaTypeAudio];
           if ([audioTracks count] > 0)
                assetAudioTrack = [audioTracks objectAtIndex:0];
           NSArray *videoTracks = [self.asset tracksWithMediaType:AVMediaTypeVideo];
           if ([videoTracks count] > 0)
                assetVideoTrack = [videoTracks objectAtIndex:0];

           if (assetAudioTrack)
           {
                // If there is an audio track to read, set the decompression settings to Linear PCM and create the asset reader output.
                NSDictionary *decompressionAudioSettings = @{ AVFormatIDKey : [NSNumber numberWithUnsignedInt:kAudioFormatLinearPCM] };
                self.assetReaderAudioOutput = [AVAssetReaderTrackOutput assetReaderTrackOutputWithTrack:assetAudioTrack outputSettings:decompressionAudioSettings];
                [self.assetReader addOutput:self.assetReaderAudioOutput];
                // Then, set the compression settings to 128kbps AAC and create the asset writer input.
                AudioChannelLayout stereoChannelLayout = {
                     .mChannelLayoutTag = kAudioChannelLayoutTag_Stereo,
                     .mChannelBitmap = 0,
                     .mNumberChannelDescriptions = 0
                };
                NSData *channelLayoutAsData = [NSData dataWithBytes:&stereoChannelLayout length:offsetof(AudioChannelLayout, mChannelDescriptions)];
                NSDictionary *compressionAudioSettings = @{
                     AVFormatIDKey         : [NSNumber numberWithUnsignedInt:kAudioFormatMPEG4AAC],
                     AVEncoderBitRateKey   : [NSNumber numberWithInteger:128000],
                     AVSampleRateKey       : [NSNumber numberWithInteger:44100],
                     AVChannelLayoutKey    : channelLayoutAsData,
                     AVNumberOfChannelsKey : [NSNumber numberWithUnsignedInteger:2]
                };
                self.assetWriterAudioInput = [AVAssetWriterInput assetWriterInputWithMediaType:[assetAudioTrack mediaType] outputSettings:compressionAudioSettings];
                [self.assetWriter addInput:self.assetWriterAudioInput];
           }

           if (assetVideoTrack)
           {
                // If there is a video track to read, set the decompression settings for YUV and create the asset reader output.
                NSDictionary *decompressionVideoSettings = @{
                     (id)kCVPixelBufferPixelFormatTypeKey     : [NSNumber numberWithUnsignedInt:kCVPixelFormatType_422YpCbCr8],
                     (id)kCVPixelBufferIOSurfacePropertiesKey : [NSDictionary dictionary]
                };
                self.assetReaderVideoOutput = [AVAssetReaderTrackOutput assetReaderTrackOutputWithTrack:assetVideoTrack outputSettings:decompressionVideoSettings];
                [self.assetReader addOutput:self.assetReaderVideoOutput];
                CMFormatDescriptionRef formatDescription = NULL;
                // Grab the video format descriptions from the video track and grab the first one if it exists.
                NSArray *videoFormatDescriptions = [assetVideoTrack formatDescriptions];
                if ([videoFormatDescriptions count] > 0)
                     formatDescription = (__bridge CMFormatDescriptionRef)[formatDescriptions objectAtIndex:0];
                CGSize trackDimensions = {
                     .width = 0.0,
                     .height = 0.0,
                };
                // If the video track had a format description, grab the track dimensions from there. Otherwise, grab them direcly from the track itself.
                if (formatDescription)
                     trackDimensions = CMVideoFormatDescriptionGetPresentationDimensions(formatDescription, false, false);
                else
                     trackDimensions = [assetVideoTrack naturalSize];
                NSDictionary *compressionSettings = nil;
                // If the video track had a format description, attempt to grab the clean aperture settings and pixel aspect ratio used by the video.
                if (formatDescription)
                {
                     NSDictionary *cleanAperture = nil;
                     NSDictionary *pixelAspectRatio = nil;
                     CFDictionaryRef cleanApertureFromCMFormatDescription = CMFormatDescriptionGetExtension(formatDescription, kCMFormatDescriptionExtension_CleanAperture);
                     if (cleanApertureFromCMFormatDescription)
                     {
                          cleanAperture = @{
                               AVVideoCleanApertureWidthKey            : (id)CFDictionaryGetValue(cleanApertureFromCMFormatDescription, kCMFormatDescriptionKey_CleanApertureWidth),
                               AVVideoCleanApertureHeightKey           : (id)CFDictionaryGetValue(cleanApertureFromCMFormatDescription, kCMFormatDescriptionKey_CleanApertureHeight),
                               AVVideoCleanApertureHorizontalOffsetKey : (id)CFDictionaryGetValue(cleanApertureFromCMFormatDescription, kCMFormatDescriptionKey_CleanApertureHorizontalOffset),
                               AVVideoCleanApertureVerticalOffsetKey   : (id)CFDictionaryGetValue(cleanApertureFromCMFormatDescription, kCMFormatDescriptionKey_CleanApertureVerticalOffset)
                          };
                     }
                     CFDictionaryRef pixelAspectRatioFromCMFormatDescription = CMFormatDescriptionGetExtension(formatDescription, kCMFormatDescriptionExtension_PixelAspectRatio);
                     if (pixelAspectRatioFromCMFormatDescription)
                     {
                          pixelAspectRatio = @{
                               AVVideoPixelAspectRatioHorizontalSpacingKey : (id)CFDictionaryGetValue(pixelAspectRatioFromCMFormatDescription, kCMFormatDescriptionKey_PixelAspectRatioHorizontalSpacing),
                               AVVideoPixelAspectRatioVerticalSpacingKey   : (id)CFDictionaryGetValue(pixelAspectRatioFromCMFormatDescription, kCMFormatDescriptionKey_PixelAspectRatioVerticalSpacing)
                          };
                     }
                     // Add whichever settings we could grab from the format description to the compression settings dictionary.
                     if (cleanAperture || pixelAspectRatio)
                     {
                          NSMutableDictionary *mutableCompressionSettings = [NSMutableDictionary dictionary];
                          if (cleanAperture)
                               [mutableCompressionSettings setObject:cleanAperture forKey:AVVideoCleanApertureKey];
                          if (pixelAspectRatio)
                               [mutableCompressionSettings setObject:pixelAspectRatio forKey:AVVideoPixelAspectRatioKey];
                          compressionSettings = mutableCompressionSettings;
                     }
                }
                // Create the video settings dictionary for H.264.
                NSMutableDictionary *videoSettings = (NSMutableDictionary *) @{
                     AVVideoCodecKey  : AVVideoCodecH264,
                     AVVideoWidthKey  : [NSNumber numberWithDouble:trackDimensions.width],
                     AVVideoHeightKey : [NSNumber numberWithDouble:trackDimensions.height]
                };
                // Put the compression settings into the video settings dictionary if we were able to grab them.
                if (compressionSettings)
                     [videoSettings setObject:compressionSettings forKey:AVVideoCompressionPropertiesKey];
                // Create the asset writer input and add it to the asset writer.
                self.assetWriterVideoInput = [AVAssetWriterInput assetWriterInputWithMediaType:[videoTrack mediaType] outputSettings:videoSettings];
                [self.assetWriter addInput:self.assetWriterVideoInput];
           }
      }
      return success;
 }
```

### 重编码 Asset

方法`startAssetReaderAndWriter`负责读取和写入 asset:

```text
 - (BOOL)startAssetReaderAndWriter:(NSError **)outError
 {
      BOOL success = YES;
      // Attempt to start the asset reader.
      success = [self.assetReader startReading];
      if (!success)
           *outError = [self.assetReader error];
      if (success)
      {
           // If the reader started successfully, attempt to start the asset writer.
           success = [self.assetWriter startWriting];
           if (!success)
                *outError = [self.assetWriter error];
      }

      if (success)
      {
           // If the asset reader and writer both started successfully, create the dispatch group where the reencoding will take place and start a sample-writing session.
           self.dispatchGroup = dispatch_group_create();
           [self.assetWriter startSessionAtSourceTime:kCMTimeZero];
           self.audioFinished = NO;
           self.videoFinished = NO;

           if (self.assetWriterAudioInput)
           {
                // If there is audio to reencode, enter the dispatch group before beginning the work.
                dispatch_group_enter(self.dispatchGroup);
                // Specify the block to execute when the asset writer is ready for audio media data, and specify the queue to call it on.
                [self.assetWriterAudioInput requestMediaDataWhenReadyOnQueue:self.rwAudioSerializationQueue usingBlock:^{
                     // Because the block is called asynchronously, check to see whether its task is complete.
                     if (self.audioFinished)
                          return;
                     BOOL completedOrFailed = NO;
                     // If the task isn't complete yet, make sure that the input is actually ready for more media data.
                     while ([self.assetWriterAudioInput isReadyForMoreMediaData] && !completedOrFailed)
                     {
                          // Get the next audio sample buffer, and append it to the output file.
                          CMSampleBufferRef sampleBuffer = [self.assetReaderAudioOutput copyNextSampleBuffer];
                          if (sampleBuffer != NULL)
                          {
                               BOOL success = [self.assetWriterAudioInput appendSampleBuffer:sampleBuffer];
                               CFRelease(sampleBuffer);
                               sampleBuffer = NULL;
                               completedOrFailed = !success;
                          }
                          else
                          {
                               completedOrFailed = YES;
                          }
                     }
                     if (completedOrFailed)
                     {
                          // Mark the input as finished, but only if we haven't already done so, and then leave the dispatch group (since the audio work has finished).
                          BOOL oldFinished = self.audioFinished;
                          self.audioFinished = YES;
                          if (oldFinished == NO)
                          {
                               [self.assetWriterAudioInput markAsFinished];
                          }
                          dispatch_group_leave(self.dispatchGroup);
                     }
                }];
           }

           if (self.assetWriterVideoInput)
           {
                // If we had video to reencode, enter the dispatch group before beginning the work.
                dispatch_group_enter(self.dispatchGroup);
                // Specify the block to execute when the asset writer is ready for video media data, and specify the queue to call it on.
                [self.assetWriterVideoInput requestMediaDataWhenReadyOnQueue:self.rwVideoSerializationQueue usingBlock:^{
                     // Because the block is called asynchronously, check to see whether its task is complete.
                     if (self.videoFinished)
                          return;
                     BOOL completedOrFailed = NO;
                     // If the task isn't complete yet, make sure that the input is actually ready for more media data.
                     while ([self.assetWriterVideoInput isReadyForMoreMediaData] && !completedOrFailed)
                     {
                          // Get the next video sample buffer, and append it to the output file.
                          CMSampleBufferRef sampleBuffer = [self.assetReaderVideoOutput copyNextSampleBuffer];
                          if (sampleBuffer != NULL)
                          {
                               BOOL success = [self.assetWriterVideoInput appendSampleBuffer:sampleBuffer];
                               CFRelease(sampleBuffer);
                               sampleBuffer = NULL;
                               completedOrFailed = !success;
                          }
                          else
                          {
                               completedOrFailed = YES;
                          }
                     }
                     if (completedOrFailed)
                     {
                          // Mark the input as finished, but only if we haven't already done so, and then leave the dispatch group (since the video work has finished).
                          BOOL oldFinished = self.videoFinished;
                          self.videoFinished = YES;
                          if (oldFinished == NO)
                          {
                               [self.assetWriterVideoInput markAsFinished];
                          }
                          dispatch_group_leave(self.dispatchGroup);
                     }
                }];
           }
           // Set up the notification that the dispatch group will send when the audio and video work have both finished.
           dispatch_group_notify(self.dispatchGroup, self.mainSerializationQueue, ^{
                BOOL finalSuccess = YES;
                NSError *finalError = nil;
                // Check to see if the work has finished due to cancellation.
                if (self.cancelled)
                {
                     // If so, cancel the reader and writer.
                     [self.assetReader cancelReading];
                     [self.assetWriter cancelWriting];
                }
                else
                {
                     // If cancellation didn't occur, first make sure that the asset reader didn't fail.
                     if ([self.assetReader status] == AVAssetReaderStatusFailed)
                     {
                          finalSuccess = NO;
                          finalError = [self.assetReader error];
                     }
                     // If the asset reader didn't fail, attempt to stop the asset writer and check for any errors.
                     if (finalSuccess)
                     {
                          finalSuccess = [self.assetWriter finishWriting];
                          if (!finalSuccess)
                               finalError = [self.assetWriter error];
                     }
                }
                // Call the method to handle completion, and pass in the appropriate parameters to indicate whether reencoding was successful.
                [self readingAndWritingDidFinishSuccessfully:finalSuccess withError:finalError];
           });
      }
      // Return success here to indicate whether the asset reader and writer were started successfully.
      return success;
 }
```

在重编码过程中, 为了提升性能, 音频处理和视频处理在两个不同队列中进行. 但这两个队列在一个 dispatchGroup 中, 当每个队列的任务都完成后, 会调用`readingAndWritingDidFinishSuccessfully`,

### 处理编码结果

对重编码的结果进行处理并同步到 UI:

```text
- (void)readingAndWritingDidFinishSuccessfully:(BOOL)success withError:(NSError *)error
{
     if (!success)
     {
          // If the reencoding process failed, we need to cancel the asset reader and writer.
          [self.assetReader cancelReading];
          [self.assetWriter cancelWriting];
          dispatch_async(dispatch_get_main_queue(), ^{
               // Handle any UI tasks here related to failure.
          });
     }
     else
     {
          // Reencoding was successful, reset booleans.
          self.cancelled = NO;
          self.videoFinished = NO;
          self.audioFinished = NO;
          dispatch_async(dispatch_get_main_queue(), ^{
               // Handle any UI tasks here related to success.
          });
     }
}
```

### 取消重编码

使用多个串行队列, 可以很轻松的取消对 asset 的重编码. 可以将下面的代码与 UI 上的 "取消" 按钮关联起来:

```text
- (void)cancel
{
     // Handle cancellation asynchronously, but serialize it with the main queue.
     dispatch_async(self.mainSerializationQueue, ^{
          // If we had audio data to reencode, we need to cancel the audio work.
          if (self.assetWriterAudioInput)
          {
               // Handle cancellation asynchronously again, but this time serialize it with the audio queue.
               dispatch_async(self.rwAudioSerializationQueue, ^{
                    // Update the Boolean property indicating the task is complete and mark the input as finished if it hasn't already been marked as such.
                    BOOL oldFinished = self.audioFinished;
                    self.audioFinished = YES;
                    if (oldFinished == NO)
                    {
                         [self.assetWriterAudioInput markAsFinished];
                    }
                    // Leave the dispatch group since the audio work is finished now.
                    dispatch_group_leave(self.dispatchGroup);
               });
          }

          if (self.assetWriterVideoInput)
          {
               // Handle cancellation asynchronously again, but this time serialize it with the video queue.
               dispatch_async(self.rwVideoSerializationQueue, ^{
                    // Update the Boolean property indicating the task is complete and mark the input as finished if it hasn't already been marked as such.
                    BOOL oldFinished = self.videoFinished;
                    self.videoFinished = YES;
                    if (oldFinished == NO)
                    {
                         [self.assetWriterVideoInput markAsFinished];
                    }
                    // Leave the dispatch group, since the video work is finished now.
                    dispatch_group_leave(self.dispatchGroup);
               });
          }
          // Set the cancelled Boolean property to YES to cancel any work on the main queue as well.
          self.cancelled = YES;
     });
}
```

## AVOutputSettingsAssistant介绍 <a id="avoutputsettingsassistant&#x4ECB;&#x7ECD;"></a>

[AVOutputSettingsAssistant](https://developer.apple.com/reference/avfoundation/avoutputsettingsassistant) 类的功能是为 asset reader 或 writer 创建设置信息. 这将简化初始化过程, 特别是在对一个高帧率的 H264 视频进行参数设置时. 下面的代码是`AVOutputSettingsAssistant`的使用示例:

```text
AVOutputSettingsAssistant *outputSettingsAssistant = [AVOutputSettingsAssistant outputSettingsAssistantWithPreset:<some preset>];
CMFormatDescriptionRef audioFormat = [self getAudioFormat];

if (audioFormat != NULL)
    [outputSettingsAssistant setSourceAudioFormat:(CMAudioFormatDescriptionRef)audioFormat];

CMFormatDescriptionRef videoFormat = [self getVideoFormat];

if (videoFormat != NULL)
    [outputSettingsAssistant setSourceVideoFormat:(CMVideoFormatDescriptionRef)videoFormat];

CMTime assetMinVideoFrameDuration = [self getMinFrameDuration];
CMTime averageFrameDuration = [self getAvgFrameDuration]

[outputSettingsAssistant setSourceVideoAverageFrameDuration:averageFrameDuration];
[outputSettingsAssistant setSourceVideoMinFrameDuration:assetMinVideoFrameDuration];

AVAssetWriter *assetWriter = [AVAssetWriter assetWriterWithURL:<some URL> fileType:[outputSettingsAssistant outputFileType] error:NULL];
AVAssetWriterInput *audioInput = [AVAssetWriterInput assetWriterInputWithMediaType:AVMediaTypeAudio outputSettings:[outputSettingsAssistant audioSettings] sourceFormatHint:audioFormat];
AVAssetWriterInput *videoInput = [AVAssetWriterInput assetWriterInputWithMediaType:AVMediaTypeVideo outputSettings:[outputSettingsAssistant videoSettings] sourceFormatHint:videoFormat];
```

