---
title: SDWebImage源码解析
date: 2017-09-02 11:49:30
tags: [源码解析,iOS,SDWebImage,图片加载]
categories: iOS
---

本篇分析SDWebImage的源码，这是一个iOS开发中非常常见的图片加载库，它完成了如下的功能：

1. 在UIKit层对主要的控件(UIImageView, UIButton)进行了扩展。
2. 可以对图片进行缓存，包括在内存中缓存和在磁盘中进行缓存。
3. 后台对图片进行解码和解压缩。
4. 对动图(GIF)进行处理。

下文将分析一些关键的模块和功能，梳理一下自己对于这个框架的理解。

<!--more-->

# 整体框架
SDWebImage可以大致分成两个部分，一个是对UIKit的扩展，是用Category来实现的，这可以方便上层用户的使用；另一部分是它主要的工具类，主要包括：SDImageCache, SDWebImageDownloader, SDWebImageCoder等等，其中前两个被SDWebImageManager来管理，最后一个交给SDWebImageCodersManager来管理。下面对它们一一进行分析。

# UIKit的扩展
这里主要看UIImageView的实现，在.h中定义的一系列sd_setImageWithURL方法最终调用的都是下面的这个方法：

    - (void)sd_internalSetImageWithURL:(nullable NSURL *)url
                      placeholderImage:(nullable UIImage *)placeholder
                               options:(SDWebImageOptions)options
                          operationKey:(nullable NSString *)operationKey
                         setImageBlock:(nullable SDSetImageBlock)setImageBlock
                              progress:(nullable SDWebImageDownloaderProgressBlock)progressBlock
                             completed:(nullable SDExternalCompletionBlock)completedBlock
                               context:(nullable NSDictionary<NSString *, id> *)context {
        // 如果传入的operationKey是nil的话，就用默认的operationKey
        NSString *validOperationKey = operationKey ?: NSStringFromClass([self class]);
        // 根据operationKey取消原来的operation
        [self sd_cancelImageLoadOperationWithKey:validOperationKey];
        objc_setAssociatedObject(self, &imageURLKey, url, OBJC_ASSOCIATION_RETAIN_NONATOMIC);
        
        if (!(options & SDWebImageDelayPlaceholder)) {
            if ([context valueForKey:SDWebImageInternalSetImageGroupKey]) {
                dispatch_group_t group = [context valueForKey:SDWebImageInternalSetImageGroupKey];
                dispatch_group_enter(group);
            }
            // 如果没有延迟PlaceholderImage的展示的话，就先展示PlaceholderImage
            dispatch_main_async_safe(^{
                [self sd_setImage:placeholder imageData:nil basedOnClassOrViaCustomSetImageBlock:setImageBlock];
            });
        }
        
        if (url) {
            // check if activityView is enabled or not
            if ([self sd_showActivityIndicatorView]) {
                [self sd_addActivityIndicator];
            }
            
            // reset the progress
            self.sd_imageProgress.totalUnitCount = 0;
            self.sd_imageProgress.completedUnitCount = 0;
            
            SDWebImageManager *manager;
            // 在这里我们可以传入自己的Manager
            if ([context valueForKey:SDWebImageExternalCustomManagerKey]) {
                manager = (SDWebImageManager *)[context valueForKey:SDWebImageExternalCustomManagerKey];
            } else {
                manager = [SDWebImageManager sharedManager];
            }
            
            __weak __typeof(self)wself = self;
            SDWebImageDownloaderProgressBlock combinedProgressBlock = ^(NSInteger receivedSize, NSInteger expectedSize, NSURL * _Nullable targetURL) {
                wself.sd_imageProgress.totalUnitCount = expectedSize;
                wself.sd_imageProgress.completedUnitCount = receivedSize;
                if (progressBlock) {
                    progressBlock(receivedSize, expectedSize, targetURL);
                }
            };
            // 这里从SDWebImageManager里面得到一个CombinedOperation
            id <SDWebImageOperation> operation = [manager loadImageWithURL:url options:options progress:combinedProgressBlock completed:^(UIImage *image, NSData *data, NSError *error, SDImageCacheType cacheType, BOOL finished, NSURL *imageURL) {
                __strong __typeof (wself) sself = wself;
                if (!sself) { return; }
                [sself sd_removeActivityIndicator];
                // if the progress not been updated, mark it to complete state
                if (finished && !error && sself.sd_imageProgress.totalUnitCount == 0 && sself.sd_imageProgress.completedUnitCount == 0) {
                    sself.sd_imageProgress.totalUnitCount = SDWebImageProgressUnitCountUnknown;
                    sself.sd_imageProgress.completedUnitCount = SDWebImageProgressUnitCountUnknown;
                }
                BOOL shouldCallCompletedBlock = finished || (options & SDWebImageAvoidAutoSetImage);
                BOOL shouldNotSetImage = ((image && (options & SDWebImageAvoidAutoSetImage)) ||
                                          (!image && !(options & SDWebImageDelayPlaceholder)));
                SDWebImageNoParamsBlock callCompletedBlockClojure = ^{
                    // 注意这个sself，提交到mainQueue的时候已经对strongSelf进行了copy，因为这个最终是给gcd的，所以没有循环引用的问题。
                    if (!sself) { return; }
                    if (!shouldNotSetImage) {
                        // 需要设置图片，外层已经setImage
                        [sself sd_setNeedsLayout];
                    }
                    if (completedBlock && shouldCallCompletedBlock) {
                        completedBlock(image, error, cacheType, url);
                    }
                };
                
                // case 1a: we got an image, but the SDWebImageAvoidAutoSetImage flag is set
                // OR
                // case 1b: we got no image and the SDWebImageDelayPlaceholder is not set
                if (shouldNotSetImage) {
                    dispatch_main_async_safe(callCompletedBlockClojure);
                    return;
                }
                
                UIImage *targetImage = nil;
                NSData *targetData = nil;
                if (image) {
                    // case 2a: we got an image and the SDWebImageAvoidAutoSetImage is not set
                    targetImage = image;
                    targetData = data;
                } else if (options & SDWebImageDelayPlaceholder) {
                    // case 2b: we got no image and the SDWebImageDelayPlaceholder flag is set
                    targetImage = placeholder;
                    targetData = nil;
                }
                
                // check whether we should use the image transition
                SDWebImageTransition *transition = nil;
                if (finished && (options & SDWebImageForceTransition || cacheType == SDImageCacheTypeNone)) {
                    transition = sself.sd_imageTransition;
                }
                if ([context valueForKey:SDWebImageInternalSetImageGroupKey]) {
                    dispatch_group_t group = [context valueForKey:SDWebImageInternalSetImageGroupKey];
                    dispatch_group_enter(group);
                    dispatch_main_async_safe(^{
                        [sself sd_setImage:targetImage imageData:targetData basedOnClassOrViaCustomSetImageBlock:setImageBlock transition:transition cacheType:cacheType imageURL:imageURL];
                    });
                    // ensure completion block is called after custom setImage process finish
                    dispatch_group_notify(group, dispatch_get_main_queue(), ^{
                        callCompletedBlockClojure();
                    });
                } else {
                    dispatch_main_async_safe(^{
                        // 最终在主线程调用这个方法
                        [sself sd_setImage:targetImage imageData:targetData basedOnClassOrViaCustomSetImageBlock:setImageBlock transition:transition cacheType:cacheType imageURL:imageURL];
                        callCompletedBlockClojure();
                    });
                }
            }];
            [self sd_setImageLoadOperation:operation forKey:validOperationKey];
        } else {
            dispatch_main_async_safe(^{
                [self sd_removeActivityIndicator];
                if (completedBlock) {
                    NSError *error = [NSError errorWithDomain:SDWebImageErrorDomain code:-1 userInfo:@{NSLocalizedDescriptionKey : @"Trying to load a nil url"}];
                    completedBlock(nil, error, SDImageCacheTypeNone, url);
                }
            });
        }
    }
