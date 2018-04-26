# 私有pod 建立

## 创建project
建立远程git仓库
pod lib creat "pod name"

## podspec

修改podspec文件

验证spec文件
pod spec lint --sources='ssh://git@git.sankuai.com/ios/specs.git,https://github.com/CocoaPods/Specs' --verbose
后面的参数可选，sources中各个url不能有空格间隔

验证 lib
pod lib lint --sources='ssh://git@git.sankuai.com/myios/specs.git,https://github.com/CocoaPods/Specs,http://git.sankuai.com/ios/specs.git' --verbose

提交到远程lib spec
pod repo 查看本地repo列表
pod repo push “远程spec名” “podspec”
pod repo push sankuai-specs MAYPlatform.podspec


