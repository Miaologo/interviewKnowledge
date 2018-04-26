# Animation with AutoLayout

从AutoLayout出现后，终于解决了不同屏幕的适配问题，因此在代码中常见到AutoLayout的代码
在进行animation设计时，当你把AutoLayout相关的代码写在对应的animation部分，进行测试发现animation并不是你所期望的那样。
具体解决办法将`[view layoutIfNeeded]`放在对应的animation代码部分，关于AutoLayout的代码放在外面

如设置searchbar的缩放动画

```
//动画开始前
self.searchBar.mas_updateConstraints({ (make) in
            make.left.equalTo()(NSNumber(float: 44))
        })
```

```
//动画相关代码
self.searchBar.mas_updateConstraints({ (make) in
            make.left.equalTo()(NSNumber(float: 0))
        })
UIView.animateWithDuration(0.5) { 
            self.searchBar.layoutIfNeeded()
        }
```