上述代码除去对context中信息的判断（我自己也没用过）,可以看到在给SDWebImageManager传入的completeBlock里面如果判断的shouldNotSetImage为NO的话，最终是在主线程执行如下方法的：

    - (void)sd_setImage:(UIImage *)image imageData:(NSData *)imageData basedOnClassOrViaCustomSetImageBlock:(SDSetImageBlock)setImageBlock transition:(SDWebImageTransition *)transition cacheType:(SDImageCacheType)cacheType imageURL:(NSURL *)imageURL {
        UIView *view = self;
        SDSetImageBlock finalSetImageBlock;
        if (setImageBlock) {
            finalSetImageBlock = setImageBlock;
        } else if ([view isKindOfClass:[UIImageView class]]) {
            UIImageView *imageView = (UIImageView *)view;
            finalSetImageBlock = ^(UIImage *setImage, NSData *setImageData) {
                imageView.image = setImage;
            };
        } else if ([view isKindOfClass:[UIButton class]]) {
            UIButton *button = (UIButton *)view;
            finalSetImageBlock = ^(UIImage *setImage, NSData *setImageData){
                [button setImage:setImage forState:UIControlStateNormal];
            };
        }
        
        if (transition) {
            [UIView transitionWithView:view duration:0 options:0 animations:^{
                // 0 duration to let UIKit render placeholder and prepares block
                if (transition.prepares) {
                    transition.prepares(view, image, imageData, cacheType, imageURL);
                }
            } completion:^(BOOL finished) {
                [UIView transitionWithView:view duration:transition.duration options:transition.animationOptions animations:^{
                    if (finalSetImageBlock && !transition.avoidAutoSetImage) {
                        finalSetImageBlock(image, imageData);
                    }
                    if (transition.animations) {
                        transition.animations(view, image);
                    }
                } completion:transition.completion];
            }];
        } else {
            if (finalSetImageBlock) {
                finalSetImageBlock(image, imageData);
            }
        }
    }
在这里首先要判断我们是否设置了transition这个关联对象，这个东西是用来保存我们想要在更新图片时所要的过渡动画的信息，例如duration, completeBlock, animationOptions等等，如果没有设置的话, 则去执行finalSetImageBlock。

值得注意的是operationKey, 一个operationKey对应的是一个UIView或其子类实例的一个operation，这个是由一个附加在UIView上的字典类型的关联对象所保存的。在执行新的操作时，要把原来的operation给cancel掉。SDWebImageOperation协议所包含的方法只有一个：

    - (void)cancel;

这个方法用来取消从Manager中得到的operation，包括从磁盘中获取或者从网络下载。

# 工具类
工具类主要包括SDWebImageManager，SDImageCache，SDWebImageDownloader这三个类，后两者归SDWebImageManager进行管理，将自己所生成的Operation提交给manager合成一个combinedOperation，然后Manager再将这个combinedOperation交给UIKit层。下面对这些类逐一进行分析。

## SDWebImageManager
SDWebImageManager是工具类的入口，主要包括如下属性：

    // Cache类
    @property (strong, nonatomic, readwrite, nonnull) SDImageCache *imageCache;
    // Downloader
    @property (strong, nonatomic, readwrite, nonnull) SDWebImageDownloader *imageDownloader;
    // 下载失败的URL
    @property (strong, nonatomic, nonnull) NSMutableSet<NSURL *> *failedURLs;
    // 正在运行的Operation
    @property (strong, nonatomic, nonnull) NSMutableArray<SDWebImageCombinedOperation *> *runningOperations;

