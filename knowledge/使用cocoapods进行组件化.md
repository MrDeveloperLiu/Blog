#### 组件化

> Tips：近些年来，搞手机（iOS）开发，听到最多的词汇就是路由+组件化

```
But，'路由'暂不在本文讨论范围内。那么我们来学习一下，使用cocoapods制作自己的私有库
```

* 前提准备

  ```
  1、首先你需要注册一个github的账号，已有可忽略
  2、安装了CocoaPods，没有的自行百度
  ```

* 创建repo

  ```
  1、创建第一个repo存放源码
  2、创建第二个repo存放podspec
  ```

  示例：

  @see [源码链接](https://github.com/MrDeveloperLiu/PirvateRepository)

  @see [Spec链接](https://github.com/MrDeveloperLiu/Specs)

* 本地创建repo

  下一步：我们把前面的源码仓库与spec仓库添加到本地

  ```
  pod repo add [REPO_NAME] [URL]
  ```

  那么我们通过以上命令，将repo添加到本地；例如：

  ```
  pod repo add source-repo https://github.com/MrDeveloperLiu/PirvateRepository.git
  pod repo add specs-repo https://github.com/MrDeveloperLiu/Specs.git
  ```

  > 这样：我们就把仓库关联到了本地

* 存放源码

  > 第一步：
  >
  > ​	我们源码仓库中将源码上传，最好是 [库名字]/[Classes文件夹] 例如：CollectionUI/Classes 
  >
  > 第二部：
  >
  > ​	我们在Sepc仓库中创建podspec文件
  >
  > ```
  > pod spec create [库名字]
  > ```

* 修改podspec文件

  ```
  Pod::Spec.new do |spec|
  	1、（库名字)
    spec.name         = "CollectionUI" 
    2、（版本号）
    spec.version      = "1.0.5" 
    3、（简述)
    spec.summary      = "CollectionUI: That is some useful wedigt for UITableView and UICollectionView" 
    4、（描述）
    spec.description  = <<-DESC 
      Hello There~
       We need to coding easy!
       We provide UITableView and UICollectionView's useful classes
       And there may take more easy
                     DESC
  	5、（主页地址）
    spec.homepage     = "https://github.com/MrDeveloperLiu/PirvateRepository.git"
    6、（开源许可）
    spec.license      = { :type => "MIT", :file => "LICENSE" }
    7、（作者信息）
    spec.author             = { "刘杨" => "164182408@qq.com" }
    8、（平台信息）
    spec.platform     = :ios, "10.0"
    9、（Swift版本号）
    spec.swift_versions = ['5.0']
  	10、（多平台开发版本支持）
    #  When using multiple platforms
    # spec.ios.deployment_target = "5.0"
    # spec.osx.deployment_target = "10.7"
    # spec.watchos.deployment_target = "2.0"
    # spec.tvos.deployment_target = "9.0"
  	11、（源码地址）
    spec.source       = { :git => "https://github.com/MrDeveloperLiu/PirvateRepository.git", :tag => "#{spec.version}" }
    12、（源码路径）
    spec.source_files  = "CollectionUI/Classes/**/*"
    13、（源码剔除路径-该文件夹下的文件不会被引入）
    spec.exclude_files = "CollectionUI/Classes/Exclude"
    14、（ARC）
    spec.requires_arc = true
    
  end
  ```

  > 注意：
  >
  > ​	笔者在这里被坑了好久。。。
  >
  > ​	1、spec.source       = { :git => "https://github.com/MrDeveloperLiu/PirvateRepository.git"} 此处的:git的值，实际上源码对应的git仓库地址
  >
  > ​	2、spec.source_files  = "CollectionUI/Classes/\*" 此处的源码路径可以多选，比如 spec.source_files  = "CollectionUI/Classes/\*", "CollectionUI/Classes/\*\*/\*"还可以指定添加文件，例如：spec.source_files  = "CollectionUI/Classes/\*", "CollectionUI/Classes/\*\*/\*.{h, m}", "CollectionUI/Classes/\*\*/\*.swift"

* 验证是否成功

  ```
  pod spec lint 或者 pod lib lint (可选项 --allow-warnings、--verbose)
  ```

  一般如果有错误的话，那么会中断，但是有警告的话也会中断。那么我们需要添加--allow-warnings去忽略警告

* 更新仓库

  ```
  pod repo push [REPO_NAME] [库.podspec] --allow-warnings
  ```

  我们需要这个操作将spec文件发布到私有库中

* 注意：spec.version      = "1.0.5" 实际上对应的是你源码的tag号



Ok，快来试一试吧

那么最终，我们在工程的podfile文件中。通过引入

```
# source 'https://github.com/CocoaPods/Spec.git'
source 'https://github.com/MrDeveloperLiu/Specs.git'

platform:ios,'10.0'
inhibit_all_warnings!
# use_frameworks!


def private_repo
  pod 'CollectionUI'
end

def public_repo
  pod 'RxSwift'
  pod 'RxCocoa'
  pod 'RxDataSources'
  pod 'SnapKit'
  pod 'Kingfisher'
  pod 'Alamofire'
  pod 'CleanJSON'
end

target 'RTSwift' do
    private_repo
    public_repo
end
```

执行

```
pod install
```

稍等片刻，就会成功！ps：许多人说需要同时添加官方源source 'https://github.com/CocoaPods/Spec.git'，其实我觉得加不加好像效果不大，不过还是先写上。快去试一试吧
