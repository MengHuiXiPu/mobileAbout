# 复杂场景下唤起App实践

#### 1. 什么是唤端？

一张图了解一下什么是唤端。

![图片](https://mmbiz.qpic.cn/mmbiz_gif/T81bAV0NNN9ichaf86RC3evOQT92aMj6ic4CLTqxKz8tJFkTS6kfxtg2p1RCnWk65Xn005KUAZKibiaIMZKPsPqPtw/640?wx_fmt=gif&tp=webp&wxfrom=5&wx_lazy=1)

#### 2. 唤端功能

唤端功能架构图如下。

![图片](data:image/gif;base64,iVBORw0KGgoAAAANSUhEUgAAAAEAAAABCAYAAAAfFcSJAAAADUlEQVQImWNgYGBgAAAABQABh6FO1AAAAABJRU5ErkJggg==)功能

#### 3. 唤端技术

唤端技术架构图如下。

![图片](data:image/gif;base64,iVBORw0KGgoAAAANSUhEUgAAAAEAAAABCAYAAAAfFcSJAAAADUlEQVQImWNgYGBgAAAABQABh6FO1AAAAABJRU5ErkJggg==)

------

## 二 面临的新问题

虽然，当前方案已经支持了基本的唤端能力，可以说是唤端技术在转转的从 0 到 1。
然而，随着时间的迁移，支持兼容的业务逻辑越来越多，项目内部结构已混乱不堪、维护艰难，并且满足不了新的业务诉求，因此，决定对其进行一次重大重构。

经过业务反馈和调研，主要问题如下：

1. 集团内App尚未完全覆盖，平台兼容性尚待完善。
2. 未与行业方案对齐，满足不了业务提出的更高要求。
3. 周边生态有待完善，业务使用体验有待提高。

对以上问题进行了梳理、评估，最终定下了本次重构目标：

1. 整体架构升级，支持唤起多App，提升唤起兼容性。
2. 对标业界方案，完善现有能力。
3. 唤起App周边生态完善，提升业务体验。

## 三 项目重构

### 整体架构

#### 1. 旧的方案与架构

首先，看一下原来的目录结构(其实一个好的基础库项目只看目录结构就能大概推测出架构的轮廓)。

```
|-src
  |-- callers
    |-- 58App.js
    |-- browser.js
    |-- qq.js
    |-- wechat.js
    |-- zzLike.js
  |-- core
    |-- base.js
    |-- index.js
  |-- libs
    |-- config.js
    |-- platform.js
    |-- utils.js
 index.js
```

架构图如下：

![图片](data:image/gif;base64,iVBORw0KGgoAAAANSUhEUgAAAAEAAAABCAYAAAAfFcSJAAAADUlEQVQImWNgYGBgAAAABQABh6FO1AAAAABJRU5ErkJggg==)旧架构

这种以运行时平台基础的策略模式+适配器模式的设计架构，随着业务复杂度的提高，主要存在以下弊端：

- 运行平台多变莫测，如果新增运行时环境则必须新增一个 caller 类支持，不符合开闭原则。
- 在进行业务扩展和处理交叉逻辑时，代码量会激增，严重影响基础库代码体积。
- 功能职责划分不清晰不明确，每一个 caller 类，都需要单独进行处理，项目整体维护成本较大并会增加心智负担。

在支持有限个运行时平台，唤起单一App时，上述架构设计完全满足需求，上述问题也不会凸显，但是随着业务复杂度和要求越来越高，缺点会越来越突出。

#### 2. 业界方案

对社区的一些唤端方案和建设进行了一些调研并进行了对比。主要内容如下：

| 项目                   | 特点                                       |                     缺点                     | star |
| :--------------------- | :----------------------------------------- | :------------------------------------------: | :--: |
| 开源-web-launch-app[1] | 功能职责划分明确，兼容平台多               | 参数设计较复杂，内部封装不够简练，不支持回调 | 678  |
| 开源-callapp-lib[2]    | 功能职责划分明确，代码组织友好，兼容平台多 |        只提供浏览器端唤起单一App方案         | 1.8K |
| 文章-唤端背后的技术[3] | 介绍了一些前沿方案                         |            缺少细节信息量，不开源            |  -   |

可以看出，社区开源的方案在设计各有差异，但是可以看出都采用了单一职责原则和功能模块化的设计思想。在架构设计上第二个更清晰更简洁，相对较有参考价值，所以我们选择基于此进行二次开发。

其实，在 JS 架构设计中这种思想在各大开源框架都在使用，比如社区的 Vue 框架，其源码内部的功能划分、职责分离和模块化做的很完美。因此，这种编程思想（架构思想）很值得去借鉴。

#### 3. 升级后的转转方案

转转的业务场景相对于业界方案较复杂，需要一行代码支持集团所有App唤端。

这种用户极其方便和无感的用法，意味着所有的业务逻辑和配置都需要在基础库内部进行处理，复杂度也随着上升，对架构的设计也有较高的要求。

```
// 转转唤端基础库的使用方法，一行代码，支持集团所有app
// path 为内部统跳平台生成的统跳协议地址
new CallApp().start({path: ''})
```

结合了自身的业务场景的特点，和业界的优秀方案，最终沉淀出以下架构，如图所示。

![图片](data:image/gif;base64,iVBORw0KGgoAAAANSUhEUgAAAAEAAAABCAYAAAAfFcSJAAAADUlEQVQImWNgYGBgAAAABQABh6FO1AAAAABJRU5ErkJggg==)

设计的原则和思想是，以功能职责划分，抽象出各个功能模块，然后再将各个模块有机的组成一个整体。

> ❝
>
> 把一个整体（完成人类生存的所有工作）切分成不同的部分（分工），由不同角色来完成这些分工，并通过建立不同部分相互沟通的机制，使得这些部分能够有机的结合为一个整体。
>
> ❞

### 重构中的技术思考

##### 1. 运行环境判断策略优化

在对运行环境进行判断时，需要处理巨大型 if else。优化的思路(手段)：合理的数据结构 + 策略模式思想。

先来看看优化前的样子。

```
if(isIos) {
  if (isWechat && isLow7WX) {
    //...
  } else if (isLow9Ios) {
    // ...
  } else if (!supportUniversal && isBaidu) {
    // ...
  } else if (!supportUniversal && (isWeibo || isWechat)) {
    // ...
  } else if (!supportUniversal || isQQ || isQQBrowser || isQzone) {
    // ...
  } else if (isQuark) {
    // ...
  } else {
    // ...
  }
}

if(isAndroid) {
  // 省略...
}
```

一般情况下，在 JS 中最普通的策略模式实现代码实现如下：

```
// 定义策略
const strategy = {
  'case1': handler1, 
  'case1': handler1 
};
// 匹配策略并运行
if(case) {
  strategy[case]();
}
```

然而，这种用对象 Object 定义的策略模式实现，并不能满足该场景的需求，原因如下：

- key 不能为引用类型。
- Object key无序性。

提到 `有序，key 为引用`，脑子里第一个想到的肯定是 Map，采用 Map 是可行的。但是在该场景中也存在以下问题：

- key 为函数，需要进行多次 set 操作。
- 需要按顺序遍历执行后才能得知 key 是否命中，不是 O(1)，发挥不出 Map 的优势。

```
const map = new Map()
map.set(fn1, handler1)
map.set(fn2, handler2)
// ....
// map.set(fnN, handlerN)
// 业务流程中进行使用（算法），最坏O(n)
for (let [key, val] of map) {
  if(key()) return val()
}
```

最后，综合了业务使用场景考虑，采用数组 + 对象的数据结构，既能保证执行顺序，又可提供判断条件为函数形式，并且还能赋予更多属性。

```
// 定义一组有序的运行时环境策略 以单一职责和开闭原则为基准
// 默认策略
export let tempIosPlatRegList = null;

export const getDefaultIosPlatRegList = (ctx) => [
  {
    name: 'wxSub',
    platReg: () => (isWechat && isLow7WX),
    handler: (instance) => { //... }
  },
  {
    name: 'low9',
    platReg: () => isLow9Ios,
    handler: (instance) => { // ... }
  },
  {
    name: 'bd',
    platReg: () => !ctx.supportUniversal && isBaidu,
    handler: (instance) => { // ... }
  }
  // ...
]
// 对外提供  获取方法
export const getIosPlatRegList = (ctx) => 
  tempIosPlatRegList || (tempIosPlatRegList = getDefaultIosPlatRegList(ctx))

// 对外提供  扩展方法 
export const addIosPlatReg(ctx, item) {
  if(validPlatRegItem(item)) {
    const list = getDefaultIosPlatRegList(ctx)
    list.splice(-1, 0, item)
    tempIosPlatRegList = [...list]
  }
  return tempIosPlatRegList
}
// 在唤端业务流程中使用  
// 匹配运行时平台 并运行匹配到的功能函数
if(isIos) {
  for(let item of iosPlatRegList) {
    try {
      if(item && item.regPlat()) {
        item.handler(ctx)
        break;
      }
    } catch (error) {
      logError(item, error)
    }
  }
}
if(isAndroid) {
  for(let item of androidPlatRegList) {
    try {
      if(item && item.regPlat()) {
        item.handler(ctx)
        break;
      }
    } catch (error) {
      logError(item, error)
    }
  }
}
```

是不是代码看着代码清晰简洁了很多，并且对外提供了扩展能力，方便使用者对其进行扩展，这也很好的遵循了开闭原则。

> ❝
>
> 感悟：其实，JS中策略模式实现并不一定非得采用用对象 Object 或 class 的数据结构去实现，只要定义好一组 case 和 handler， 然后让算法按照指定的 case 去运行对应的逻辑即可，与具体哪种方式无关。参照策略模式的设计准则：定义一族算法（业务规则）；封装每个算法；这族算法可互换代替。
>
> ❞

#### 2. 引入 hooks

AOP 思想应该大家都有听说，在前端框架中常常会用到比如：钩子函数或者生命周期函数，其实也是AOP思想的一种实践。

引入了钩子函数可以很方便让使用者在相应的位置插入一些自定义逻辑。比如埋点，检查或魔改配置等等。

PS: 在唤端处理流程中，由于没有 mixin 之类的用法，没必要兼容 hooks 队列的处理方式，所以并没有引入发布订阅（、emit）的方式进行hooks实现，只采用最简单函数调用方式即可满足。

```
// 引入 hooks ,方便逻辑插入，埋点等
callApp.start({
  path: '',
  // 开始唤起钩子, 暴露出来配置方便进行检查或魔改
  callStart(opts) {},
  // 唤起成功钩子
  callSuccess() {},
  // 唤起失败钩子
  callFailed() {},
  // 开始下载钩子
  callDownload() {}
})
```

#### 3. 利用位操作进行优化

众所周知，在计算机的世界中其有 0 和 1，在对高级语言的处理时，对位运算的处理效率也是最高的，并且在多重判断时候还可以结合二进制 与/或/非 运算特性来简化操作。

在判断 currentApp 与 targetApp 时，引入了位运算符，部分代码如下：

```
// AppFlags 标记
export const enum AppFlags {
  ZZ = 1,
  ZZSeller = 1 << 1,
  ZZHunter = 1 << 2,
  ZZSeeker = 1 << 3,
  WXMini = 1 << 4,
  NoZZ = (1 << 1) | (1 << 2) | (1 << 3) | (1 << 4),
}

// targetAppFlag 为唤起目标app flag
if (targetAppFlag & AppFlags.ZZ) {
  // ...
} else if(targetAppFlag & AppFlags.NoZZ) {
  // ...
} else {}

// 引入 flag 之前的写法
- if (targetApp === 'zz') {
-  // 
- } else if(
-  targetApp === 'zzSeeKer' || 
-  targetApp === 'zzHunter' || 
-  targetApp === 'zzSeller' || 
-  targetApp === 'wxMini'
- ) {
-  // 
- } else {}
```

## 四 更多实践

### 下载后还原

还原方案主要分为两种：

#### 1.存储到剪切板 

在转转IOS端进行初次下载还原活动页面功能，主要采用的是复制到剪切板的方案。

唤端失败会触发失败 hook 或者点击下载触发下载 hook 时，会把约定好的协议内容copy 到系统剪切板， 下载完成后 App 启动会自动读取剪切板内容，如果内容的协议格式匹配成功，则跳转到指定页面进而实现页面还原。

在前端的工作主要就是 把协议内容复制到剪切板即可。代码实现如下：

```
const copyToClipboard = str => {
  // 创建 textarea 元素
  const el = document.createElement('textarea');
  el.value = str;
  el.setAttribute('readonly', '');
  el.style.position = 'absolute';
  el.style.left = '-9999px';
  // 添加到页面
  document.body.appendChild(el);
  // 获取原有 selected range
  const selected =
    document.getSelection().rangeCount > 0
      ? document.getSelection().getRangeAt(0)
      : false;
  // 选中并复制
  el.select();
  document.execCommand('copy');
  // 移除 textarea
  document.body.removeChild(el);
  // 还原 selected range
  if (selected) {
    document.getSelection().removeAllRanges();
    document.getSelection().addRange(selected);
  }
};
```

![图片](data:image/gif;base64,iVBORw0KGgoAAAANSUhEUgAAAAEAAAABCAYAAAAfFcSJAAAADUlEQVQImWNgYGBgAAAABQABh6FO1AAAAABJRU5ErkJggg==)页面还原流程

页面还原效果如下：

![图片](data:image/gif;base64,iVBORw0KGgoAAAANSUhEUgAAAAEAAAABCAYAAAAfFcSJAAAADUlEQVQImWNgYGBgAAAABQABh6FO1AAAAABJRU5ErkJggg==)



#### 2. 上传到服务端

在安卓侧是通过绑定 deeplinkid 来进行实现，首先讲活动页地址在服务端生成一个唯一的 depplinkid，然后在触发下载时，请求指定的链接（url要带上deeplinkid），服务端会返回带有deeplinkid信息的下载包链接，触发进行下载安装即可还原活动页。

```
// 拼接 deeplinkid 到下载 api url 即可
const zzDownloadLink = `${downloadApi}?applinkId=${}&channelId=${}`

// 触发下载
evokeDownload(zzDownloadLink)
```

整体流程如下图：

![图片](data:image/gif;base64,iVBORw0KGgoAAAANSUhEUgAAAAEAAAABCAYAAAAfFcSJAAAADUlEQVQImWNgYGBgAAAABQABh6FO1AAAAABJRU5ErkJggg==)还原流程

### 短信短链接唤起

如果用户可以在短信中直接唤起 App（毕竟短信也是普遍流行的通信工具之一） 那将是一件很美好的事情，唤端体验丝滑。

由于有 IOS 的 universal link 协议和 Android 的 app links 协议的支持（基于 http scheme），让投放的短信短链接唤起 App 也成为了可能。

经过调研和实践发现实现起来并不难，只需要把短链接服务的域名支持上相对应的协议并在客户端稍作配置即可。

下面是 IOS 短信短链接唤起 App 的实现流程。

![图片](data:image/gif;base64,iVBORw0KGgoAAAANSUhEUgAAAAEAAAABCAYAAAAfFcSJAAAADUlEQVQImWNgYGBgAAAABQABh6FO1AAAAABJRU5ErkJggg==)短链接唤起流程

短链接唤起效果如下：

![图片](data:image/gif;base64,iVBORw0KGgoAAAANSUhEUgAAAAEAAAABCAYAAAAfFcSJAAAADUlEQVQImWNgYGBgAAAABQABh6FO1AAAAABJRU5ErkJggg==)



### 唤端数据统计 

为了方便唤端数据的统计（唤端 PV 以及唤端成功率/失败率/下载率等）， 可以结合 hooks 和 乐高埋点（转转内部埋点系统）进行唤端数据上报和后续分析。

hooks + lego埋点代码如下：

```
import lego from './lego'

new CallApp({
   path: '',
   callStart() {
    lego.send({
      actiontype: 'START',
      pagetype: 'ZZDOWNLOADH5',
      backup: { channelId },
    })
  },
  callSuccess() {
    lego.send({
      actiontype: 'SUCCESS',
      pagetype: 'ZZDOWNLOADH5',
      backup: { channelId },
    })
  },
  callFailed() {
    lego.send({
      actiontype: 'FAILED',
      pagetype: 'ZZDOWNLOADH5',
      backup: { channelId },
    })
  },
  callDownload() {
    lego.send({
      actiontype: 'DOWNLOAD',
      pagetype: 'ZZDOWNLOADH5',
      backup: { channelId },
    })
  },
})
```

通过埋点数据我们可以做进一步的分析统计，比如唤端次数，系统信息 UA 等等，进而可针对性的对系统进行完善。

下面举个例子：

下图是在lego平台查看的某时段的埋点数据：

| 唤端触发 | 唤端成功 | 唤端失败 |
| :------- | :------- | :------- |
| 678      | 359      | 238      |

通过数据发现一个问题，数据中`触发 > 成功 + 失败`, 那么说明什么原因造成的呢？可以推测唤端触发时按钮没有加防抖，那么就可以加以对其进行修复完善，这样就可以形成一个数据驱动反馈的闭环。

![图片](data:image/gif;base64,iVBORw0KGgoAAAANSUhEUgAAAAEAAAABCAYAAAAfFcSJAAAADUlEQVQImWNgYGBgAAAABQABh6FO1AAAAABJRU5ErkJggg==)

## 五 周边生态建设

### 工具平台

前端工具平台可一键生成唤起App长链接、短链接，短信短链接等，便于业务使用。

![图片](data:image/gif;base64,iVBORw0KGgoAAAANSUhEUgAAAAEAAAABCAYAAAAfFcSJAAAADUlEQVQImWNgYGBgAAAABQABh6FO1AAAAABJRU5ErkJggg==)工具平台

### 统跳平台

统跳平台便于多个App跳转目标URL进行统一管理。URL可以是一个页面地址，也可以是一个行为的描述，比如弹窗。![图片](data:image/gif;base64,iVBORw0KGgoAAAANSUhEUgAAAAEAAAABCAYAAAAfFcSJAAAADUlEQVQImWNgYGBgAAAABQABh6FO1AAAAABJRU5ErkJggg==)

### 中间落地页

统一的中间页，让使用者可以接入URL页面的形式来接入唤起功能，而无需修改项目代码引入唤起库，唤端中间页支持多App。![图片](data:image/gif;base64,iVBORw0KGgoAAAANSUhEUgAAAAEAAAABCAYAAAAfFcSJAAAADUlEQVQImWNgYGBgAAAABQABh6FO1AAAAABJRU5ErkJggg==)

## 六 开源建设

项目已经在 Github 开源，欢迎吐槽。

Github 地址：**https://github.com/zhuanzhuanfe/call-app**

## 七 总结与展望

经过大量的实践，在唤端方面积累沉淀了一些经验，但是还有很长的路要走。

比如现在的唤端成功率只是一个很粗略的统计。如果用户触发了 universal link 唤端，唤起失败就会跳转到 universal link 绑定的域名，这种成功/失败是统计不到的（页面进行了跳转，页面栈消失），需要结合客户端侧共同做一套全链路的上报系统。

另外，现在端内分享出去的页面，和唤端回流形不成闭环，因此这种分享链路以及用户之间的隐藏关系就无法统计，无法更好的进行算法推荐。

------