下面直接看其关键函数：

    - (id <SDWebImageOperation>)loadImageWithURL:(nullable NSURL *)url
                                         options:(SDWebImageOptions)options
                                        progress:(nullable SDWebImageDownloaderProgressBlock)progressBlock
                                       completed:(nullable SDInternalCompletionBlock)completedBlock {
        // Invoking this method without a completedBlock is pointless
        NSAssert(completedBlock != nil, @"If you mean to prefetch the image, use -[SDWebImagePrefetcher prefetchURLs] instead");

        // Very common mistake is to send the URL using NSString object instead of NSURL. For some strange reason, Xcode won't
        // throw any warning for this type mismatch. Here we failsafe this error by allowing URLs to be passed as NSString.
        if ([url isKindOfClass:NSString.class]) {
            url = [NSURL URLWithString:(NSString *)url];
        }

        // Prevents app crashing on argument type error like sending NSNull instead of NSURL
        if (![url isKindOfClass:NSURL.class]) {
            url = nil;
        }

        SDWebImageCombinedOperation *operation = [SDWebImageCombinedOperation new];
        operation.manager = self;

        BOOL isFailedUrl = NO;
        if (url) {
            @synchronized (self.failedURLs) {
                isFailedUrl = [self.failedURLs containsObject:url];
            }
        }

        // 如果这个URL包含在失败的URL里面并且没有开启SDWebImageRetryFailed选项的话
        if (url.absoluteString.length == 0 || (!(options & SDWebImageRetryFailed) && isFailedUrl)) {
            [self callCompletionBlockForOperation:operation completion:completedBlock error:[NSError errorWithDomain:NSURLErrorDomain code:NSURLErrorFileDoesNotExist userInfo:nil] url:url];
            return operation;
        }

        @synchronized (self.runningOperations) {
            [self.runningOperations addObject:operation];
        }
        NSString *key = [self cacheKeyForURL:url];
        
        SDImageCacheOptions cacheOptions = 0;
        if (options & SDWebImageQueryDataWhenInMemory) cacheOptions |= SDImageCacheQueryDataWhenInMemory;
        if (options & SDWebImageQueryDiskSync) cacheOptions |= SDImageCacheQueryDiskSync;
        
        __weak SDWebImageCombinedOperation *weakOperation = operation;
        operation.cacheOperation = [self.imageCache queryCacheOperationForKey:key options:cacheOptions done:^(UIImage *cachedImage, NSData *cachedData, SDImageCacheType cacheType) {
            __strong __typeof(weakOperation) strongOperation = weakOperation;
            if (!strongOperation || strongOperation.isCancelled) {
                [self safelyRemoveOperationFromRunning:strongOperation];
                return;
            }
            
            // Check whether we should download image from network
            BOOL shouldDownload = (!(options & SDWebImageFromCacheOnly))
                && (!cachedImage || options & SDWebImageRefreshCached)
                && (![self.delegate respondsToSelector:@selector(imageManager:shouldDownloadImageForURL:)] || [self.delegate imageManager:self shouldDownloadImageForURL:url]);
            if (shouldDownload) {
                if (cachedImage && options & SDWebImageRefreshCached) {
                    // If image was found in the cache but SDWebImageRefreshCached is provided, notify about the cached image
                    // AND try to re-download it in order to let a chance to NSURLCache to refresh it from server.
                    [self callCompletionBlockForOperation:strongOperation completion:completedBlock image:cachedImage data:cachedData error:nil cacheType:cacheType finished:YES url:url];
                }

                // download if no image or requested to refresh anyway, and download allowed by delegate
                SDWebImageDownloaderOptions downloaderOptions = 0;
                if (options & SDWebImageLowPriority) downloaderOptions |= SDWebImageDownloaderLowPriority;
                if (options & SDWebImageProgressiveDownload) downloaderOptions |= SDWebImageDownloaderProgressiveDownload;
                if (options & SDWebImageRefreshCached) downloaderOptions |= SDWebImageDownloaderUseNSURLCache;
                if (options & SDWebImageContinueInBackground) downloaderOptions |= SDWebImageDownloaderContinueInBackground;
                if (options & SDWebImageHandleCookies) downloaderOptions |= SDWebImageDownloaderHandleCookies;
                if (options & SDWebImageAllowInvalidSSLCertificates) downloaderOptions |= SDWebImageDownloaderAllowInvalidSSLCertificates;
                if (options & SDWebImageHighPriority) downloaderOptions |= SDWebImageDownloaderHighPriority;
                if (options & SDWebImageScaleDownLargeImages) downloaderOptions |= SDWebImageDownloaderScaleDownLargeImages;
                
                if (cachedImage && options & SDWebImageRefreshCached) {
                    // force progressive off if image already cached but forced refreshing
                    downloaderOptions &= ~SDWebImageDownloaderProgressiveDownload;
                    // ignore image read from NSURLCache if image if cached but force refreshing
                    downloaderOptions |= SDWebImageDownloaderIgnoreCachedResponse;
                }
                
                // `SDWebImageCombinedOperation` -> `SDWebImageDownloadToken` -> `downloadOperationCancelToken`, which is a `SDCallbacksDictionary` and retain the completed block below, so we need weak-strong again to avoid retain cycle
                __weak typeof(strongOperation) weakSubOperation = strongOperation;
                strongOperation.downloadToken = [self.imageDownloader downloadImageWithURL:url options:downloaderOptions progress:progressBlock completed:^(UIImage *downloadedImage, NSData *downloadedData, NSError *error, BOOL finished) {
                    // 注意这里又进行了一次weak变strong，因为这个时候要处理strongOperation的downloadOperation
                    __strong typeof(weakSubOperation) strongSubOperation = weakSubOperation;
                    if (!strongSubOperation || strongSubOperation.isCancelled) {
                        // Do nothing if the operation was cancelled
                        // See #699 for more details
                        // if we would call the completedBlock, there could be a race condition between this block and another completedBlock for the same object, so if this one is called second, we will overwrite the new data
                    } else if (error) {
                        [self callCompletionBlockForOperation:strongSubOperation completion:completedBlock error:error url:url];
                        BOOL shouldBlockFailedURL;
                        // Check whether we should block failed url
                        if ([self.delegate respondsToSelector:@selector(imageManager:shouldBlockFailedURL:withError:)]) {
                            shouldBlockFailedURL = [self.delegate imageManager:self shouldBlockFailedURL:url withError:error];
                        } else {
                            // 判断加入failedURLs的条件
                            shouldBlockFailedURL = (   error.code != NSURLErrorNotConnectedToInternet
                                                    && error.code != NSURLErrorCancelled
                                                    && error.code != NSURLErrorTimedOut
                                                    && error.code != NSURLErrorInternationalRoamingOff
                                                    && error.code != NSURLErrorDataNotAllowed
                                                    && error.code != NSURLErrorCannotFindHost
                                                    && error.code != NSURLErrorCannotConnectToHost
                                                    && error.code != NSURLErrorNetworkConnectionLost);
                        }
                        
                        if (shouldBlockFailedURL) {
                            @synchronized (self.failedURLs) {
                                [self.failedURLs addObject:url];
                            }
                        }
                    }
                    else {
                        if ((options & SDWebImageRetryFailed)) {
                            @synchronized (self.failedURLs) {
                                [self.failedURLs removeObject:url];
                            }
                        }
                        
                        BOOL cacheOnDisk = !(options & SDWebImageCacheMemoryOnly);
                        
                        // We've done the scale process in SDWebImageDownloader with the shared manager, this is used for custom manager and avoid extra scale.
                        if (self != [SDWebImageManager sharedManager] && self.cacheKeyFilter && downloadedImage) {
                            downloadedImage = [self scaledImageForKey:key image:downloadedImage];
                        }

                        if (options & SDWebImageRefreshCached && cachedImage && !downloadedImage) {
                            // Image refresh hit the NSURLCache cache, do not call the completion block
                        } else if (downloadedImage && (!downloadedImage.images || (options & SDWebImageTransformAnimatedImage)) && [self.delegate respondsToSelector:@selector(imageManager:transformDownloadedImage:withURL:)]) {
                            dispatch_async(dispatch_get_global_queue(DISPATCH_QUEUE_PRIORITY_HIGH, 0), ^{
                                // 在globalQueue里面执行transform
                                UIImage *transformedImage = [self.delegate imageManager:self transformDownloadedImage:downloadedImage withURL:url];

                                if (transformedImage && finished) {
                                    BOOL imageWasTransformed = ![transformedImage isEqual:downloadedImage];
                                    NSData *cacheData;
                                    // pass nil if the image was transformed, so we can recalculate the data from the image
                                    if (self.cacheSerializer) {
                                        cacheData = self.cacheSerializer(transformedImage, (imageWasTransformed ? nil : downloadedData), url);
                                    } else {
                                        cacheData = (imageWasTransformed ? nil : downloadedData);
                                    }
                                    // 最后执行store方法
                                    [self.imageCache storeImage:transformedImage imageData:cacheData forKey:key toDisk:cacheOnDisk completion:nil];
                                }
                                
                                [self callCompletionBlockForOperation:strongSubOperation completion:completedBlock image:transformedImage data:downloadedData error:nil cacheType:SDImageCacheTypeNone finished:finished url:url];
                            });
                        } else {
                            if (downloadedImage && finished) {
                                if (self.cacheSerializer) {
                                    dispatch_async(dispatch_get_global_queue(DISPATCH_QUEUE_PRIORITY_HIGH, 0), ^{
                                        NSData *cacheData = self.cacheSerializer(downloadedImage, downloadedData, url);
                                        [self.imageCache storeImage:downloadedImage imageData:cacheData forKey:key toDisk:cacheOnDisk completion:nil];
                                    });
                                } else {
                                    [self.imageCache storeImage:downloadedImage imageData:downloadedData forKey:key toDisk:cacheOnDisk completion:nil];
                                }
                            }
                            [self callCompletionBlockForOperation:strongSubOperation completion:completedBlock image:downloadedImage data:downloadedData error:nil cacheType:SDImageCacheTypeNone finished:finished url:url];
                        }
                    }

                    if (finished) {
                        [self safelyRemoveOperationFromRunning:strongSubOperation];
                    }
                }];
            } else if (cachedImage) {
                [self callCompletionBlockForOperation:strongOperation completion:completedBlock image:cachedImage data:cachedData error:nil cacheType:cacheType finished:YES url:url];
                [self safelyRemoveOperationFromRunning:strongOperation];
            } else {
                // Image not in cache and download disallowed by delegate
                [self callCompletionBlockForOperation:strongOperation completion:completedBlock image:nil data:nil error:nil cacheType:SDImageCacheTypeNone finished:YES url:url];
                [self safelyRemoveOperationFromRunning:strongOperation];
            }
        }];

        return operation;
    }

这个函数看起来是相当冗长的，其大致逻辑如下：

