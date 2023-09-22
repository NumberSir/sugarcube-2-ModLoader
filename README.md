
# ModLoader

---

* [ModLoader](#ModLoader)
  * [简介](#简介)
  * [如何制作 Mod.zip 文件](#如何制作-modzip-文件)
      * [注意事项](#注意事项)
  * [如何打包Mod](#如何打包mod)
    * [手动打包方法](#手动打包方法) <========= **最简单的打包Mod方法**， 若没有NodeJs环境，请使用此方法
    * [自动打包方法](#自动打包方法)
  * [ModLoader开发及修改方法](#ModLoader开发及修改方法)

---

# 简介

此项目是为 SugarCube-2 引擎编写的Mod加载器，初衷是为 [Degrees-of-Lewdity] 设计一个支持Mod加载和管理的Mod框架，支持加载本地Mod、远程Mod、旁加载Mod（从IndexDB中加载）。

本项目的目的是为了方便制作Mod以及加载Mod，同时也为了方便制作Mod的开发者，提供了一些API来方便读取和修改游戏数据。

本项目由于需要在SC2引擎启动前注入Mod，故对SC2引擎做了部分修改，添加了ModLoader的引导点，以便在引擎启动前完成Mod的各项注入工作。

修改过的 SC2 在此处：[sugarcube-2](https://github.com/Lyoko-Jeremie/sugarcube-2_Vrelnir) ，使用此ModLoader的游戏需要使用此版本的SC2引擎才能引导本ModLoader。

具体有关如何打包一个带有ModLoader的游戏本体的方法，详见 [ModLoader开发及修改方法](#ModLoader开发及修改方法) 。

---

# 如何制作 Mod.zip 文件：


给自己的mod命名

以mod名字组织自己的mod

编写mod的引导描述文件 boot.json 文件

格式如下（样例 src/insertTools/MyMod/boot.json）：

```json5
{
  "name": "MyMod",    // （必须存在） mod名字
  "version": "1.0.0", // （必须存在） mod版本
  "scriptFileList_inject_early": [  // （可选） 提前注入的 js 脚本 ， 会在当前mod加载后立即插入到dom中由浏览器按照script的标注执行方式执行
    "MyMod_script_inject_early_example.js"
  ],
  "scriptFileList_earlyload": [     // （可选） 提前加载的 js 脚本 ， 会在当前mod加载后，inject_early脚本全部插入完成后，由modloader执行并等待异步指令返回，可以在这里读取到未修改的Passage的内容
    "MyMod_script_earlyload_example.js"
  ],
  "scriptFileList_preload": [     // （可选） 预加载的 js 脚本文件 ， 会在引擎初始化前、mod的数据文件全部加载并合并到html的tw-storydata中后，由modloader执行并等待异步指令返回， 可以在此处调用modloader的API读取最新的Passage数据并动态修改覆盖Passage的内容
    "MyMod_script_preload_example.js"     // 注意 scriptFileList_preload 文件有固定的格式，参见样例 src/insertTools/MyMod/MyMod_script_preload_example.js
  ],
  "styleFileList": [      // （必须存在） css 样式文件
    "MyMod_style_1.css",
    "MyMod_style_2.css"
  ],
  "scriptFileList": [     // （必须存在） js 脚本文件，这是游戏的一部分
    "MyMod_script_1.js",
    "MyMod_script_2.js"
  ],
  "tweeFileList": [       // （必须存在） twee 剧本文件
    "MyMod_Passage1.twee",
    "MyMod_Passage2.twee"
  ],
  "imgFileList": [        // （必须存在） 图片文件，尽可能不要用容易与文件中其他字符串混淆的文件路径，否则会意外破坏文件内容
    "MyMod_Image/typeAImage/111.jpg",
    "MyMod_Image/typeAImage/222.png",
    "MyMod_Image/typeAImage/333.gif",
    "MyMod_Image/typeBImage/111.jpg",
    "MyMod_Image/typeBImage/222.png",
    "MyMod_Image/typeBImage/333.gif"
  ],
  "additionFile": [     // （必须存在） 附加文件列表，额外打包到zip中的文件，此列表中的文件不会被加载，仅作为附加文件存在
    "readme.txt"      // 第一个以readme(不区分大小写)开头的文件会被作为mod的说明文件，会在mod管理器中显示
  ]
}

```

最小 boot.json 文件样例：

```json
{
  "name": "EmptyMod",
  "version": "1.0.0",
  "styleFileList": [
  ],
  "scriptFileList": [
  ],
  "tweeFileList": [
  ],
  "imgFileList": [
  ],
  "additionFile": [
    "readme.txt"
  ]
}
```

### 注意事项

1. boot.json 文件内的路径都是相对路径，相对于zip文件根目录的路径。
2. 图片文件的路径是相对于zip文件根目录的路径。
3. 同一个mod内的文件名不能重复，也尽量不要和原游戏或其他mod重复。与原游戏重复的部分会覆盖游戏源文件。
4. 具体的来说，mod会按照mod列表中的顺序加载，靠后的mod会覆盖靠前的mod的passage同名文件，mod之间的同名css/js文件会直接将内容concat到一起，故不会覆盖css/js等同名文件。
5. 加载时首先计算mod之间的覆盖，（互相覆盖同名passage段落，将同名js/css连接在一起），然后将结果覆盖到原游戏中（覆盖原版游戏的同名passage/js/css）
6. 当前版本的mod加载器的工作方式是直接将css/js/twee文件按照原版sc2的格式插入到html文件中。


---


对于一个想要修改passage的mod，有这么4个可以修改的地方
1. scriptFileList_inject_early ， 这个会在当前mod读取之后，“立即”插入到script脚本由浏览器按照script标签的标准执行，这里可以调用ModLoader的API，可以读取未经修改的SC2 data （包括原始的passage）
2. scriptFileList_earlyload  ，这个会在当前mod读取之后，inject_early 脚本插入完之后，由modloader执行并等待异步指令返回，这里可以调用ModLoader的API，可以执行异步操作，干一些远程加载之类的活，也可以在这里读取未经修改的SC2 data（包括原始的passage）
3. tweeFileList ，这个是mod的主体，会在modloader读取所有mod之后，做【1 合并所有mod追加的数据，2 将合并结果覆盖到原始游戏】的过程应用修改到原始游戏SC2 data上
4. scriptFileList_preload ， 这个会在mod文件全部应用到SC2 data之后由modloader执行并等待异步操作返回，这里可以像earlyload一样做异步工作，也可以读取到mod应用之后的SC2 data

上面的步骤结束之后SC2引擎才会开始启动，读取SC2 data，然后开始游戏，整个步骤都是在加载屏幕（那个转圈圈）完成的。

---

另，由于SC2引擎本身会触发以下的一些事件，故可以使用jQuery监听这些事件来监测游戏的变化
```
// 游戏完全启动完毕
:storyready
// 一个新的 passage 上下文开始初始化
:passageinit
// 一个新的 passage 开始渲染
:passagestart
// 一个新的 passage 渲染结束
:passagerender
// 一个新的 passage 准备插入到HTML
:passagedisplay
// 一个新的 passage 已经处理结束
:passageend
```

由于，可以以下方法监听jQuery事件
```js
$(document).one(":storyready", () => {
   // ....... 触发一次
});
$(document).on(":storyready", () => {
   // ....... 触发多次
});
```
---

### 变更：

【2023-09-21】 删除 `imgFileReplaceList` ，现在使用新的ImageHookLoader直接拦截图像请求来实现图像替换，因此，与原始图像文件重名的图像会被覆盖

---

## 如何打包Mod

### 手动打包方法

这个方法只需要有一个可以压缩Zip压缩包的压缩工具，和细心的心。

1. 使用你喜爱的编辑器，编辑好boot.json文件

2. 在boot.json文件根目录使用压缩工具（例如如下例子中使用的 [7-Zip](7-zip.org/)，其他软件方法类似），**仔细选择 boot.json 以及在其中引用的文件打包成zip文件**

![](https://raw.githubusercontent.com/wiki/Lyoko-Jeremie/sugarcube-2-ModLoader/fast/step1.png)

3. 设置压缩参数（**格式Zip，算法Deflate，压缩等级越大越好，没有密码**）

![](https://raw.githubusercontent.com/wiki/Lyoko-Jeremie/sugarcube-2-ModLoader/fast/step2.png)

4. 点击确定，等待压缩完成

5. 压缩后请打开压缩文件再次检查：（**boot.json 文件在根目录**，boot.json中编写的文件路径和压缩完毕之后的结构**一模一样**，任何一个文件的压缩算法只能是**Store或Deflate**）

![](https://raw.githubusercontent.com/wiki/Lyoko-Jeremie/sugarcube-2-ModLoader/fast/step3.png)


6. 重命名压缩包为 mod名字.mod.zip  （这一步**可选**）

7. 使用Mod管理器加载Mod

### 自动打包方法

这个方法需要有NodeJs和Yarn使用知识

打包：

编译脚本

```shell
yarn run webpack:insertTools:w
```

切换到 Mod 所在文件夹，（即boot.json所在文件夹）
```shell
cd src/insertTools/MyMod
```

执行

```shell
node "<packModZip.js 文件路径>" "<boot.json 文件路径>"
```

例如：

```shell
node "H:\Code\sugarcube-2\ModLoader\dist-insertTools\packModZip.js" "boot.json"
```

之后会在当前目录下打包生成一个以boot.js文件中的mod名命名的zip文件，例如：

```
MyMod.mod.zip
```


---

# ModLoader开发及修改方法

**以下是关于如何修改ModLoader本体、如何将ModLoader本体以及插入到游戏中、如何将预装Mod嵌入到Html中的方法。若仅仅制作Mod，仅需按照以上方法[打包Mod](#如何打包mod)即可在Mod管理界面加载zip文件。**

---

本项目由于需要在SC2引擎启动前注入Mod，故对SC2引擎做了部分修改，添加了ModLoader的引导点，以便在引擎启动前完成Mod的各项注入工作。

修改过的 SC2 在此处：[sugarcube-2](https://github.com/Lyoko-Jeremie/sugarcube-2_Vrelnir) ，使用此ModLoader的游戏需要使用此版本的SC2引擎才能引导本ModLoader。

可在SC2游戏引擎项目中执行 `build.js -d -u -b 2` 来编译SC2游戏引擎，编译结果在SC2游戏引擎项目的 `build/twine2/sugarcube-2/format.js` ，
将其覆盖 [Degrees-of-Lewdity] 游戏的原版 `devTools/tweego/storyFormats/sugarcube-2/format.js` ，编译原版游戏本体获得带有ModLoader引导点的Html游戏文件，
随后按照下方的方法编译并注入此ModLoader到Html游戏文件，即可使用此ModLoader。

---

编译脚本

```shell
yarn run webpack:BeforeSC2:w
yarn run ts:ForSC2:w
yarn run webpack:insertTools:w
```

如何插入Mod加载器以及将预装Mod内嵌到html文件：

编写 modList.json 文件，格式如下：
（样本可参见 src/insertTools/modList.json ）
```json
[
  'mod1.zip',
  'mod2.zip'
]
```


切换到 modList.json 所在文件夹

```shell
cd ./src/insertTools/modList.json
```

```shell
node "<insert2html.js 文件路径>" "<Degrees of Lewdity VERSION.html 文件路径>" "<modList.json 文件>" "<BeforeSC2.js 文件路径>"
```

例如：

```shell
node "H:\Code\sugarcube-2\ModLoader\dist-insertTools\insert2html.js" "H:\Code\degrees-of-lewdity\Degrees of Lewdity VERSION.html" "modList.json" "H:\Code\sugarcube-2\ModLoader\dist-BeforeSC2\BeforeSC2.js"
```

会在原始html文件同目录下生成一个同名的html.mod.html文件，例如：
```
Degrees of Lewdity VERSION.html.mod.html
```
打开`Degrees of Lewdity VERSION.html.mod.html`文件， play


---

## 附NodeJs及Yarn环境安装方法

1. 从 [NodeJs 官网](https://nodejs.org) 下载NodeJs并安装，例如 [node-v18.18.0-x64.msi](https://nodejs.org/dist/v18.18.0/node-v18.18.0-x64.msi)
2. 在命令行运行 `corepack enable` 来启用包管理器支持
3. 结束

