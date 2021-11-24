<!-- Add banner here -->

# 简介
使用 uniapp 框架开发需要自定义下拉刷新和自动加载？自带的 scrolltolower 事件有时候在意料之外触发？这个组件正在尝试高质量解决这些问题！该组件有以下特点：

- 使用 scroll-view 自定义下拉 + wxs 高性能操作 dom ，使用 IntersectionObserver 精准控制自动加载。
- 微信小程序、H5、APP体验基本一致。
- 除了 uniapp 运行时依赖，没有任何第三方依赖。
- 基于常用业务场景实现了一套开箱即用的规则，不喜欢的话支持自定义配置进行控制。

# 组件效果预览
你有两种种方式预览
- 使用浏览器访问该地址：https://dimple-uni-scroll.vercel.app 进行预览（pc端注意开启开发者模式调整到移动设备）。
- 这个项目本身就是一个 uniapp 项目，git clone该项目到本地后，可使用HBuiderX导入该项目进行预览。

# 目录

- [简介](#简介)
- [组件效果预览](#组件效果预览)
- [目录](#目录)
- [安装](#安装)
- [使用说明](#使用说明)
    - [参数](#参数)
    - [插槽](#插槽)
    - [事件](#事件)
- [开发说明](#开发说明)
- [贡献](#贡献)
    - [赞助](#赞助)
- [许可证](#许可证)
- [最后](#最后)

# 安装
[(Back to top)](#目录)

```
// 在命令行里执行
$ npm i @dimple-smile/uni-scroll

// 在你的代码里写
import DimpleUniScroll from '@dimple-smile/uni-scroll'
```

# 使用说明
[(Back to top)](#目录)

下面是一个简单的使用例子
> 在src/pages/index/index.vue文件里能看到预览使用的栗子
```
<template>
  <view style="height: 400px">
    <d-scroll :total="total" :skip="skip" @fetch="fetch">
      <view v-for="(item, index) in list" :key="index">{{ index }}</view>
    </d-scroll>
  </view>
</template>

<script>
import DScroll from '@dimple-smile/uni-scroll'
export default {
  components: { DScroll },
  data() {
    return {
      list: [],
      total: -1, // 默认值不能设置为0，初始值为0时无法判断是 一开始就无数据 还是 请求之后为无数据，当然可以用noData和noMore自行控制
      skip: -1, // 默认值不能设置为0，理由同上，当然可以用noData和noMore自行控制
    }
  },
  methods: {
    async fetch(e) {
      const { stop } = e
      await this.getData(e)
      stop()
    },
    async getData(options = {}) {
      const { page = 1, skip = 0, limit = 20 } = options
      const { total, data } = await new Promise((res) => setTimeout(() => res({ total: 100, data: Array(limit).fill('') }), 2000))
      this.total = total
      if (page === 1) this.list = []
      this.list.push(...data)
      this.skip = this.list.length
    },
  },
  async mounted() {
    uni.showLoading({ title: '加载中' })
    await this.getData()
    uni.hideLoading()
  },
}
</script>
```
### 参数
[(Back to top)](#目录)
| 参数名 | <img width="180px" /> 意义 <img width="180px" /> | 类型 | <img width="120px" /> 默认值 <img width="120px" /> | 说明 |
| -| -| -| - | - |
| height | 滚动容器的高度  | String | 100%| 默认100%，代表使用该组件的父元素必须具有高度。ps: 高度继承page的100%时，page需要设置height: 100%，page默认无高度。
| background | 滚动容器的背景色 | String | #eeeeee | 这个背景色会和下拉/上拉时的背景色一致。 |
| threshold | 距离顶部多少距离触发释放方法 | Number | 80 | 单位为px。
| limit | 请求数据一页的长度 | Number | 20 | 该参数偏业务，如不使用可以不设置，结果是无限加载，不会出现到底了。 |
| skip | 请求数据时跳过多少已加载的条数 | Number | -1 | 比如列表已经加载了20条数据，那么skip应为20。该参数偏业务，如不使用可以不设置，结果是无限加载，不会出现到底了。 |
| total | 一共有多少条数据 | Number | -1 | 默认值不能设置为0，应在请求之后设置该值。该参数偏业务，如不使用可以不设置，结果是无限加载，不会出现到底了。 |
| refresherText | 下拉过程显示的文本 | String | 下拉刷新 | 无 |
| refresherActivedText | 下拉可刷新时显示的文本 | String | 松开刷新 | 无 |
| loadingText | 加载中显示的文本 | String | 加载中... | 无 |
| noDataText | 无数据时显示的文本 | String | 暂无数据 | 无 |
| noMoreText | 无更多数据时显示的文本 | String | 没有更多数据了| 无 |
| loaderSize | loader图标大小 | Number | 25 | 无 |
| disabled | 是否禁用下拉和自动加载 | Boolean | false | 无 |
| noData | 是否为无数据状态 | Boolean,String | '' | 只有设置为布尔值才会生效 |
| noMore | 是否为没有更多数据状态 | Boolean,String | '' | 只有设置为布尔值才会生效 |

注意：limit、skip、total三个参数配合可以得出，是否没有数据 以及 是否已经加载完数据。注意都要在请求之后再改变skip和total，默认值不要设置为0。当然可以用noData和noMore自行控制。

### 插槽
[(Back to top)](#目录)

| 参数名 | <img width="200px" /> 意义 <img width="200px" /> | 说明 |
| - | - | - |
| loadmorer | 自定义上拉加载 | 通过v-slot:loadmorer="{ threshold, loading}"获取自定义需要的数据。threshold：上拉拉触发事件的阈值；loading：是否处于加载状态。|
| noData | 无数据时的内容 | 需要设置skip和total来配合判断。 |
| noMore | 到底时的内容 | 需要设置skip和total来配合判断。 |
| 默认 | 用来放置内容 | 无

### 事件
[(Back to top)](#目录)
| 参数名 | <img width="200px" /> 意义 <img width="200px" /> | 说明 |
| - | - | - |
| fetch | 下拉/上拉超过阈值之后触发的方法 | 使用者需要把更新列表的方法写在这个回调里，回调得到值是一个对象，对象里是一些可用参数。skip：见参数说明；limit：见参数说明；total：见参数说明；page：当前加载到的页码；loadmore：是否是上拉加载；stop：一个function，必须要手动调用stop()才会停止刷新/加载动作！所以要求对请求进行防呆处理，保证请求失败也会调用stop()。 |
| scroll | scroll-view组件的滚动事件 | 按需使用。 |

注意：fetch是必须要handle的，handle后需要手动调用回调得到参数对象里的stop方法才能停止下拉/上拉动作。
# 开发说明
[(Back to top)](#目录)

src文件夹存放着组件的全部源码。入口为src/components/dimple-uni-scroll.vue。

# 贡献
[(Back to top)](#目录)

@dimple-smile

### 赞助
[(Back to top)](#目录)

Love
# 许可证
[(Back to top)](#目录)

只要不商用，注明出处即可。

[GNU General Public License version 3](https://opensource.org/licenses/GPL-3.0)

# 最后
[(Back to top)](#目录)

谢谢你的使用~