1. 生成一个SDWebImageCombinedOperation实例，对这个实例的cacheOperation进行赋值。
2. 在cacheOperation所传入的Block里面，先对是否需要执行网络下载进行判断。
3. 如果不需要下载可能有两种情况：一种是找到了cachedImage；一种是外界的managerDelegate显式地禁用了download。
4. 如果需要下载，对combinedOperation的downloadToken进行赋值，这个实际上是一个dictionary，稍后再说。
5. 在下载完成的block里面，先对error出现的情况进行判断，并根据blockFailedURL的规则来判断是否应该加入到failedURLs当中。
6. 如果没有出现error, 判断managerDelegate是否定义了transformImage方法。如果定义了，在globalQueue里面执行transform，并执行步骤7；如果没有定义，则直接执行步骤7。
7. 如果在外部定义了cacheSerializer，则用该block得到一个修改过的NSData，并传给imageCache进行保存；如果没有，直接传入下载的NSData进行保存。

这个地方值得注意的是，传入imageCache所用的key，调用了下面的方法：

    - (nullable NSString *)cacheKeyForURL:(nullable NSURL *)url {
        if (!url) {
            return @"";
        }

        if (self.cacheKeyFilter) {
            return self.cacheKeyFilter(url);
        } else {
            return url.absoluteString;
        }
    }

这个方法有一个cacheKeyFilter, 用来自定义传给imageCache所用的key值，imageCahce会对它进行MD5摘要。这个一般情况下用原本的URL就可以，比较特殊的情况是遇到CDN的情况，因为在CDN的情况下有可能对于同一张图片，不同的地区返回的URL是不同的，这个时候就需要我们对于这个URL进行判定，选取其中的有用部分来进行之后的MD5摘要，比如可以对同一张图片采用相同的id。

还有一个注意的地方是SDWebImageCombinedOperation这个类，这个类包含了读取本地文件的NSOperation和在网络下载的token，还有一个标记状态的cancelled属性，这样执行cancel方法，就可以将两项操作同时取消。

## SDImageCache
这个包含了对图片的缓存操作，主要包括在内存中的缓存和在磁盘上的缓存；其中在内存中的缓存委托给了iOS中的NSCache，而在硬盘中的缓存则是交给了一个serial类型的io队列。它主要的成员如下：

    @property (strong, nonatomic, nonnull) SDMemoryCache *memCache;
    @property (assign, nonatomic) NSUInteger maxMemoryCost;
    @property (assign, nonatomic) NSUInteger maxMemoryCountLimit;
    @property (strong, nonatomic, nullable) dispatch_queue_t ioQueue;

其中第一个SDMemoryCache是NSCache的子类，其内部监听了didReceiveMemoryWarning事件，当收到内存警告时，会清空所有的缓存；第二个和第三个是用来设置NSCache的，第二个的意思是所占用的最大内存，第三个的意思是所缓存对象的最大数目；最后一个是读取本地磁盘文件对应的queue。

下面来看关键函数：

    - (nullable NSOperation *)queryCacheOperationForKey:(nullable NSString *)key options:(SDImageCacheOptions)options done:(nullable SDCacheQueryCompletedBlock)doneBlock {
        if (!key) {
            if (doneBlock) {
                doneBlock(nil, nil, SDImageCacheTypeNone);
            }
            return nil;
        }
        
        // First check the in-memory cache...
        UIImage *image = [self imageFromMemoryCacheForKey:key];
        BOOL shouldQueryMemoryOnly = (image && !(options & SDImageCacheQueryDataWhenInMemory));
        if (shouldQueryMemoryOnly) {
            if (doneBlock) {
                doneBlock(image, nil, SDImageCacheTypeMemory);
            }
            return nil;
        }
        
        NSOperation *operation = [NSOperation new];
        void(^queryDiskBlock)(void) =  ^{
            if (operation.isCancelled) {
                // do not call the completion if cancelled
                return;
            }
            
            @autoreleasepool {
                NSData *diskData = [self diskImageDataBySearchingAllPathsForKey:key];
                UIImage *diskImage;
                SDImageCacheType cacheType = SDImageCacheTypeDisk;
                if (image) {
                    // the image is from in-memory cache
                    diskImage = image;
                    cacheType = SDImageCacheTypeMemory;
                } else if (diskData) {
                    // decode image data only if in-memory cache missed
                    diskImage = [self diskImageForKey:key data:diskData];
                    if (diskImage && self.config.shouldCacheImagesInMemory) {
                        NSUInteger cost = SDCacheCostForImage(diskImage);
                        [self.memCache setObject:diskImage forKey:key cost:cost];
                    }
                }
                
                if (doneBlock) {
                    if (options & SDImageCacheQueryDiskSync) {
                        doneBlock(diskImage, diskData, cacheType);
                    } else {
                        dispatch_async(dispatch_get_main_queue(), ^{
                            doneBlock(diskImage, diskData, cacheType);
                        });
                    }
                }
            }
        };
        
        if (options & SDImageCacheQueryDiskSync) {
            queryDiskBlock();
        } else {
            dispatch_async(self.ioQueue, queryDiskBlock);
        }
        
        return operation;
    }

这段代码的逻辑比较简单，首先根据key读取memory当中的缓存，如果有并且没有设置SDImageCacheQueryDataWhenInMemory（即便在内存中，也要访问磁盘），就直接调用doneBlock，否则生成一个NSOperation，这个operation的作用仅仅是为了标记，并不是和NSOperationQueue有直接的关联；接下来定义一个block，先在所有的cachePaths下找到对应的NSData, 找到的话生成UIImage并将其缓存，最后根据是否设置同步读取来判断是直接调用doneBlock还是异步调用。

下面来看一下关于“存”的实现，其核心函数为：

    - (void)storeImage:(nullable UIImage *)image
             imageData:(nullable NSData *)imageData
                forKey:(nullable NSString *)key
                toDisk:(BOOL)toDisk
            completion:(nullable SDWebImageNoParamsBlock)completionBlock {
        if (!image || !key) {
            if (completionBlock) {
                completionBlock();
            }
            return;
        }
        // if memory cache is enabled
        if (self.config.shouldCacheImagesInMemory) {
            NSUInteger cost = SDCacheCostForImage(image);
            [self.memCache setObject:image forKey:key cost:cost];
        }
        
        if (toDisk) {
            dispatch_async(self.ioQueue, ^{
                @autoreleasepool {
                    NSData *data = imageData;
                    if (!data && image) {
                        // If we do not have any data to detect image format, check whether it contains alpha channel to use PNG or JPEG format
                        SDImageFormat format;
                        if (SDCGImageRefContainsAlpha(image.CGImage)) {
                            format = SDImageFormatPNG;
                        } else {
                            format = SDImageFormatJPEG;
                        }
                        data = [[SDWebImageCodersManager sharedInstance] encodedDataWithImage:image format:format];
                    }
                    [self _storeImageDataToDisk:data forKey:key];
                }
                
                if (completionBlock) {
                    dispatch_async(dispatch_get_main_queue(), ^{
                        completionBlock();
                    });
                }
            });
        } else {
            if (completionBlock) {
                completionBlock();
            }
        }
    }

需要先判断一下图片是否包含透明的部分来决定图片的格式，如果包含透明部分就用PNG，没有用JPEG，最后根据key生成文件名写入，不再多说。有兴趣的可以去看一下它生成文件名的MD5摘要算法（不用就忘，没用）。

## SDWebImageDownloader

