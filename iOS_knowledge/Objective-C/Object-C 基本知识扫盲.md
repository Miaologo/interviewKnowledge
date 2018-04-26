# Object-C 基本知识扫盲
## 盲点
静态方法（类方法？）

self = [super init]

UITableViewCell *cell = [tableview dequeueReusableCellWithIdentifier:defineString]
修改为：
UITableViewCell *cell = [tableview cellForRowAtIndexPath:indexPath];


多线程
![多线程导图](media/14689799366441/%E5%A4%9A%E7%BA%BF%E7%A8%8B%E5%AF%BC%E5%9B%BE.png)


















## 参考文章
<http://www.jianshu.com/p/5d2163640e26>
<http://www.jianshu.com/p/f6ee822ef84a>

