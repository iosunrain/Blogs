# iOS自动打包之fastlane

在平时的iOS开发中，打包测试、上架是不可避免的操作，而相对于Android打包，iOS则更加繁琐耗时，所以，当发现还有fastlane这种一键打包的工具时，就一定不能放过！
![1](https://upload-images.jianshu.io/upload_images/962634-3901b9c8a6c8ba14..png?imageMogr2/auto-orient/strip%7CimageView2/2/w/871/format/webp)
>Fastlane是一套使用Ruby写的自动化工具集，旨在简化Android和iOS的部署过程，自动化你的工作流。它可以简化一些乏味、单调、重复的工作，像截图、代码签名以及发布App

[GitHub](https://link.jianshu.com/?t=https%3A%2F%2Fgithub.com%2Ffastlane%2Ffastlane)
[官方安装指南](https://docs.fastlane.tools/getting-started/ios/setup/)
官方文档打开不是很稳定，下面总结下自己安装的经验
## 一、安装fastlane
`sudo gem install fastlane -NV`或是`brew cask install fastlane`我这里使用gem安装的
注意，gem最新地址是`https://gems.ruby-china.com`可以用`gem sources -l`查看
安装完了执行 `fastlane --version`，确认下是否安装完成和当前使用的版本号。
我的版本是
![fastlane](https://upload-images.jianshu.io/upload_images/6579721-52a618b7d0a958d6.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

## 二、初始化fastlane
cd到你的项目目录执行
`fastlane init`
![借图](https://upload-images.jianshu.io/upload_images/962634-70bb268897a56c6c.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1000/format/webp)
这里会弹出四个选项，问你想要用Fastlane做什么？ 之前的老版本是不用选择的。选几都行，后续我们自行根据需求完善就可以，这里我选的是3。

如果你的工程是用cocoapods的那么`可能`会提示让你勾选工程的Scheme，步骤就是打开你的xcode，点击Manage Schemes，在一堆三方库中找到你的项目Scheme，在后面的多选框中进行勾选，然后`rm -rf fastlane`文件夹，重新`fastlane init`一下就不会报错了。
接着会提示你输入开发者账号和密码。
`[20:48:55]: Please enter your Apple ID developer credentials`
`[20:48:55]: Apple ID Username:`
登录成功后会提示你是否需要下载你的App的metadata。点y等待就可以。
*过程中可能会自行bundle install 如果时间过久，我选择ctrl + z *
## 三、文件
初始化成功后会在当前工程目录生成一个fastlane文件夹，文件目录为下。
![image.png](https://upload-images.jianshu.io/upload_images/6579721-b9de0f985a15be60.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
其中metadata和screenshots分别对应App元数据和商店应用截图。

Appfile主要存放App的apple_id team_id app_identifier等信息

Deliverfile中为发布的配置信息，一般情况用不到。

**Fastfile是我们最应该关注的文件，也是我们的工作文件。**

```
default_platform(:ios)

platform :ios do
  desc "Push a new release build to the App Store"
  lane :release do
    build_app(workspace: "xxxx.xcworkspace", scheme: "xxx")
    upload_to_app_store
  end
  desc "打包到pgy"
  lane :beta do |options|
  gym(
  clean:true, #打包前clean项目
  export_method: "development", #导出方式
  scheme:"xxxxx", #scheme
  configuration: "Debug",#环境
  output_directory:"./app",#ipa的存放目录
  output_name:"pugongying"#输出ipa的文件名为当前的build号
  )
 pgyer(api_key: "2b08ec0xxxxx95bd7c2d", user_key: "36811b4xxxx0be2785e")
  end
end
```
每个lane对应一份操作，可以写好多份，不通情况使用不同的lane。
我这边是打包上传蒲公英的例子，
打开终端输入`fastlane add_plugin pgyer`
更多信息查看[蒲公英文档](https://link.jianshu.com/?t=https%3A%2F%2Fwww.pgyer.com%2Fdoc%2Fview%2Ffastlane)
### 执行
在工作目录的终端执行
`fastlane beta desc:测试打包`

下面是执行结果：

![成功.png](https://upload-images.jianshu.io/upload_images/6579721-aea605e952655c92.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

### 其他的一些小提示
1. 可以在before_all中做一些前置操作，比如进行build号的更新，我个人建议不要对Version进行自动修改，可以作为参数传递进来。

2. 如果ipa包存放的文件夹为工作区，记得在.gitignore中进行忽略处理,我建议把fastlane文件也进行忽略，否则回退版本打包时缺失文件还需要手动打包。

3. 如果你的Apple ID在登录时进行了验证码验证，那么需要设置一个专业密码供fastlane上传使用，否则是上传不上去的。

4. 如果你们的应用截图和Metadata信息是运营人员负责编辑和维护的，那么在打包到AppStore时，记得要忽略截图和元数据，否则有可能因为不一致而导致覆盖。skip_metadata:true, #不上传元数据 skip_screenshots:true,#不上传屏幕截图

### 关于fastlane的一些想法
其实对于小团队来说，fastlane就可以简化很多操作，提升一些效率，但是还不够极致，后续可以通过结合Jenkins或是其他的CI的尝试来打通Git环节，测试环节，反馈环节等。