下面来看一下Downloader的实现，Downloader是用NSOperationQueue和NSOperation来执行下载的。下面来看一下它的核心方法：

    - (nullable SDWebImageDownloadToken *)downloadImageWithURL:(nullable NSURL *)url
                                                       options:(SDWebImageDownloaderOptions)options
                                                      progress:(nullable SDWebImageDownloaderProgressBlock)progressBlock
                                                     completed:(nullable SDWebImageDownloaderCompletedBlock)completedBlock {
        __weak SDWebImageDownloader *wself = self;

        return [self addProgressCallback:progressBlock completedBlock:completedBlock forURL:url createCallback:^SDWebImageDownloaderOperation *{
            __strong __typeof (wself) sself = wself;
            NSTimeInterval timeoutInterval = sself.downloadTimeout;
            if (timeoutInterval == 0.0) {
                timeoutInterval = 15.0;
            }

            // In order to prevent from potential duplicate caching (NSURLCache + SDImageCache) we disable the cache for image requests if told otherwise
            NSURLRequestCachePolicy cachePolicy = options & SDWebImageDownloaderUseNSURLCache ? NSURLRequestUseProtocolCachePolicy : NSURLRequestReloadIgnoringLocalCacheData;
            NSMutableURLRequest *request = [[NSMutableURLRequest alloc] initWithURL:url
                                                                        cachePolicy:cachePolicy
                                                                    timeoutInterval:timeoutInterval];
            
            request.HTTPShouldHandleCookies = (options & SDWebImageDownloaderHandleCookies);
            request.HTTPShouldUsePipelining = YES;
            if (sself.headersFilter) {
                request.allHTTPHeaderFields = sself.headersFilter(url, [sself allHTTPHeaderFields]);
            }
            else {
                request.allHTTPHeaderFields = [sself allHTTPHeaderFields];
            }
            SDWebImageDownloaderOperation *operation = [[sself.operationClass alloc] initWithRequest:request inSession:sself.session options:options];
            operation.shouldDecompressImages = sself.shouldDecompressImages;
            
            if (sself.urlCredential) {
                operation.credential = sself.urlCredential;
            } else if (sself.username && sself.password) {
                operation.credential = [NSURLCredential credentialWithUser:sself.username password:sself.password persistence:NSURLCredentialPersistenceForSession];
            }
            
            if (options & SDWebImageDownloaderHighPriority) {
                operation.queuePriority = NSOperationQueuePriorityHigh;
            } else if (options & SDWebImageDownloaderLowPriority) {
                operation.queuePriority = NSOperationQueuePriorityLow;
            }
            
            if (sself.executionOrder == SDWebImageDownloaderLIFOExecutionOrder) {
                // Emulate LIFO execution order by systematically adding new operations as last operation's dependency
                [sself.lastAddedOperation addDependency:operation];
                sself.lastAddedOperation = operation;
            }

            return operation;
        }];
    }

主要的一大堆代码都是对于NSURLSession的配置，包括是否用URL缓存，是否处理Cookies，以及headersFilter，然后生成一个SDWebImageDownloaderOperation返回，这是一个NSOperation的子类，注意提交的默认顺序是FIFO的，如果设置成LIFO的，则将最后一次加入的operation的dependency设置为当前加入的operation。

下面来看一下SDWebImageDownloaderOperation的实现，它是继承于NSOperation，内部采用NSURLSession进行下载，并作为所用session的delegate来监听下载的进度，主要包括progressiveDownload(如果开启此选项)和完成下载的回调：

    - (void)URLSession:(NSURLSession *)session task:(NSURLSessionTask *)task didCompleteWithError:(NSError *)error {
        @synchronized(self) {
            self.dataTask = nil;
            __weak typeof(self) weakSelf = self;
            dispatch_async(dispatch_get_main_queue(), ^{
                [[NSNotificationCenter defaultCenter] postNotificationName:SDWebImageDownloadStopNotification object:weakSelf];
                if (!error) {
                    [[NSNotificationCenter defaultCenter] postNotificationName:SDWebImageDownloadFinishNotification object:weakSelf];
                }
            });
        }
        
        // make sure to call `[self done]` to mark operation as finished
        if (error) {
            [self callCompletionBlocksWithError:error];
            [self done];
        } else {
            if ([self callbacksForKey:kCompletedCallbackKey].count > 0) {
                /**
                 -  If you specified to use `NSURLCache`, then the response you get here is what you need.
                 */
                __block NSData *imageData = [self.imageData copy];
                if (imageData) {
                    /**  if you specified to only use cached data via `SDWebImageDownloaderIgnoreCachedResponse`,
                     -  then we should check if the cached data is equal to image data
                     */
                    if (self.options & SDWebImageDownloaderIgnoreCachedResponse && [self.cachedData isEqualToData:imageData]) {
                        // call completion block with nil
                        [self callCompletionBlocksWithImage:nil imageData:nil error:nil finished:YES];
                        [self done];
                    } else {
                        // decode the image in coder queue
                        dispatch_async(self.coderQueue, ^{
                            UIImage *image = [[SDWebImageCodersManager sharedInstance] decodedImageWithData:imageData];
                            NSString *key = [[SDWebImageManager sharedManager] cacheKeyForURL:self.request.URL];
                            image = [self scaledImageForKey:key image:image];
                            
                            BOOL shouldDecode = YES;
                            // Do not force decoding animated GIFs and WebPs
                            if (image.images) {
                                shouldDecode = NO;
                            } else {
    #ifdef SD_WEBP
                                SDImageFormat imageFormat = [NSData sd_imageFormatForImageData:imageData];
                                if (imageFormat == SDImageFormatWebP) {
                                    shouldDecode = NO;
                                }
    #endif
                            }
                            
                            if (shouldDecode) {
                                if (self.shouldDecompressImages) {
                                    BOOL shouldScaleDown = self.options & SDWebImageDownloaderScaleDownLargeImages;
                                    image = [[SDWebImageCodersManager sharedInstance] decompressedImageWithImage:image data:&imageData options:@{SDWebImageCoderScaleDownLargeImagesKey: @(shouldScaleDown)}];
                                }
                            }
                            CGSize imageSize = image.size;
                            if (imageSize.width == 0 || imageSize.height == 0) {
                                [self callCompletionBlocksWithError:[NSError errorWithDomain:SDWebImageErrorDomain code:0 userInfo:@{NSLocalizedDescriptionKey : @"Downloaded image has 0 pixels"}]];
                            } else {
                                [self callCompletionBlocksWithImage:image imageData:imageData error:nil finished:YES];
                            }
                            [self done];
                        });
                    }
                } else {
                    [self callCompletionBlocksWithError:[NSError errorWithDomain:SDWebImageErrorDomain code:0 userInfo:@{NSLocalizedDescriptionKey : @"Image data is nil"}]];
                    [self done];
                }
            } else {
                [self done];
            }
        }
    }

逻辑比较简单，先判断出现error的情况，如果发生了error直接返回；再来看之后下载的数据，这里后面的shouldDecode参数不应该放到前面么？？我猜它的意思是先对图片解码，再判断图片的格式是否能够解压缩，如果能则对其解压缩，最后调用对应的completeBlock。

值得一提的是coderQueue，这是一个serial类型的queue，之所以设置为serial类型是因为保证同时只有一个corder在工作，有效避免了同时加载多个中等分辨率图片带来的内存峰值。

progressiveDownload这个特性也是和上面差不多的，区别在于progressiveDownload图片解码部分是交给特定的progressiveCorder去做的。在发送下载图片请求时先要知道这个图片的大小，这是在**- (void)URLSession:dataTask:didReceiveResponse:completionHandler:**方法中拿到的，在**- (void)URLSession:dataTask:didReceiveData:**方法中根据下载完成的数据和期待数据量的比值来调用对应的progressHandler，这里不再细谈。

