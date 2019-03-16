---
layout: post
author: czb1n
title:  "iOS基于AVPlayer自定义播放器"
date:   2016-09-13 15:52:49
categories: [iOS]
tags: [iOS, Player]
---

简介：用AVPlayer实现一个播放器

``` Swift
@interface PlayerView : UIView

@property (nonatomic, strong) AVPlayer *avPlayer;

@end
```

### 1. 配置

AVPlayer本身是无法显示视频的，首先要把AVPlayer添加到AVPlayerLayer。

> @abstract Indicates the instance of AVPlayer for which the AVPlayerLayer displays visual output

在PlayerView中重写`+ (Class)layerClass;` 方法，使得PlayerView的 underlying layer 为 AVPlayerLayer：

``` Swift
 + (Class)layerClass
{
    return [AVPlayerLayer class];
}
```

然后初始化AVPlayer：

``` Swift
 + (instancetype)playerWithPlayerItem:(AVPlayerItem *)item;
 + (instancetype)initWithPlayerItem:(AVPlayerItem *)item;
```

AVPlayerItem可以根据AVAsset创建，可以创建在线资源item，也可以是本地资源item。
创建AVPlayer之后就添加到AVPlayerLayer。

``` Swift
[(AVPlayerLayer *)self.layer setPlayer:self.avPlayer];
```
### 2. 使用
播放和暂停。

``` Swift
 + (void)play;
 + (void)pause;
```

音量控制。

``` Swift
@property (nonatomic) float volume;
@property (nonatomic, getter=isMuted) BOOL muted;
```

切换播放Item。

``` Swift
- (void)replaceCurrentItemWithPlayerItem:(nullable AVPlayerItem *)item;
```

获取播放状态和缓存进度，这需要监听AVPlayer中当前播放的item中的属性变化。

``` Swift
[self.avPlayer.currentItem addObserver:self
                            forKeyPath:@"status"
                               options:NSKeyValueObservingOptionNew
                               context:nil];
    
[self.avPlayer.currentItem addObserver:self
                            forKeyPath:@"loadedTimeRanges"
                               options:NSKeyValueObservingOptionNew
                               context:nil];
```

``` Swift
- (void)observeValueForKeyPath:(NSString *)keyPath ofObject:(id)object change:(NSDictionary *)change context:(void *)context
{
    AVPlayerItem *playerItem = (AVPlayerItem *)object;
    
    if ([keyPath isEqualToString:@"status"]) {
        if ([playerItem status] == AVPlayerStatusReadyToPlay) {
            // 正常播放
        }
        else if ([playerItem status] == AVPlayerStatusFailed) {
            // 播放失败
            // show retry
        }
        else {
            // 未知错误
            // show retry
        }
    }
    else if ([keyPath isEqualToString:@"loadedTimeRanges"]) {
        NSArray *loadedTimeRanges = [playerItem loadedTimeRanges];
        // 获取缓冲区域
        CMTimeRange timeRange = [loadedTimeRanges.firstObject CMTimeRangeValue];
        float startSeconds = CMTimeGetSeconds(timeRange.start);
        float durationSeconds = CMTimeGetSeconds(timeRange.duration);
        // 计算缓冲总进度
        NSTimeInterval result = startSeconds + durationSeconds;
    }
}
```
