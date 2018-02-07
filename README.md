本demo的ios和android版本都可以直接运行，只需要在reactNativeSplit目录下执行npm init安装好环境即可。

ios版本需要稍微修改一点源代码，在RCTBridge.h上增加一个loadModule接口。需要修改的三个文件RCTBridge.h RCTBridge.m RCTCxxBridge.mm放在了ios/reactNativeSplit文件夹内。
其中RCTCxxBridge.mm内的loadModule函数的onComplete接口在不同RN版本上有一点差异，如果报错，修复一下即可。

android版本因为可以使用反射，所以没有修改任何源代码，直接运行即可。

拆分bundle并分步加载的原理其实很简单，在ios和android平台都是一样的，源代码是把创建View和加载bundle耦合在一起，我们把先加载common.bundle并创建好js context，然后保存起来，
在android平台这个context的管理类是ReactInstanceManager, 在ios平台是RCTBridge。 然后当需要打开界面时，手动创建一个View，在android平台是ReactRootView,在ios平台是RCTRootView，同时
用保存起来的js context加载业务bundle即可，所以需要把加载bundle的接口从源代码里暴露出来以供我们调用，在ios平台是新建了一个loadModule方法，在android平台则是通过反射执行了CatalystInstanceImpl的loadScriptFromFile方法

这个demo只是实现了bundle的分步加载，demo跑起来后还有很多额外的事情需要做，例如bundle如何拆分，运行时可能出现的异常等等。但我们的拆分bundle项目已经在线上稳定运行了半年多时间，基本没什么问题了。

bundle的拆分方案是这样的：设定一个common.js，对它执行react-native bundle后生成common bundle，业务代码的入口例如index.android.js里最开始require的内容保持和common.js里一致。生成suba.bundle后，把里面common.bundle的内容去掉。
通过修改react-native/local-cli里的代码，可以给不同业务的moduleID加一个增量，比如suba.bundle就从10000起，subb.bundle从20000起，否则moduleId冲突的话，同时加载多个业务bundle就会出错了。

加载业务Bundle时，它们各自的资源查找也会出问题，需要手动修改一下源代码，在加载某个业务Bundle时，调整资源的查找路径。