现在我们看一下返回的SDWebImageDownloadToken, 它包含了下载的url, 对应的Operation，还有一个cancelToken，其中cancelToken是通过调用SDWebImageDownloaderOperation中的**- (nullable id)addHandlersForProgress:completed:**，其实现如下：

    - (nullable id)addHandlersForProgress:(nullable SDWebImageDownloaderProgressBlock)progressBlock
                                completed:(nullable SDWebImageDownloaderCompletedBlock)completedBlock {
        SDCallbacksDictionary *callbacks = [NSMutableDictionary new];
        if (progressBlock) callbacks[kProgressCallbackKey] = [progressBlock copy];
        if (completedBlock) callbacks[kCompletedCallbackKey] = [completedBlock copy];
        LOCK(self.callbacksLock);
        [self.callbackBlocks addObject:callbacks];
        UNLOCK(self.callbacksLock);
        return callbacks;
    }

返回的其实是一个NSMutableDictionary类型，其中保存了completeHandler和progressiveHandler，下面来看一下SDWebImageCombinedOperation是怎么取消的：

    - (void)cancel {
        @synchronized(self) {
            self.cancelled = YES;
            if (self.cacheOperation) {
                [self.cacheOperation cancel];
                self.cacheOperation = nil;
            }
            if (self.downloadToken) {
                [self.manager.imageDownloader cancel:self.downloadToken];
            }
            [self.manager safelyRemoveOperationFromRunning:self];
        }
    }

Downloader当中的cancel方法：

    - (void)cancel:(nullable SDWebImageDownloadToken *)token {
        NSURL *url = token.url;
        if (!url) {
            return;
        }
        LOCK(self.operationsLock);
        SDWebImageDownloaderOperation *operation = [self.URLOperations objectForKey:url];
        if (operation) {
            BOOL canceled = [operation cancel:token.downloadOperationCancelToken];
            if (canceled) {
                [self.URLOperations removeObjectForKey:url];
            }
        }
        UNLOCK(self.operationsLock);
    }

最后看看SDWebImageDownloaderOperation中的cancel方法：

    - (BOOL)cancel:(nullable id)token {
        BOOL shouldCancel = NO;
        LOCK(self.callbacksLock);
        [self.callbackBlocks removeObjectIdenticalTo:token];
        if (self.callbackBlocks.count == 0) {
            shouldCancel = YES;
        }
        UNLOCK(self.callbacksLock);
        if (shouldCancel) {
            [self cancel];
        }
        return shouldCancel;
    }

cancel方法会比较传进来的token是否是在callbackBlocks里面，callbackBlocks是一个数组，如果成功移除那么它对应的count就是0，那就执行之后的cancel操作。这里既然调用了removeObjectIdenticalTo:方法，那为啥不直接在之前保存callbacks的地址然后在这里进行判断呢，纳闷……

## SDWebImageCorder

SDWebImageCorder是用来对图片进行编码，解码，解压缩的，它本身是一个协议，实现此协议的子类为SDWebImageImageIOCorder和SDWebImageGIFCorder，这两个统一交给SDWebImageCodersManager管理。先看一下协议所定义的主要方法：

    - (BOOL)canDecodeFromData:(nullable NSData *)data;
    
    - (nullable UIImage *)decodedImageWithData:(nullable NSData *)data;

    - (nullable UIImage *)decompressedImageWithImage:(nullable UIImage *)image
                                                data:(NSData * _Nullable * _Nonnull)data
                                             options:(nullable NSDictionary<NSString*, NSObject*>*)optionsDict;
    - (BOOL)canEncodeToFormat:(SDImageFormat)format;

从函数名直接可以看出这个方法什么意思，这里不多说。它调用的时机主要是两个，一个是在SDImageCache从磁盘读取完毕之后，需要在ioQueue或者同步进行进行解码和解压缩：

    - (nullable UIImage *)diskImageForKey:(nullable NSString *)key data:(nullable NSData *)data {
        if (data) {
            UIImage *image = [[SDWebImageCodersManager sharedInstance] decodedImageWithData:data];
            image = [self scaledImageForKey:key image:image];
            if (self.config.shouldDecompressImages) {
                image = [[SDWebImageCodersManager sharedInstance] decompressedImageWithImage:image data:&data options:@{SDWebImageCoderScaleDownLargeImagesKey: @(NO)}];
            }
            return image;
        } else {
            return nil;
        }
    }

另外一个地方是在下载完毕之后进行解码和解压缩，看前面的代码。

为什么需要解压缩？图片的格式有PNG，JPEG等等，这些都不是位图格式，只是图片的存储格式。如果我们需要展示在UIImageView上面的话，还需要进行解压缩。默认的处理为：当我们把一个未解压的图片赋给UIImageView的时候，在展示之前系统会对其进行解压缩，并把解压缩后的位图缓存供下次使用，这个过程是发生在主线程并交给CPU处理的，因此在解压大图的时候会带来卡顿以及内存飙升的问题。当系统收到memoryWarning的时候，会把之前的位图缓存给清空。既然解压缩默认是在主线程中进行的，那么一个优化手段是放到后台进行解压缩，从而不干扰主线程。

下面看SDWebImageImageIOCorder的实现，先看核心函数：

    - (UIImage *)decompressedImageWithImage:(UIImage *)image
                                       data:(NSData *__autoreleasing  _Nullable *)data
                                    options:(nullable NSDictionary<NSString*, NSObject*>*)optionsDict {
    #if SD_MAC
        return image;
    #endif
    #if SD_UIKIT || SD_WATCH
        BOOL shouldScaleDown = NO;
        if (optionsDict != nil) {
            NSNumber *scaleDownLargeImagesOption = nil;
            if ([optionsDict[SDWebImageCoderScaleDownLargeImagesKey] isKindOfClass:[NSNumber class]]) {
                scaleDownLargeImagesOption = (NSNumber *)optionsDict[SDWebImageCoderScaleDownLargeImagesKey];
            }
            if (scaleDownLargeImagesOption != nil) {
                shouldScaleDown = [scaleDownLargeImagesOption boolValue];
            }
        }
        if (!shouldScaleDown) {
            return [self sd_decompressedImageWithImage:image];
        } else {
            UIImage *scaledDownImage = [self sd_decompressedAndScaledDownImageWithImage:image];
            if (scaledDownImage && !CGSizeEqualToSize(scaledDownImage.size, image.size)) {
                // if the image is scaled down, need to modify the data pointer as well
                SDImageFormat format = [NSData sd_imageFormatForImageData:*data];
                NSData *imageData = [self encodedDataWithImage:scaledDownImage format:format];
                if (imageData) {
                    *data = imageData;
                }
            }
            return scaledDownImage;
        }
    #endif
    }

逻辑比较简单，先判断是否设置了ScaleDownLargeImage，如果没有直接调用sd_decompressedImageWithImage:这个方法，有的话调用sd_decompressedAndScaledDownImageWithImage:，这个方法是用来避免大图带来的内存峰值问题的有效手段，稍后会说怎么实现的。现在来看一下sd_decompressedImageWithImage:方法：

    - (nullable UIImage *)sd_decompressedImageWithImage:(nullable UIImage *)image {
        if (![[self class] shouldDecodeImage:image]) {
            return image;
        }
        
        // autorelease the bitmap context and all vars to help system to free memory when there are memory warning.
        // on iOS7, do not forget to call [[SDImageCache sharedImageCache] clearMemory];
        @autoreleasepool{
            
            CGImageRef imageRef = image.CGImage;
            CGColorSpaceRef colorspaceRef = [[self class] colorSpaceForImageRef:imageRef];
            
            size_t width = CGImageGetWidth(imageRef);
            size_t height = CGImageGetHeight(imageRef);
            
            // kCGImageAlphaNone is not supported in CGBitmapContextCreate.
            // Since the original image here has no alpha info, use kCGImageAlphaNoneSkipLast
            // to create bitmap graphics contexts without alpha info.
            CGContextRef context = CGBitmapContextCreate(NULL,
                                                         width,
                                                         height,
                                                         kBitsPerComponent,
                                                         0,
                                                         colorspaceRef,
                                                         kCGBitmapByteOrderDefault|kCGImageAlphaNoneSkipLast);
            if (context == NULL) {
                return image;
            }
            
            // Draw the image into the context and retrieve the new bitmap image without alpha
            CGContextDrawImage(context, CGRectMake(0, 0, width, height), imageRef);
            CGImageRef imageRefWithoutAlpha = CGBitmapContextCreateImage(context);
            UIImage *imageWithoutAlpha = [[UIImage alloc] initWithCGImage:imageRefWithoutAlpha scale:image.scale orientation:image.imageOrientation];
            CGContextRelease(context);
            CGImageRelease(imageRefWithoutAlpha);
            
            return imageWithoutAlpha;
        }
    }

