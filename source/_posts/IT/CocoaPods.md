---
title: cocoapods
categories: IT
---

## 一、Cocoapods 的安装

Cocoapods 的安装非常简单，只需一行命令即可。

```
$ sudo gem install cocoapods
```

安装完后可在终端输入 pod ，会有如下输出：

<center>
![](http://dzliving.com/CocoaPods_0.png)
</center>

显示了 pod 的所有可用的命令和命令选项。


## 二、Cocoapods 的使用

打开终端，切换到你的工程目录，输入下面的命令

```
$ pod init
```

终端输入 `ls` 可以看到已经多了个 Podfile 文件，使用 vi 打开，内容如下

```
# Uncomment the next line to define a global platform for your project
# platform :ios, '9.0'

target 'SwiftDemo' do
  # Comment the next line if you're not using Swift and don't want to use dynamic frameworks
  use_frameworks!

  # Pods for SwiftDemo

  target 'SwiftDemoTests' do
    inherit! :search_paths
    # Pods for testing
  end

  target 'SwiftDemoUITests' do
    inherit! :search_paths
    # Pods for testing
  end

end
```

Podfile 的注释已经很清楚的告诉我们应该如何编写 Podfile 文件。Podfile 的具体编写规则请参考[官方文档](https://link.jianshu.com/?t=https://guides.cocoapods.org/syntax/podfile.html)。

编写好 Podfile 后执行以下命令：

```
$ pod install
```

终端会输出 `Setting up CocoaPods Master repo` 并会停顿很久。因为第一次使用 pod 会从 GitHub 拉取包含所有可用三方库信息的 [repo](https://link.jianshu.com/?t=https://github.com/cocoapods/Specs)，该代码仓库会随着可用三方库的增多逐渐变大，2014 年 2 月该 repo 仅有 129MB，2016 年 8 月该 repo 已经增加到 841 MB，2016 年 11 月该 repo 已达到 1.07GB（具体请参考[链接](https://link.jianshu.com/?t=http://stackoverflow.com/questions/21022638/pod-install-is-staying-on-setting-up-cocoapods-master-repo)）。后面会在使用小技巧中介绍如何避免在这一步"卡死"。


## 三、Cocoapods 的工作流程

pod install 执行流程可分为如下五个步骤

* 查看 ~/.cocoapods/repo/master/Specs 是否存在
* 存在，从这个本地三方库信息库中获取 Podfile 中对应三方库的 git 地址
* 不存在，输出 Setting up CocoaPods Master repo，并拉取[三方库信息库](https://link.jianshu.com/?t=https://github.com/cocoapods/Specs)到 ~/.cocoapods/repo/中
* 使用 git 命令从 GitHub 上拉取 Podfile 中对应的三方库源码

<center>
![](http://dzliving.com/CocoaPods_8.png)
</center>

在终端中输入如下命令

```
$ cd ~/.cocoapods/repos/master/Specs
$ open .
```

输入 `open .` 可以看到很多文件夹，其中就包含我们平常使用的三方库的文件夹。

随便选进入一个文件夹，这里找到 AFNetworking。

<center>
![](http://dzliving.com/CocoaPods_1.png)
</center>

当我们使用 `pod search AFNetworking` 命令时会将这里所有的 release 版本号输出出来。

<center>
![](http://dzliving.com/CocoaPods_3.png)
![](http://dzliving.com/CocoaPods_2.png)
</center>

cd 进最新版本的文件夹中，里面是一个 AFNetworking.podspec.json 文件，使用`编辑器/vi` 打开，内容如下

<center>
![](http://dzliving.com/CocoaPods_4.png)
</center>

可以看到这里包含了所有的三方库的相关信息，包括名字，协议，描述，Github 地址，支持平台等。

这些信息由三方库作者提交到该信息仓库中。比如你要发布自己的三方库到 Cocoapods 上，你需要 fork 该 [repo](https://link.jianshu.com/?t=https://github.com/cocoapods/Specs)，然后按照 Cocoapods 规定的格式把自己三方库的信息填写上，再 push 到主分支上。

不过这个过程 Cocoapods 也提供的相应的命令来完成，具体请参考[官方文档](https://link.jianshu.com/?t=https://guides.cocoapods.org/making/making-a-cocoapod.html)。

在查找到对应文件夹后，进行 json 数据解析，获得三方库的 repo 地址，调用本地 git 命令拉取源码，拉取完成后调用本地 xcodebuild 命令把三方库编译为 Framework。


## 四、源码分析

我们使用的 cocoapods 命令的 [GitHub 路径](https://github.com/CocoaPods/CocoaPods/tree/master/lib/cocoapods/command)。

<center>
![](http://dzliving.com/CocoaPods_5.png)
</center>

其中 setup 命令用于第一次使用 cocoapods 时对使用环境进行设置，下面对 setup 命令的源码进行分析。

> git 的详细使用教程请参考[文档](https://link.jianshu.com/?t=http://git-scm.com/book/zh/v2)

```
require 'fileutils'

module Pod
  class Command
    class Setup < Command
      self.summary = 'Setup the CocoaPods environment'

      self.description = <<-DESC
        Creates a directory at `~/.cocoapods/repos` which will hold your spec-repos.
        This is where it will create a clone of the public `master` spec-repo from:
            https://github.com/CocoaPods/Specs
        If the clone already exists, it will ensure that it is up-to-date.
      DESC

      extend Executable
      executable :git

      def run
        UI.section 'Setting up CocoaPods master repo' do
          if master_repo_dir.exist?
            set_master_repo_url
            set_master_repo_branch
            update_master_repo
          else
            add_master_repo
          end
        end

        UI.puts 'Setup completed'.green
      end

      #--------------------------------------#

      # @!group Setup steps

      # Sets the url of the master repo according to whether it is push.
      #
      # @return [void]
      #
      def set_master_repo_url
        Dir.chdir(master_repo_dir) do
          git('remote', 'set-url', 'origin', url)
        end
      end

      # Adds the master repo from the remote.
      #
      # @return [void]
      #
      def add_master_repo
        cmd = ['master', url, 'master']
        Repo::Add.parse(cmd).run
      end

      # Updates the master repo against the remote.
      #
      # @return [void]
      #
      def update_master_repo
        show_output = !config.silent?
        config.sources_manager.update('master', show_output)
      end

      # Sets the repo to the master branch.
      #
      # @note   This is not needed anymore as it was used for CocoaPods 0.6
      #         release candidates.
      #
      # @return [void]
      #
      def set_master_repo_branch
        Dir.chdir(master_repo_dir) do
          git %w(checkout master)
        end
      end

      #--------------------------------------#

      # @!group Private helpers

      # @return [String] the url to use according to whether push mode should
      #         be enabled.
      #
      def url
        self.class.read_only_url
      end

      # @return [String] the read only url of the master repo.
      #
      def self.read_only_url
        'https://github.com/CocoaPods/Specs.git'
      end

      # @return [Pathname] the directory of the master repo.
      #
      def master_repo_dir
        config.sources_manager.master_repo_dir
      end
    end
  end
end
```

其执行流程可分为如下步骤

1. 判断 ~/.cocoapods/repo目录是否存在
2. 存在，依次调用 set\_master\_repo\_url，set\_master\_repo\_branch，update\_master\_repo 三个函数分别设置 repo 主分支地址，git checkout 到主分支，拉取主分支代码，更新repo
3. 不存在，添加主分支 repo


## 五、使用 Cocoapods 的小技巧

大家在使用 pod install 命令时一般会加上 `--no-repo-update` 选项。这使得 pod install 不进行本地三方库信息库 git pull 的更新操作。若 Podfile 中指定 AFNetworking 的版本号为 4.2.0，但本地 `~/cocoapods/repo/master/Specs/.../AFNetworking` 中并没有此版本号，此时使用 `pod install --no-repo-update` 会出现如下错误

<center>
![](http://dzliving.com/CocoaPods_6.png)
</center>

若直接使用 pod install 便会先执行 pod repo update，由于 github 需要翻墙，并且更新的内容过大就会等待很久。

既然知道了 Cocoapods 的原理，我们便可以手动在 ~./cocoapods/repo/master/Specs 中添加我们需要的三方库版本信息，避免了把所有的并没有使用到的三方库信息更新到本地。

下面以添加 Alamofire 4.2.0 版本信息到本地为例子。

在终端输入如下命令

```
cd ~/.cocoapods/repo/master/Specs/Alamofire/
mkdir 4.2.0
cp ~/.cocoapods/repo/master/Specs/Alamofire/1.1.3/Alamofire.podspec.json ./4.2.0
```

上面的命令创建了名为 4.2.0 的文件夹，并复制了一个库信息的 json 文件到新创建的文件夹中，用 vi 打开该文件，内容如下

```
{
  "name": "Alamofire",
  "version": "1.1.3",
  "license": "MIT",
  "summary": "Elegant HTTP Networking in Swift",
  "homepage": "https://github.com/Alamofire/Alamofire",
  "social_media_url": "http://twitter.com/mattt",
  "authors": {
    "Mattt Thompson": "m@mattt.me"
  },
  "source": {
    "git": "https://github.com/Alamofire/Alamofire.git",
    "tag": "1.1.3"
  },
  "platforms": {
    "ios": "8.0"
  },
  "source_files": "Source/*.swift",
  "requires_arc": true
}
```

我们需要修改版本号，source 的 tag。那 source 的 tag 怎么知道是多少呢，这个可以从 GitHub 上找到。

<center>
![](http://dzliving.com/CocoaPods_7.png)
</center>

可以看到所有 release 版本信息。


## 文章

[zongmumask](https://www.jianshu.com/u/d8bbc4831623) - [Cocoapods 工作原理和源码分析](https://www.jianshu.com/p/c17cee5e9c7f)
[Include of non-modular header inside framework module]()