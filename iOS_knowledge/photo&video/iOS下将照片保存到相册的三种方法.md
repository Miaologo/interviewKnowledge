iOS下将照片保存到相册的三种方法

## 方法一

使用UIImageWriteToSavedPhotosAlbum函数将图片保存到相册，如：

```
- (void)loadImageFinished:(UIImage *)image
{
    UIImageWriteToSavedPhotosAlbum(image, self, @selector(image:didFinishSavingWithError:contextInfo:), (__bridge void *)self);
}

- (void)image:(UIImage *)image didFinishSavingWithError:(NSError *)error contextInfo:(void *)contextInfo
{

    NSLog(@"image = %@, error = %@, contextInfo = %@", image, error, contextInfo);
}
```

第一个参数是要保存到相册的图片对象

第二个参数是保存完成后回调的目标对象

第三个参数就是保存完成后回调到目标对象的哪个方法中，方法的声明要如代码中所示的：

```
- (void)image:(UIImage *)image didFinishSavingWithError:(NSError *)error contextInfo:(void *)contextInfo;
```

第四个参数在保存完成后，会原封不动地传回到回调方法的contextInfo参数中。

## 方法二

使用AssetsLibrary框架中的ALAssetsLibrary类来实现。具体代码如下：

```
- (void)loadImageFinished:(UIImage *)image
{
    __block ALAssetsLibrary *lib = [[ALAssetsLibrary alloc] init];
    [lib writeImageToSavedPhotosAlbum:image.CGImage metadata:nil completionBlock:^(NSURL *assetURL, NSError *error) {

        NSLog(@"assetURL = %@, error = %@", assetURL, error);
        lib = nil;

    }];
}
```

使用了ALAssetsLibrary类的writeImageToSavedPhotosAlbum:metadata:completionBlock:方法实现。其中第一个参数是一个CGImageRef的对象，表示要传入的图片。第二个参数是图片的一些属性，这里没有设置所以传入nil。最后一个completionBlock是保存完成后的回调，在这个回调中可以取到保存后的图片路径以及保存失败时的错误信息。

> 注意：使用该类时需要导入AssetsLibrary.framework。而且该类需要在iOS4.0以上可以使用，但是在iOS9.0之后就被标记为过时方法。官方建议使用Photos.framework中的PHPhotoLibrary进行代替，也就是下面所说的第三种方法。

## 方法三

使用Photos框架的PHPhotoLibrary类来实现保存到相册功能。代码如下：

```
- (void)loadImageFinished:(UIImage *)image
{
    [[PHPhotoLibrary sharedPhotoLibrary] performChanges:^{

         /写入图片到相册
         PHAssetChangeRequest *req = [PHAssetChangeRequest creationRequestForAssetFromImage:image];


     } completionHandler:^(BOOL success, NSError * _Nullable error) {

         NSLog(@"success = %d, error = %@", success, error);

    }];
}
```

该例子中先调用PHPhotoLibrary类的performChanges:completionHandler:方法，然后在它的changeBlock中，通过PHAssetChangeRequest类的creationRequestForAssetFromImage:方法传入一个图片对象即可实现保存到相册的功能。然后completionHandler中会告诉我们是否操作成功。

## 进阶使用：得到保存到相册的图片对象

也许会有人需要在保存相册后得到图片的PHAsset对象来进行后续操作（昨天刚好碰到有朋友遇到这样的问题）。那么，这里对上面例子进行改进，在创建PHAssetChangeRequest后将它的placeholderForCreatedAsset属性的localIdentifier属性保存到一个数组中，等待操作完成后再通过这个数组来查找刚刚添加的图片对象。请看下面栗子：

```
- (void)loadImageFinished:(UIImage *)image
{
    NSMutableArray *imageIds = [NSMutableArray array];
        [[PHPhotoLibrary sharedPhotoLibrary] performChanges:^{

            //写入图片到相册
            PHAssetChangeRequest *req = [PHAssetChangeRequest creationRequestForAssetFromImage:image];
            //记录本地标识，等待完成后取到相册中的图片对象
            [imageIds addObject:req.placeholderForCreatedAsset.localIdentifier];


        } completionHandler:^(BOOL success, NSError * _Nullable error) {

            NSLog(@"success = %d, error = %@", success, error);

            if (success)
            {
                //成功后取相册中的图片对象
                __block PHAsset *imageAsset = nil;
                PHFetchResult *result = [PHAsset fetchAssetsWithLocalIdentifiers:imageIds options:nil];
                [result enumerateObjectsUsingBlock:^(PHAsset * _Nonnull obj, NSUInteger idx, BOOL * _Nonnull stop) {

                    imageAsset = obj;
                    *stop = YES;

                }];

                if (imageAsset)
                {
                    //加载图片数据
                    [[PHImageManager defaultManager] requestImageDataForAsset:imageAsset
                          options:nil
                          resultHandler:^(NSData * _Nullable imageData, NSString * _Nullable dataUTI, UIImageOrientation orientation, NSDictionary * _Nullable info) {

                               NSLog("imageData = %@", imageData);

                          }];
                }
            }

        }];
}
```

## 总结

第一种方式是最常用的，使用起来很方便，传入UIImage就可以了，也不需要担心iOS不同版本的问题。唯一缺点就是无法找到对应添加的图片。

第二种方式是iOS4之后加入的，在iOS9后又不推荐使用了。他也提供了很直观的方式来保存图片，并且也能够取到保存后相对应的图片路径。

第三种方式是iOS8之后加入的，他的使用稍微复杂一点，但是它允许进行批量的操作，例如添加、修改、删除等。如果要做更加复杂的操作的话，这种方式是比较推荐的方式。

