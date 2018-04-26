FPS 指示器

## 写在最前
CADisplayLink是CoreAnimation提供的另一个类似于NSTimer的类，总是在屏幕完成一次更新之前启动，接口设计和NSTimer很类似，所以它实际上就是一个内置实现的替代，但是和timeInterval以秒为单位不同，CADisplayLink有一个整形的frameInterval属性，指定了间隔多少帧之后才执行。默认值为1，意味着每次屏幕更新之前都会执行一次。但是如果代码执行超过1/60秒，可以制定frameInterval为2，就是说每隔一帧执行一次，或者设置为3，也就是1秒钟执行20次，等等

## 设计思路
既然CADisplayLink可以以屏幕刷新的频率调用制定selector，而且iOS系统中正常的屏幕刷新频率为60Hz，那么只要在这个方法中统计在1秒内执行了多次，再计算`次数／时间`就可以得出当前屏幕的FPS
talk is simple， show you the code

```
- (void)setupDisplayLink {
    //创建CADisplayLink，并添加到当前run loop的NSRunLoopCommonModes
    _displayLink = [CADisplayLink displayLinkWithTarget:self selector:@selector(linkTicks:)];
    [_displayLink addToRunLoop:[NSRunLoop currentRunLoop] forMode:NSRunLoopCommonModes];
}

- (void)linkTicks:(CADisplayLink *)link
{
    //执行次数
    _scheduleTimes ++;

    //当前时间戳
    if(_timestamp == 0){
        _timestamp = link.timestamp;
    }
    CFTimeInterval timePassed = link.timestamp - _timestamp;

    if(timePassed >= 1.f)
        //fps
        CGFloat fps = _scheduleTimes/timePassed; 
        printf("fps:%.1f, timePassed:%f\n", fps, timePassed);

        //reset
        _timestamp = link.timestamp;
        _scheduleTimes = 0;
    }
}
```

> 一切没有在真机测试过的功能都是耍流氓
> 测试和Instrument进行对比

简单测试下发现还不错

## 意想不到

真机都有其极限条件，当遇到极端情况下会怎么样呢
show you code

```
@implementation DemoViewController
- (void)viewDidLoad {
    [super viewDidLoad];

    //1000条记录，每条记录包含一个名字和一个头像
    NSMutableArray *array = [NSMutableArray array];
    for (int i = 0; i< 1000; i++) {
        [array addObject:@{@"name": [self randomName], @"image": [self randomAvatar]}];
    }

    self.items = array;
    [self.tableView registerClass:[UITableViewCell class] forCellReuseIdentifier:@"Cell"];
}

#pragma mark - UITableViewDataSource
- (NSInteger)tableView:(UITableView *)tableView numberOfRowsInSection:(NSInteger)section {
    return [self.items count];
}

- (UITableViewCell *)tableView:(UITableView *)tableView cellForRowAtIndexPath:(NSIndexPath *)indexPath {
    UITableViewCell *cell = [self.tableView dequeueReusableCellWithIdentifier:@"Cell" forIndexPath:indexPath];

    NSDictionary *item = self.items[indexPath.row];
    NSString *filePath = [[NSBundle mainBundle] pathForResource:item[@"image"] ofType:@"png"];

    //name and image
    cell.imageView.image = [UIImage imageWithContentsOfFile:filePath];
    cell.textLabel.text = item[@"name"];

    //image shadow
    cell.imageView.layer.shadowOffset = CGSizeMake(0, 5);
    cell.imageView.layer.shadowOpacity = 1;
    cell.imageView.layer.cornerRadius = 5.0f;
    cell.imageView.layer.masksToBounds = YES;

    //text shadow
    cell.textLabel.backgroundColor = [UIColor clearColor];
    cell.textLabel.layer.shadowOffset = CGSizeMake(0, 2);
    cell.textLabel.layer.shadowOpacity = 1;

    return cell;
}

- (NSString *)randomName {
    NSArray *first = @[@"Alice",@"Bob",@"Bill"];
    NSArray *last = @[@"Appleseed",@"Bandicoot",@"Caravan"];
    NSUInteger index1 = (rand()/(double)INT_MAX) * [first count];
    NSUInteger index2 = (rand()/(double)INT_MAX) * [last count];
    return [NSString stringWithFormat:@"%@ %@", first[index1], last[index2]];
}

- (NSString *)randomAvatar {
    NSArray *images = @[@"A",@"B",@"C"];
    NSUInteger index = (rand()/(double)INT_MAX) * [images count];
    return images[index];
}
@end
```

上述测试代码中包含1000条数据，每条数据包含了一张图片和一段文字，用于在列表的cell里显示，每张图片都设置里圆角，且图片于文本都设置了阴影
显而易见，一开始快速滑动列表是，FPS下降到肉眼能够识别的程度，FPS指示器显示的数字也同步下降到个位数
继续快速滑动列表，看得出列表滑动依然很不流畅，但FPS指示器却保持较高的FPS，通过和Instrument中的core animation FPS显示出现较大的差距

iOS中每一帧画面的生成是一个复杂的过程，大致的过程如下

* 系统根据你的代码，设置布局各个元素的位置（frame，AutoLayout）、属性（颜色、透明度、阴影等）
* CPU对需要提前绘制的元素、图形使用Core Graphics进行绘制
* CPU将一切需要绘制到屏幕上的内容（包括解压后的图片）打包发送到GPU
* GPU对内容进行计算绘制，显示到屏幕上

因此，总结出Demo中造成性能下降的原因主要有：

* 滑动列表时（即时是慢速滑动），GPU也需要计算图片、文本的动态阴影的位置和形状来进行阴影的绘制，此时GPU将形成性能瓶颈，能明显感受到FPS的下降
* 快速滑动列表时Cell每次在显示前都需要通过imageWithContentsOfFile从硬盘加载图片并解压，此时文件的IO，图片的解压让CPU也遇到性能瓶颈，是主线程无法流畅执行，让FPS雪上加霜


>CADisplayLink运行在主线程RunLoop之中，RunLoop中所管理的任务的调度时机受任务所处的RunLoopMode和CPU的繁忙程度所影响。
在第二个原因中受文件IO、解压图片的影响，RunLoop 自然无法保证CADisplayLink被调用的次数达到每秒60次，这里的调用频率正是我们的FPS指示器中所显示FPS。
而在第一个原因中主要瓶颈在于GPU，即使RunLoop能保持每秒60次调用CADisplayLink，也无法说明此时的屏幕刷新率能达到60FPS（Core Animation通过与OpenGl打交道控制GPU进行屏幕绘制），也正因为这样FPS指示器显示55+的FPS，但Instrument中的Core Animation FPS 却很低

## 参考文档

[iOS中基于CADisplayLink的FPS指示器详解](http://www.jianshu.com/p/86705c95c224)

