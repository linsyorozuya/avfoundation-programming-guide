# 播放 Assets

`AVPlayer`对象可以控制资源的播放, 在播放过程中, 可以使用`AVPlayerItem`对象管理资源的播放状态, `AVPlayerItemTrack`对象则可以用来管理一个独立轨道的播放状态. 要展示一段视频, 可以使用`AVPlayerLayer`对象.

`AVPlayer`对象是一个用来管理资源播放的控制中心, 比如控制播放的开始和停止, 定位到一个指定的时间点进行播放等等. 使用单个`AVPlayer`对象来播放单个资源, 使用`AVQueuePlayer`对象顺序播放多个资源 \(`AVQueuePlayer`是`AVPlayer`的子类\). 在 OS X 上, 还可以使用 AVKit 框架中的`AVPlayerView`类在 view 上直接播放一段内容.

`AVPlayer`对象提供了资源播放状态相关的信息, 你可以根据播放状态的变化来同步更新 UI. 通常会将一个`AVPlayer`对象输出到一个特定的`Core Animation layer`上进行展示 \(`AVPlayerLayer`对象或`AVSynchronizedLayer`对象\). 更多 layer 相关的信息, 参考 [_Core Animation Programming Guide_](https://developer.apple.com/library/prerelease/content/documentation/Cocoa/Conceptual/CoreAnimation_guide/Introduction/Introduction.html#//apple_ref/doc/uid/TP40004514).

> 可以使用一个`AVPlayer`对象创建多个`AVPlayerLayer`实例, 但是最后一个被创建的 layer 才会在屏幕上展示播放的内容.

虽然我们的最终目的是播放某个资源, 但并不需要直接向`AVPlayer`对象提供要播放的资源, 而是提供一个`AVPlayerItem`对象. 一个 player item 管理与其相绑定的资源 \(asset\) 的呈现状态. 个 player item 包含多个 player item tracks\( `AVPlayerItemTrack`对象\). 这些变量直接的关系如下图:

![](https://developer.apple.com/library/prerelease/content/documentation/AudioVideo/Conceptual/AVFoundationPG/Art/avplayerLayer_2x.png)

这种抽象意味着你可以使用不同的 player 来播放给定的 asset, 但是每个 player 又有各自的渲染方式. 下图展示了其中一种可能性, 两个播放器以不同的设置来播放同一个 asset. 通过使用 item tracks, 可以在播放过程中禁用某些 track\(比如禁用音频 track, 即静音\).

![](https://developer.apple.com/library/prerelease/content/documentation/AudioVideo/Conceptual/AVFoundationPG/Art/playerObjects_2x.png)

可以使用已存在的 asset 对 player 进行初始化, 也可以直接使用 URL 初始化以便于用来播放某个指定位置的资源. 使用`AVAsset`时要注意, 初始化一个 player item 并不意味着就可以直接播放了. 你需要使用 KVO 监听 item 的`status`属性来判断是否为可播放状态.

## 处理不同类型的Asset <a id="&#x5904;&#x7406;&#x4E0D;&#x540C;&#x7C7B;&#x578B;&#x7684;asset"></a>

asset 的播放配置取决于 asset 的类型. 总的来说, asset 有两个主要类型: 基于文件的 asset\(file-based, 比如本地文件\), 以及基于流的 asset\(stream-based, 比如 HLS 流媒体\).

**要播放一个 file-based asset**, 有以下一些步骤:

* 创建一个 [`AVURLAsset`](https://developer.apple.com/reference/avfoundation/avurlasset)
* 根据 asset 创建一个`AVPlayerItem`对象
* 将 item 关联到一个 AVPlayer 对象上
* 使用 KVO 监听 item 的`status`属性, 当播放准备就绪后, 开始播放

**要播放一个 HTTP 流媒体**, 使用 URL 初始化一个`AVPlayerItem`对象 \(不能直接创建一个 AVAsset 对象用来代表 HTTP 直播流媒体\).

```text
NSURL *url = [NSURL URLWithString:@"<#Live stream URL#>];
// You may find a test stream at <http://devimages.apple.com/iphone/samples/bipbop/bipbopall.m3u8>.
self.playerItem = [AVPlayerItem playerItemWithURL:url];
[playerItem addObserver:self forKeyPath:@"status" options:0 context:&ItemStatusContext];
self.player = [AVPlayer playerWithPlayerItem:playerItem];
```

当把一个 player item 关联到一个 player 对象时, player 就开始准备播放了. 当准备就绪后, player item 会创建`AVAsset`和`AVAssetTrack`对象.

要获取流媒体的时长, 可以监听 player item 的`duration`属性.

如果只需要简单的播放一个直播流, 可以更快捷的使用 URL 来创建 player:

```text
self.player = [AVPlayer playerWithURL:<#Live stream URL#>];
[player addObserver:self forKeyPath:@"status" options:0 context:&PlayerStatusContext];
```

与使用 AVAsset 和 AVPlayerItem 相同, 初始化一个 player 并不意味了就可以直接播放了, 还是需要监听 player 的`status`属性, 当其变为`AVPlayerStatusReadyToPlay`时才能开始播放. 也可以监听`currentItem`属性来访问为流媒体创建的 AVPlayerItem.

**如果不确定是何种类型的 URL**, 参照以下步骤:

1. 尝试根据 URL 初始化一个`AVURLAsset`对象, 然后加载其`tracks`属性.
2. 如果上一步失败, 直接根据 URL 创建 AVPlayerItem 对象, 监听 player 的 status 属性判断是否可以播放

## 播放单个AVPlayerItem <a id="&#x64AD;&#x653E;&#x5355;&#x4E2A;avplayeritem"></a>

可以向 player 发送`play`消息来开始播放:

```text
- (IBAction)play:sender {
    [player play];
}
```

除了简单的播放, 你还可以控制一些播放的细节. 比如播放速率和播放状态.

### 播放速率

可以通过设置 player 的`rate`属性改变播放速率:

```text
aPlayer.rate = 0.5;
aPlayer.rate = 2.0;
```

rate 为 1.0 表示正常速率, 设置为 0 表示暂停播放 \(同`pause`\).

可以将 rate 设置为一个负值来进行倒放. 通过 playerItem 的以下属性可以判断是否支持倒放:`canPlayReverse`\(支持 rate -1.0\), `canPlaySlowReverse`\(支持 rate - 1.0 到 0.0\), `canPlayFastReverse`\(支持 rate 小于 -1.0\).

### 定位播放点

要定位到指定时间点进行播放, 可以使用 [seekToTime:](https://developer.apple.com/reference/avfoundation/avplayer/1385953-seek) 如下:

```text
CMTime fiveSecondsIn = CMTimeMake(5, 1);
[player seekToTime:fiveSecondsIn];
```

`seekToTime:`方法牺牲了精确度来优化性能, 如果需要精确定位, 使用 [seekToTime:toleranceBefore:toleranceAfter:](https://developer.apple.com/reference/avfoundation/avplayer/1387741-seek) 方法.

```text
CMTime fiveSecondsIn = CMTimeMake(5, 1);
[player seekToTime:fiveSecondsIn toleranceBefore:kCMTimeZero toleranceAfter:kCMTimeZero];
```

将容忍值设置为 0 意味着框架会解码大量的数据来保证精确度. 确保只有在真正需要的时候才会如此设置.

播放完毕之后, 播放点会被设置为 item 的结束点. 要将播放点重新设置到 item 开头, 可以注册接收`AVPlayerItemDidPlayToEndTimeNotification`通知, 在处理通知的方法中, 调用`seekToTime:`并传入参数 kCMTimeZero.

```text
// Register with the notification center after creating the player item.
    [[NSNotificationCenter defaultCenter]
        addObserver:self
        selector:@selector(playerItemDidReachEnd:)
        name:AVPlayerItemDidPlayToEndTimeNotification
        object:<#The player item#>];

- (void)playerItemDidReachEnd:(NSNotification *)notification {
    [player seekToTime:kCMTimeZero];
}
```

## 播放多个AVPlayerItem <a id="&#x64AD;&#x653E;&#x591A;&#x4E2A;avplayeritem"></a>

可以使用`AVQueuePlayer`对象顺序播放多个 AVPlayerItem. `AVQueuePlayer`类是`AVPlayer`的一个子类. 可以使用一个 player items 的数组来初始化一个 queue player:

```text
NSArray *items = <#An array of player items#>;
AVQueuePlayer *queuePlayer = [[AVQueuePlayer alloc] initWithItems:items];
```

接下来可以像使用 AVPlayer 对象一样发送`play`消息开始播放 player items. queue player 依次播放每个 item, 如果需要跳到下一个 item, 可以向 queue player 发送`advanceToNextItem`消息.

可以使用`insertItem:afterItem:, removeItem:`以及`removeAllItems`对播放队列进行操作. 添加一个新的 item 时, 需要使用`canInsertItem:afterItem:` 第二个参数可以传递`nil`用来判断该 item 是否可以被添加到队列中.

```text
AVPlayerItem *anItem = <#Get a player item#>;
if ([queuePlayer canInsertItem:anItem afterItem:nil]) {
    [queuePlayer insertItem:anItem afterItem:nil];
}
```

## 监听播放 <a id="&#x76D1;&#x542C;&#x64AD;&#x653E;"></a>

可以监听包括播放状态以及正在被播放的 player item 在内的许多方面的信息. 当状态变化不可控时, 这一点十分有用. 例如:

* 当用户切换的其他应用时, player 的 rate 将会降至 0.0
* 如果正在播放远程媒体, player 的`loadedTimeRanges`和`loadedTimeRanges`属性将会随着数据的加载而变化
* 播放 HTTP 流媒体时, 一个 player 的`currentItem`属性会被自动创建.
* 播放 HTTP 流媒体时, 一个 player 的`tracks`属性可能会随着播放而变化
* 由于某些原因导致播放失败时, 一个 player 或者 item 的`status`属性会随之变化

可以使用 KVO 来监听这些属性的变化.

### 响应播放状态的改变

当一个 player 或者 item 的状态发生改变时, 会发出对应 KVO 变化的 notification. 如果一个资源因为某些原因不能播放, status 会变为`AVPlayerStatusFailed`或者`AVPlayerItemStatusFailed`. 这种情况下, `error`对象会包含相关的错误信息.

AV Foundation 并不会指定 notification 发出的线程. 如果需要更新 UI, 你必须手动切换到主线程. 下面是使用`dispatch_async`切换到主线程的示例代码:

```text
- (void)observeValueForKeyPath:(NSString *)keyPath ofObject:(id)object
                        change:(NSDictionary *)change context:(void *)context {

    if (context == <#Player status context#>) {
        AVPlayer *thePlayer = (AVPlayer *)object;
        if ([thePlayer status] == AVPlayerStatusFailed) {
            NSError *error = [<#The AVPlayer object#> error];
            // Respond to error: for example, display an alert sheet.
            return;
        }
        // Deal with other status change if appropriate.
    }
    // Deal with other change notifications if appropriate.
    [super observeValueForKeyPath:keyPath ofObject:object
           change:change context:context];
    return;
}
```

### 跟踪可视化内容的就绪状态

可是监听`AVPlayerLayer`对象的`readyForDisplay`属性判断当前是否可以展示可视化的内容. 特别是当你只需要在有可视化内容时才将 layer 插入到视图层级中的情况.

### 跟踪播放时间

要跟踪 AVPlayer 对象当前的播放位置, 可以使用`addPeriodicTimeObserverForInterval:queue:usingBlock:`或者`addBoundaryTimeObserverForTimes:queue:usingBlock:`.

当你需要进行更新播放时间, 或者其他 UI 同步操作时可能会用到这些方法.

* `addPeriodicTimeObserverForInterval:queue:usingBlock:`, block 会在指定的时间间隔被调用, 哪怕时间发生了跳动, 播放开始或结束.
* `addBoundaryTimeObserverForTimes:queue:usingBlock:`, 传递一个包含了`CMTime`的`NSValue`对象的数组. 当播放到数组中的时间点时, block 会被调用.

这两个方法都会返回一个不可见的观察者对象, 必须对其保持强引用. 当需要注销这个观察者时, 可以使用`removeTimeObserver:`.

对于这两个方法, AV Foundation 并不能确保 blcok 都会被按时调用. 如果有上一个 block 尚未执行完毕, 那么本次将不会调用 block. 所以不要在 block 中执行复杂的操作.

```text
// Assume a property: @property (strong) id playerObserver;

Float64 durationSeconds = CMTimeGetSeconds([<#An asset#> duration]);
CMTime firstThird = CMTimeMakeWithSeconds(durationSeconds/3.0, 1);
CMTime secondThird = CMTimeMakeWithSeconds(durationSeconds*2.0/3.0, 1);
NSArray *times = @[[NSValue valueWithCMTime:firstThird], [NSValue valueWithCMTime:secondThird]];

self.playerObserver = [<#A player#> addBoundaryTimeObserverForTimes:times queue:NULL usingBlock:^{

    NSString *timeDescription = (NSString *)
        CFBridgingRelease(CMTimeCopyDescription(NULL, [self.player currentTime]));
    NSLog(@"Passed a boundary at %@", timeDescription);
}];
```

### 播放结束

可以注册`AVPlayerItemDidPlayToEndTimeNotification`通知来监听 player item 的结束.

```text
[[NSNotificationCenter defaultCenter] addObserver:<#The observer, typically self#>
                                         selector:@selector(<#The selector name#>)
                                             name:AVPlayerItemDidPlayToEndTimeNotification
                                           object:<#A player item#>];
```

## 使用AVPlayerLayer播放视频文件 <a id="&#x4F7F;&#x7528;avplayerlayer&#x64AD;&#x653E;&#x89C6;&#x9891;&#x6587;&#x4EF6;"></a>

下面的代码简要展示了如何使用`AVPlayer`对象播放视频文件, 包括:

* 使用`AVPlayerLayer`
* 创建`AVPlayer`对象
* 创建基于 file-based asset 的`AVPlayerItem`对象, 并监听其状态
* 响应播放状态的变化, 同步改变播放按钮的可用状态
* 播放 item

> 提示: 为了展示核心代码, 这份示例省略了某些内容, 比如内存管理和通知的移除等. 使用 AV Foundation 之前, 你最好已经拥有 Cocoa 框架的使用经验.

### Player View

要播放一个 asset 的可视部分, 你需要一个包含`AVPlayerLayer`对象的 view, 用来接收`AVPlayer`对象的输出. 可以简单的定义一个 UIView 的子类来实现这一功能:

```text
#import <UIKit/UIKit.h>
#import <AVFoundation/AVFoundation.h>

@interface PlayerView : UIView
@property (nonatomic) AVPlayer *player;
@end

@implementation PlayerView
+ (Class)layerClass {
    return [AVPlayerLayer class];
}
- (AVPlayer*)player {
    return [(AVPlayerLayer *)[self layer] player];
}
- (void)setPlayer:(AVPlayer *)player {
    [(AVPlayerLayer *)[self layer] setPlayer:player];
}
@end
```

### View Controller

假设有一个简单的 View Controller 声明如下:

```text
@class PlayerView;
@interface PlayerViewController : UIViewController

@property (nonatomic) AVPlayer *player;
@property (nonatomic) AVPlayerItem *playerItem;
@property (nonatomic, weak) IBOutlet PlayerView *playerView;
@property (nonatomic, weak) IBOutlet UIButton *playButton;
- (IBAction)loadAssetFromFile:sender;
- (IBAction)play:sender;
- (void)syncUI;
@end
```

`syncUI`方法用来根据 player 的状态同步按钮的可用状态:

```text
- (void)syncUI {
    if ((self.player.currentItem != nil) &&
        ([self.player.currentItem status] == AVPlayerItemStatusReadyToPlay)) {
        self.playButton.enabled = YES;
    }
    else {
        self.playButton.enabled = NO;
    }
}
```

可以在 view controller 的`viewDidLoad`方法中就调用`syncUI`同步按钮的状态:

```text
- (void)viewDidLoad {
    [super viewDidLoad];
    [self syncUI];
}
```

### 创建 Asset

使用`AVURLAsset`根据 URL 创建 asset.\(下面的代码假设项目中包含了一个视频资源\)

```text
- (IBAction)loadAssetFromFile:sender {

    NSURL *fileURL = [[NSBundle mainBundle]
        URLForResource:<#@"VideoFileName"#> withExtension:<#@"extension"#>];

    AVURLAsset *asset = [AVURLAsset URLAssetWithURL:fileURL options:nil];
    NSString *tracksKey = @"tracks";

    [asset loadValuesAsynchronouslyForKeys:@[tracksKey] completionHandler:
     ^{
         // The completion block goes here.
     }];
}
```

在 completion block 中创建 `AVPlayerItem`, 并设置 player view 的 player.

```text
// Define this constant for the key-value observation context.
static const NSString *ItemStatusContext;

// Completion handler block.
 dispatch_async(dispatch_get_main_queue(),
    ^{
        NSError *error;
        AVKeyValueStatus status = [asset statusOfValueForKey:tracksKey error:&error];

        if (status == AVKeyValueStatusLoaded) {
            self.playerItem = [AVPlayerItem playerItemWithAsset:asset];
             // ensure that this is done before the playerItem is associated with the player
            [self.playerItem addObserver:self forKeyPath:@"status"
                        options:NSKeyValueObservingOptionInitial context:&ItemStatusContext];
            [[NSNotificationCenter defaultCenter] addObserver:self
                                                      selector:@selector(playerItemDidReachEnd:)
                                                          name:AVPlayerItemDidPlayToEndTimeNotification
                                                        object:self.playerItem];
            self.player = [AVPlayer playerWithPlayerItem:self.playerItem];
            [self.playerView setPlayer:self.player];
        }
        else {
            // You should deal with the error appropriately.
            NSLog(@"The asset's tracks were not loaded:\n%@", [error localizedDescription]);
        }
    });
```

## 响应 Player Item 的状态改变

```text
- (void)observeValueForKeyPath:(NSString *)keyPath ofObject:(id)object
                        change:(NSDictionary *)change context:(void *)context {

    if (context == &ItemStatusContext) {
        dispatch_async(dispatch_get_main_queue(),
                       ^{
                           [self syncUI];
                       });
        return;
    }
    [super observeValueForKeyPath:keyPath ofObject:object
           change:change context:context];
    return;
}
```

### 播放 Item

```text
- (IBAction)play:sender {
    [player play];
}
```

item 只被播放一次, 播放结束后, 播放点会被设置为 item 的结束点, 这样下一次调用 play 方法将会失效. 要将播放点设置到 item 的起始处, 参考如下代码:

```text
// Register with the notification center after creating the player item.
    [[NSNotificationCenter defaultCenter]
        addObserver:self
        selector:@selector(playerItemDidReachEnd:)
        name:AVPlayerItemDidPlayToEndTimeNotification
        object:[self.player currentItem]];

- (void)playerItemDidReachEnd:(NSNotification *)notification {
    [self.player seekToTime:kCMTimeZero];
}
```