逻辑比较简单，先得到原来的未被解压的位图和色彩空间，再根据色彩空间创造一个新的位图上下文，再用Core Graphics的API在画布上进行绘制，从而得到一个解压之后的位图。最后不要忘记把之前持有的位图释放掉。

注意如果包含了alpha通道是不能够被解压缩的，因为包含alpha通道的图片要交给GPU进行处理，GPU要根据图层来算出一个点的混合像素值。

上述方法虽然把内存释放掉了，但是还是不可以避免内存峰值的问题，在解压的一瞬间内存还是会飙升，我们一方面可以通过禁用解压缩来避免内存峰值（不过之后解压缩不可避免），另一方面可以通过设置SDWebImageCoderScaleDownLargeImagesKey来采用SDWebImage提供的优化逻辑，它采用了一个非常优雅的方案来规避内存峰值的问题，主要思想如下：

- 判断原图的大小是否超过了指定的大小，如果超过了就指定一个比较小的目标位图，并计算出scale的值。
- 设置每次copy原图数据使用的块（tile）的大小，并根据scale来设置每次输出到目标位图所使用的块的大小。
- 每次在循环中对原图进行部分拷贝, 并将拷贝的部分数据输出到目标位图的对应位置，最后循环结束释放临时变量，有效避免峰值。
- 最后得到一个完整的目标位图，生成UIImage，释放位图，返回。

下面来看一下它的实现，代码比较长，已在关键地方做了注释：

    - (nullable UIImage *)sd_decompressedAndScaledDownImageWithImage:(nullable UIImage *)image {
        if (![[self class] shouldDecodeImage:image]) {
            return image;
        }
        
        if (![[self class] shouldScaleDownImage:image]) {
            return [self sd_decompressedImageWithImage:image];
        }
        
        CGContextRef destContext;
        
        // autorelease the bitmap context and all vars to help system to free memory when there are memory warning.
        // on iOS7, do not forget to call [[SDImageCache sharedImageCache] clearMemory];
        @autoreleasepool {
            CGImageRef sourceImageRef = image.CGImage;
            // 得到原图的信息，width，height以及像素数。
            CGSize sourceResolution = CGSizeZero;
            sourceResolution.width = CGImageGetWidth(sourceImageRef);
            sourceResolution.height = CGImageGetHeight(sourceImageRef);
            float sourceTotalPixels = sourceResolution.width * sourceResolution.height;
            // 算出scale, kDestTotalPixels是定义好的常量。
            float imageScale = kDestTotalPixels / sourceTotalPixels;
            CGSize destResolution = CGSizeZero;
            destResolution.width = (int)(sourceResolution.width*imageScale);
            destResolution.height = (int)(sourceResolution.height*imageScale);
            
            // current color space
            CGColorSpaceRef colorspaceRef = [[self class] colorSpaceForImageRef:sourceImageRef];
            
            // kCGImageAlphaNone is not supported in CGBitmapContextCreate.
            // Since the original image here has no alpha info, use kCGImageAlphaNoneSkipLast
            // to create bitmap graphics contexts without alpha info.
            destContext = CGBitmapContextCreate(NULL,
                                                destResolution.width,
                                                destResolution.height,
                                                kBitsPerComponent,
                                                0,
                                                colorspaceRef,
                                                kCGBitmapByteOrderDefault|kCGImageAlphaNoneSkipLast);
            
            if (destContext == NULL) {
                return image;
            }
            // 插值设置
            CGContextSetInterpolationQuality(destContext, kCGInterpolationHigh);
            
            // 设置用于拷贝原图数据的块和输出到目的位图的块，由于种种原因，拷贝原图数据的块的宽度必须是和原图相同的，输出到目的位图的块的宽度必须是和目的位图的宽度是相同的。
            CGRect sourceTile = CGRectZero;
            sourceTile.size.width = sourceResolution.width;
            // kTileTotalPixels也是一个常量
            sourceTile.size.height = (int)(kTileTotalPixels / sourceTile.size.width );
            sourceTile.origin.x = 0.0f;

            CGRect destTile;
            destTile.size.width = destResolution.width;
            destTile.size.height = sourceTile.size.height * imageScale;
            destTile.origin.x = 0.0f;
            // overlap暂且不知道是干嘛的，猜是为了插值用的吧。
            float sourceSeemOverlap = (int)((kDestSeemOverlap/destResolution.height)*sourceResolution.height);
            CGImageRef sourceTileImageRef;
            // 算出循环的次数
            int iterations = (int)( sourceResolution.height / sourceTile.size.height );
            int remainder = (int)sourceResolution.height % (int)sourceTile.size.height;
            if(remainder) {
                iterations++;
            }
            
            float sourceTileHeightMinusOverlap = sourceTile.size.height;
            sourceTile.size.height += sourceSeemOverlap;
            destTile.size.height += kDestSeemOverlap;
            for( int y = 0; y < iterations; ++y ) {
                @autoreleasepool {
                    // 每次循环，对于sourceTile的y是递增的，destTile的y是递减的，因为UIKit和Core Graphics用的坐标系是反过来的。
                    sourceTile.origin.y = y * sourceTileHeightMinusOverlap + sourceSeemOverlap;
                    destTile.origin.y = destResolution.height - (( y + 1 ) * sourceTileHeightMinusOverlap * imageScale + kDestSeemOverlap);
                    // 对原图进行部分拷贝
                    sourceTileImageRef = CGImageCreateWithImageInRect( sourceImageRef, sourceTile );
                    if( y == iterations - 1 && remainder ) {
                        float dify = destTile.size.height;
                        destTile.size.height = CGImageGetHeight( sourceTileImageRef ) * imageScale;
                        dify -= destTile.size.height;
                        destTile.origin.y += dify;
                    }
                    // 绘制位图
                    CGContextDrawImage( destContext, destTile, sourceTileImageRef );
                    // 释放，每次在for-loop中释放，避免内存峰值
                    CGImageRelease( sourceTileImageRef );
                }
            }
            
            // 得到最终位图之后
            CGImageRef destImageRef = CGBitmapContextCreateImage(destContext);
            CGContextRelease(destContext);
            if (destImageRef == NULL) {
                return image;
            }
            UIImage *destImage = [[UIImage alloc] initWithCGImage:destImageRef scale:image.scale orientation:image.imageOrientation];
            CGImageRelease(destImageRef);
            if (destImage == nil) {
                return image;
            }
            return destImage;
        }
    }

看上去很长，大致逻辑是和我上面所说的是一致的。此函数还用到了很多定义好的常量，用于定义目标位图的大小，每次使用的块的大小等等，其实就是几个基本的数学运算，这里不再细谈。

