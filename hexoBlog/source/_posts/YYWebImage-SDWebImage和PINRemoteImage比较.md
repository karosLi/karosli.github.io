---
title: 'YYWebImage,SDWebImage和PINRemoteImage比较'
date: 2018-02-27 10:18:53
tags: WebImage
categories: WebImage
---

# YYWebImage,SDWebImage和PINRemoteImage比较

### 共同的特性
1. 以类别 api 下载远程图片。
2. 图片缓存
3. 图片提前解码
4. 其他

### 图片框架比较

#### 图片后处理
根据下面的比较，可以看出图片后处理方面，PINRemoteImage > YYWebImage > SDWebImage

* YYWebImage:
    * 支持不带标记的后处理。
    
    ```
    /**
     Set the view's `image` with a specified URL.
     
     @param imageURL    The image url (remote or local file path).
     @param placeholder he image to be set initially, until the image request finishes.
     @param options     The options to use when request the image.
     @param manager     The manager to create image request operation.
     @param progress    The block invoked (on main thread) during image request.
     @param transform   The block invoked (on background thread) to do additional image process.
     @param completion  The block invoked (on main thread) when image request completed.
     */
    - (void)yy_setImageWithURL:(nullable NSURL *)imageURL
                   placeholder:(nullable UIImage *)placeholder
                       options:(YYWebImageOptions)options
                       manager:(nullable YYWebImageManager *)manager
                      progress:(nullable YYWebImageProgressBlock)progress
                     transform:(nullable YYWebImageTransformBlock)transform
                    completion:(nullable YYWebImageCompletionBlock)completion;
    ```

* SDWebImage: 不支持图片后处理。
    
* PINRemoteImage:
    * 支持带标记的图片后处理。对于同一张图片，当需要不同的后处理方式时（a 界面需要正圆角，b 界面需要小幅度的圆角），尤为有用。
    
    ```
    /**
 Set placeholder on view and retrieve the image from the given URL, process it using the passed in processor block and set result on view. Call completion after image has been fetched, processed and set on view.
 
 @param url NSURL to fetch from.
 @param placeholderImage PINImage to set on the view while the image at URL is being retrieved.
 @param processorKey NSString key to uniquely identify processor. Used in caching.
 @param processor PINRemoteImageManagerImageProcessor processor block which should return the processed image.
 @param completion Called when url has been retrieved and set on view.
 */
- (void)pin_setImageFromURL:(nullable NSURL *)url placeholderImage:(nullable PINImage *)placeholderImage processorKey:(nullable NSString *)processorKey processor:(nullable PINRemoteImageManagerImageProcessor)processor completion:(nullable PINRemoteImageManagerImageCompletion)completion;
    ```
    
    
<!--more-->

#### 图片格式支持
根据下面的比较，可以看出图片格式支持方面，YYWebImage = SDWebImage = PINRemoteImage

另外对于 WebP 的支持，需要下载 google 的 libwebp pod，这就需要先配置命令行代理了，才能安装此 pod，命令行代理的配置就不在此说明了。

而 YYImage 是提前先把编译 webp 的源码，并打包成了 framework，直接引入到了项目里了，避免了配置代理的繁琐工作。编译 webp 成 framework 可以参考[此文](http://www.cnblogs.com/damiao/p/4562186.html)。

* YYWebImage:
    * GIF: [YYImage](https://github.com/ibireme/YYImage)
    * WebP: [libwebp](https://chromium.googlesource.com/webm/libwebp)

* SDWebImage:
    * GIF: [FLAnimatedImage](https://github.com/Flipboard/FLAnimatedImage)
    * WebP: [libwebp](https://chromium.googlesource.com/webm/libwebp)

* PINRemoteImage:
    * GIF: [FLAnimatedImage](https://github.com/Flipboard/FLAnimatedImage)
    * WebP: [libwebp](https://chromium.googlesource.com/webm/libwebp)

#### 图片解码控制:
根据下面的比较，可以看出图片解码控制方面，PINRemoteImage > YYWebImage > SDWebImage

* YYWebImage: 
    * 下载完图片，会自动解码，可根据参数  `YYWebImageOptionIgnoreImageDecoding` 来控制不解码。
    
        ```
        代码位置：
        YYWebImageOperation(connectionDidFinishLoading:)
        
        BOOL shouldDecode = (self.options & YYWebImageOptionIgnoreImageDecoding) == 0;
        BOOL allowAnimation = (self.options & YYWebImageOptionIgnoreAnimatedImage) == 0;
        UIImage *image;
        BOOL hasAnimation = NO;
        if (allowAnimation) {
            image = [[YYImage alloc] initWithData:self.data scale:[UIScreen mainScreen].scale];
            if (shouldDecode) image = [image yy_imageByDecoded];
            if ([((YYImage *)image) animatedImageFrameCount] > 1) {
                hasAnimation = YES;
            }
        } else {
            YYImageDecoder *decoder = [YYImageDecoder decoderWithData:self.data scale:[UIScreen mainScreen].scale];
            image = [decoder frameAtIndex:0 decodeForDisplay:shouldDecode].image;
        }                 
        ```

    * 当我们在传入 YYWebImageOptionIgnoreImageDecoding，是期望图片不被解码，确实图片在下载完成的时候，不会被解码，但是当把图片存入到内存缓存的时候，图片还是一样会被解码一次并存入到缓存。代码 `[_cache setImage:image imageData:data forKey:_cacheKey withType:YYImageCacheTypeAll];` 会针对没有解码的图片，进行一次解码，并存入缓存。所以对于大图的解码，还是会占用很大的内存。
    
      ```
      代码位置：
      YYWebImageOperation(_didReceiveImageFromWeb:)
            
      - (void)_didReceiveImageFromWeb:(UIImage *)image {
        @autoreleasepool {
            [_lock lock];
            if (![self isCancelled]) {
                if (_cache) {
                    if (image || (_options & YYWebImageOptionRefreshImageCache)) {
                        NSData *data = _data;
                        dispatch_async([YYWebImageOperation _imageQueue], ^{
                            // 保存图片到缓存中
                            [_cache setImage:image imageData:data forKey:_cacheKey withType:YYImageCacheTypeAll];
                        });
                    }
                }
                _data = nil;
                NSError *error = nil;
                if (!image) {
                    error = [NSError errorWithDomain:@"com.ibireme.image" code:-1 userInfo:@{ NSLocalizedDescriptionKey : @"Web image decode fail." }];
                    if (_options & YYWebImageOptionIgnoreFailedURL) {
                        if (URLBlackListContains(_request.URL)) {
                            error = [NSError errorWithDomain:NSURLErrorDomain code:NSURLErrorFileDoesNotExist userInfo:@{ NSLocalizedDescriptionKey : @"Failed to load URL, blacklisted." }];
                        } else {
                            URLInBlackListAdd(_request.URL);
                        }
                    }
                }
                if (_completion) _completion(image, _request.URL, YYWebImageFromRemote, YYWebImageStageFinished, error);
                [self _finish];
            }
            [_lock unlock];
        }
    }
      
      ```
  
  
  
       ```
       代码位置：
       YYImageCache(setImage:imageData:)
       
       - (void)setImage:(UIImage *)image imageData:(NSData *)imageData forKey:(NSString *)key withType:(YYImageCacheType)type {
           if (!key || (image == nil && imageData.length == 0)) return;
           
           __weak typeof(self) _self = self;
           if (type & YYImageCacheTypeMemory) { // add to memory cache
               if (image) {
                   if (image.yy_isDecodedForDisplay) {
                       [_memoryCache setObject:image forKey:key withCost:[_self imageCost:image]];
                   } else {
                       // 如果没有解码过，解码图片，并保存到内存缓存 
                       dispatch_async(YYImageCacheDecodeQueue(), ^{
                           __strong typeof(_self) self = _self;
                           if (!self) return;
                           [self.memoryCache setObject:[image yy_imageByDecoded] forKey:key withCost:[self imageCost:image]];
                       });
                   }
               } else if (imageData) {
                   dispatch_async(YYImageCacheDecodeQueue(), ^{
                       __strong typeof(_self) self = _self;
                       if (!self) return;
                       UIImage *newImage = [self imageFromData:imageData];
                       [self.memoryCache setObject:newImage forKey:key withCost:[self imageCost:newImage]];
                   });
               }
           }
           if (type & YYImageCacheTypeDisk) { // add to disk cache
               if (imageData) {
                   if (image) {
                       [YYDiskCache setExtendedData:[NSKeyedArchiver archivedDataWithRootObject:@(image.scale)] toObject:imageData];
                   }
                   [_diskCache setObject:imageData forKey:key];
               } else if (image) {
                   dispatch_async(YYImageCacheIOQueue(), ^{
                       __strong typeof(_self) self = _self;
                       if (!self) return;
                       NSData *data = [image yy_imageDataRepresentation];
                       [YYDiskCache setExtendedData:[NSKeyedArchiver archivedDataWithRootObject:@(image.scale)] toObject:data];
                       [self.diskCache setObject:data forKey:key];
                   });
               }
           }
       }
    ```
    
    
* SDWebImage: 下载完图片，会自动解码，没有 api 暴露去控制是否解码下载的图片。所以对于大图的解码，还是会占用很大的内存。
    
* PINRemoteImage: 
    * 下载完图片，会自动解码，可根据参数  `PINRemoteImageManagerDownloadOptionsSkipDecode` 来控制不解码。并且在不解码时缓存不解码的图片数据，解码时缓存解码的图片数据。可以根据场景来决定是否需要提前解码图片。
    
    ```
    代码位置：
    PINRemoteImageManager(materializeAndCacheObject:(id)object
                      cacheInDisk:(NSData *)diskData
                   additionalCost:(NSUInteger)additionalCost
                              url:(NSURL *)url
                              key:(NSString *)key
                          options:(PINRemoteImageManagerDownloadOptions)options
                         outImage:(PINImage **)outImage
                        outAltRep:(id *)outAlternateRepresentation)
       
    //takes the object from the cache and returns an image or animated image.
    //if it's a non-alternative representation and skipDecode is not set it also decompresses the image.
    - (BOOL)materializeAndCacheObject:(id)object
                          cacheInDisk:(NSData *)diskData
                       additionalCost:(NSUInteger)additionalCost
                                  url:(NSURL *)url
                                  key:(NSString *)key
                              options:(PINRemoteImageManagerDownloadOptions)options
                             outImage:(PINImage **)outImage
                            outAltRep:(id *)outAlternateRepresentation
    {
        NSAssert(object != nil, @"Object should not be nil.");
        if (object == nil) {
            return NO;
        }
        BOOL alternateRepresentationsAllowed = (PINRemoteImageManagerDisallowAlternateRepresentations & options) == 0;
        BOOL skipDecode = (options & PINRemoteImageManagerDownloadOptionsSkipDecode) != 0;
        __block id alternateRepresentation = nil;
        __block PINImage *image = nil;
        __block NSData *data = nil;
        __block BOOL updateMemoryCache = NO;
        
        PINRemoteImageMemoryContainer *container = nil;
        if ([object isKindOfClass:[PINRemoteImageMemoryContainer class]]) {
            container = (PINRemoteImageMemoryContainer *)object;
            [container.lock lockWithBlock:^{
                data = container.data;
            }];
        } else {
        // 缓存图片原始数据
            updateMemoryCache = YES;
            
            // don't need to lock the container here because we just init it.
            container = [[PINRemoteImageMemoryContainer alloc] init];
            
            if ([object isKindOfClass:[PINImage class]]) {
                data = diskData;
                container.image = (PINImage *)object;
            } else if ([object isKindOfClass:[NSData class]]) {
                data = (NSData *)object;
            } else {
                //invalid item in cache
                updateMemoryCache = NO;
                data = nil;
                container = nil;
            }
            
            container.data = data;
        }
        
        if (alternateRepresentationsAllowed) {
            alternateRepresentation = [_alternateRepProvider alternateRepresentationWithData:data options:options];
        }
        
        if (alternateRepresentation == nil) {
            //we need the image
            [container.lock lockWithBlock:^{
                image = container.image;
            }];
            if (image == nil && container.data) {
                image = [PINImage pin_decodedImageWithData:container.data skipDecodeIfPossible:skipDecode];
                
                if (url != nil) {
                    image = [PINImage pin_scaledImageForImage:image withKey:key];
                }
                
                if (skipDecode == NO) {
                    [container.lock lockWithBlock:^{
                
                // 需要缓存图片解码后的数据        updateMemoryCache = YES;
                        container.image = image;
                    }];
                }
            }
        }
        
        if (updateMemoryCache) {
            [container.lock lockWithBlock:^{
                NSUInteger cacheCost = additionalCost;
                cacheCost += [container.data length];
                CGImageRef imageRef = container.image.CGImage;
                NSAssert(container.image == nil || imageRef != NULL, @"We only cache a decompressed image if we decompressed it ourselves. In that case, it should be backed by a CGImageRef.");
                if (imageRef) {
                    cacheCost += CGImageGetHeight(imageRef) * CGImageGetBytesPerRow(imageRef);
                }
                [self.cache setObjectInMemory:container forKey:key withCost:cacheCost];
            }];
        }
        
        if (diskData) {
        // 缓存原始图片数据到磁盘
            [self.cache setObjectOnDisk:diskData forKey:key];
        }
        
        if (outImage) {
            *outImage = image;
        }
        
        if (outAlternateRepresentation) {
            *outAlternateRepresentation = alternateRepresentation;
        }
        
        if (image == nil && alternateRepresentation == nil) {
            PINLog(@"Invalid item in cache");
            [self.cache removeObjectForKey:key completion:nil];
            return NO;
        }
        return YES;
    }
    ```


#### 网络请求：
根据下面的比较，可以看出网络请求方面，PINRemoteImage = SDWebImage > YYWebImage 

* YYWebImage: 使用还是老的网络请求对象 `NSURLConnection`

* SDWebImage: 使用的是新的网络请求对象 `NSURLSession`
    
* PINRemoteImage: 使用的是新的网络请求对象 `NSURLSession`


#### 性能比较
关于性能比较，只是简单的使用 FPS 工具，在列表页面，分别使用三种图片库去看效果，滑动效果比较结果如下：
YYWebImage > SDWebImage ~= PINRemoteImage

使用 YYWebImage 加载图片时的滑动效果是最好的。SDWebImage 和 PINRemoteImage 加载图片时的滑动效果是差不多的。

#### 遇到的问题
由于想要尝试使用 ASDisplayKit 去做局部列表页面的优化，但是这个库使用的图片库是 PINRemoteImage 库，而项目中使用的是 YYWebImage 库去加载图片的，这样的话就会有两套图片库，所以为了统一就需要让ASNetworkImageNode 
也要使用 YYWebImage 库去加载图片。这也是我为什么要做这个比较的原因。

代码如下：

```
#import <AsyncDisplayKit/AsyncDisplayKit.h>
#import <YYWebImage/YYWebImage.h>

@interface YYWebImageManager(ASNetworkImageNode)<ASImageCacheProtocol, ASImageDownloaderProtocol>

@end

@implementation YYWebImageManager(ASNetworkImageNode)

- (nullable id)downloadImageWithURL:(NSURL *)URL
                      callbackQueue:(dispatch_queue_t)callbackQueue
                   downloadProgress:(nullable ASImageDownloaderProgress)downloadProgress
                         completion:(ASImageDownloaderCompletion)completion {
                         // 这里传入的  YYWebImageOptionIgnoreImageDecoding，是期望图片不需要解码，因为 ASNetworkImageNode 内部自己会在合适的时机进行解码的。但是正如我上面说的，YYWebImage 不能很好的控制解码，所以这个参数是不起作用的。    
                         __weak typeof(YYWebImageOperation) *operation = nil;
    operation = [self requestImageWithURL:URL options:YYWebImageOptionIgnoreImageDecoding progress:^(NSInteger receivedSize, NSInteger expectedSize) {
        dispatch_async(callbackQueue, ^{
            CGFloat progress = expectedSize == 0 ? 0.0 : (CGFloat)receivedSize / expectedSize;

            if (downloadProgress) {
                downloadProgress(progress);
            }
        });
    } transform:nil completion:^(UIImage * _Nullable image, NSURL * _Nonnull url, YYWebImageFromType from, YYWebImageStage stage, NSError * _Nullable error) {
        dispatch_async(callbackQueue, ^{
            if (completion) {
                completion(image, error, operation);
            }
        });
    }];

    return operation;
}

- (void)cancelImageDownloadForIdentifier:(id)downloadIdentifier {
    if ([downloadIdentifier isKindOfClass:[YYWebImageOperation class]]) {
        [(YYWebImageOperation *)downloadIdentifier cancel];
    }
}

- (void)cancelImageDownloadWithResumePossibilityForIdentifier:(id)downloadIdentifier
{
    if ([downloadIdentifier isKindOfClass:[YYWebImageOperation class]]) {
        [(YYWebImageOperation *)downloadIdentifier cancel];
    }
}

- (void)clearFetchedImageFromCacheWithURL:(NSURL *)URL
{
    YYImageCache *cache = self.cache;
    dispatch_async(dispatch_get_global_queue(DISPATCH_QUEUE_PRIORITY_DEFAULT, 0), ^{
        NSString *key = [self cacheKeyForURL:URL transformKey:nil];
        [cache.memoryCache removeObjectForKey:key];
    });
}

- (void)cachedImageWithURL:(NSURL *)URL
             callbackQueue:(dispatch_queue_t)callbackQueue
                completion:(ASImageCacherCompletion)completion {
    [self.cache getImageForKey:[self cacheKeyForURL:URL transformKey:nil] withType:YYImageCacheTypeAll withBlock:^(UIImage * _Nullable image, YYImageCacheType type) {
        dispatch_async(callbackQueue, ^{
            if (completion) {
                completion(image);
            }
        });
    }];
}

@end

@interface ASNetworkImageNode(YYWebImageManager)

+ (ASNetworkImageNode *)lk_imageNode;

@end

@implementation ASNetworkImageNode(YYWebImageManager)

+ (ASNetworkImageNode *)lk_imageNode {
    return [[ASNetworkImageNode alloc] initWithCache:[YYWebImageManager sharedManager] downloader:[YYWebImageManager sharedManager]];
}

@end

// 使用方式如下：
ASNetworkImageNode *node = [ASNetworkImageNode lk_imageNode];
node.URL = [NSURL URLWithString:@""];
```

可以看到我在下载的回调方法里传入了  YYWebImageOptionIgnoreImageDecoding，是期望图片不需要解码，因为 ASNetworkImageNode 内部自己会在合适的时机进行解码的。但是正如我上面说的，YYWebImage 不能很好的控制解码，所以这个参数是不起作用的。

由于这个不能控制解码的问题，会导致在快速滑动的过程中，几十张图片同时解码，内存会飙升，YYMemoryCache 的内存警告也来不及回收，最终很容易引起 OOM 而让 app 崩溃。

所以针对这个问题，目前还是保留使用两套图片库。

### 总结
这里只是可用度方面进行了比较，并没有做全面的评估，所以实际项目还是需要根据自己的需求来决定。

### 参考
* [本地编译WebP库并添加其他到工程中
](https://github.com/ibireme/YYImage/blob/master/YYImage.podspec)

* [本地编译WebP库并添加其他到工程中1](http://www.cnblogs.com/damiao/p/4562186.html)

* [UIWebView 使用 WebP
](http://blog.devzeng.com/blog/ios-webp-usage.html
)