此方案几乎完美地解决了前几个版本加载大图崩溃的问题，建议每次加载大图开启SDWebImageScaleDownLargeImages选项来进行优化。

关于编码和解码，实际上就是NSData类型和UIImage类型的转换，逻辑比较简单，在编解码时需要对图片的类型进行判断，这里不再细谈。不同的图片类型也对应不同的Corder，GIF动图对应的是SDWebImageGIFCorder，WebP交给SDWebImageWebPCorder，PNG和JPEG交给SDWebImageImageIOCorder。

# 几个比较有意思的地方

下面来记录几个比较有意思的地方。之后想到了再加。

## 用信号量代替锁

在SDWebImage当中主要的锁的实现委托给了信号量dispatch_semaphore_t, 并定义了两个宏来方便加解锁：

    #define LOCK(lock) dispatch_semaphore_wait(lock, DISPATCH_TIME_FOREVER);
    #define UNLOCK(lock) dispatch_semaphore_signal(lock);

采用信号量的原因是因为它的加锁解锁性能是最优的，比@synchronized，NSLock等等快得多，而且在等待时不会占用CPU时间，具体的评测请看：[深入理解iOS开发中的锁](https://bestswifter.com/ios-lock/)

## 后台清理任务

SDImageCache会监听退到后台和程序退出的事件，清理过期的文件，看一下后台退出是怎么实现的：

    - (void)backgroundDeleteOldFiles {
        Class UIApplicationClass = NSClassFromString(@"UIApplication");
        if(!UIApplicationClass || ![UIApplicationClass respondsToSelector:@selector(sharedApplication)]) {
            return;
        }
        UIApplication *application = [UIApplication performSelector:@selector(sharedApplication)];
        __block UIBackgroundTaskIdentifier bgTask = [application beginBackgroundTaskWithExpirationHandler:^{
            // Clean up any unfinished task business by marking where you
            // stopped or ending the task outright.
            [application endBackgroundTask:bgTask];
            bgTask = UIBackgroundTaskInvalid;
        }];

        // Start the long-running task and return immediately.
        [self deleteOldFilesWithCompletionBlock:^{
            [application endBackgroundTask:bgTask];
            bgTask = UIBackgroundTaskInvalid;
        }];
    }

清理文件的大致逻辑如下：先根据我们所设置的maxCacheAge清除过期的文件，再比较缓存的大小是否比设置的maxCacheSize大，如果大的话，再对所剩的文件根据日期的降序进行排序，依次删除，直到所剩文件体积到达合适的大小（默认为maxCacheSize / 2）。

## 收到内存警告的处理

SDImageCache会在初始化时注册收到内存警告的Notification，然后清除掉在内存当中的缓存：

    - (void)didReceiveMemoryWarning:(NSNotification *)notification {
        // Only remove cache, but keep weak cache
        [super removeAllObjects];
    }

这样可以清除NSCache在内存中的缓存，但是自己的weakCache表里面还保存着image的weak指针，这样做的好处是：**收到内存警告后，image可能因为在某个UIImageView里面而没有被释放，这样下次加载的时候直接就可以从weakCache表中读出来，并且同步到NSCache当中，而不用重新从磁盘读取**：

    - (id)objectForKey:(id)key {
        id obj = [super objectForKey:key];
        if (key && !obj) {
            // Check weak cache
            LOCK(self.weakCacheLock);
            obj = [self.weakCache objectForKey:key];
            UNLOCK(self.weakCacheLock);
            if (obj) {
                // 从weakCache中读出来，sync到NSCache里面
                // Sync cache
                NSUInteger cost = 0;
                if ([obj isKindOfClass:[UIImage class]]) {
                    cost = SDCacheCostForImage(obj);
                }
                [super setObject:obj forKey:key cost:cost];
            }
        }
        return obj;
    }

注意weakCache是一个NSMapTable类型，其key是strong的，value是weak的：

    self.weakCache = [[NSMapTable alloc] initWithKeyOptions:NSPointerFunctionsStrongMemory valueOptions:NSPointerFunctionsWeakMemory capacity:0];

## 用GCD模拟锁

遵从《Effective Objective-C》的原则之一：用GCD操作来模拟锁来获取底层的性能优化。前面说过SDWebImageCorder是交给SDWebImageCodersManager来管理的，它内部有一个mutableCoders成员，对它的访问和修改使用一个并行队列来模拟的锁，大致思路是这样的：

- 对于访问操作是可以并行的，此时可以对并行队列执行dispatch_sync操作。
- 对于修改操作应该是原子性的，此时可以对并行队列执行dispatch_barrier_sync操作。

代码如下所示：

    - (void)addCoder:(nonnull id<SDWebImageCoder>)coder {
        if ([coder conformsToProtocol:@protocol(SDWebImageCoder)]) {
            dispatch_barrier_sync(self.mutableCodersAccessQueue, ^{
                [self.mutableCoders addObject:coder];
            });
        }
    }

    - (void)removeCoder:(nonnull id<SDWebImageCoder>)coder {
        dispatch_barrier_sync(self.mutableCodersAccessQueue, ^{
            [self.mutableCoders removeObject:coder];
        });
    }

    - (NSArray<SDWebImageCoder> *)coders {
        __block NSArray<SDWebImageCoder> *sortedCoders = nil;
        dispatch_sync(self.mutableCodersAccessQueue, ^{
            sortedCoders = (NSArray<SDWebImageCoder> *)[[[self.mutableCoders copy] reverseObjectEnumerator] allObjects];
        });
        return sortedCoders;
    }

    - (void)setCoders:(NSArray<SDWebImageCoder> *)coders {
        dispatch_barrier_sync(self.mutableCodersAccessQueue, ^{
            self.mutableCoders = [coders mutableCopy];
        });
    }

## dispatch_queue_async_safe

同样在《Effective Objective-C》一书中写道：dispatch_get_current_queue不应该被使用，因为它已经被弃用，在官方文档中的说法为：

    Recommended for debugging and logging purposes only:
    The code must not make any assumptions about the queue returned, unless it
    is one of the global queues or a queue the code has itself created.
    The code must not assume that synchronous execution onto a queue is safe
    from deadlock if that queue is not the one returned by
    dispatch_get_current_queue().
     
    When dispatch_get_current_queue() is called on the main thread, it may
    or may not return the same value as dispatch_get_main_queue(). Comparing
    the two is not a valid way to test whether code is executing on the
    main thread (see dispatch_assert_queue() and dispatch_assert_queue_not()).
     
    This function is deprecated and will be removed in a future release.

在SDWebImage中有一个宏叫做dispatch_queue_async_safe，它可以比较当前所执行的queue和目标queue是不是同一个，如果是则立即执行，如果不是则执行dispatch_async方法：

    #define dispatch_queue_async_safe(queue, block)\
        if (strcmp(dispatch_queue_get_label(DISPATCH_CURRENT_QUEUE_LABEL), dispatch_queue_get_label(queue)) == 0) {\
            block();\
        } else {\
            dispatch_async(queue, block);\
        }
比较的方式是比较两个queue的label是不是相同的，通过dispatch_queue_get_label(DISPATCH_CURRENT_QUEUE_LABEL)来得到当前的queue的label，再和目标queue的label进行比较。这是一种比较靠谱的方法，label是我们在创建queue的时候指定的，因此在创建queue的时候指定一个唯一的名字特别重要，最好是前面加上自己的包名，然后再加上类名。

我们也可以通过queue的specific数据来进行判断，specific数据还可以根据queue的target向上寻找，有效地避免了queue的死锁问题，详情参见《Effective Objective-C》第46条。