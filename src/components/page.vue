<template>
  <view class="dimple-uni-scroll" :style="{ height: height, overflow: freshing || loading ? 'hidden' : 'auto' }" @touchstart="touchstart" @touchmove="touchmove" @touchend="touchend">
    <view class="dimple-uni-scroll-refresher-flag"></view>
    <slot name="refresher" :dy="-dy" :threshold="threshold" :loading="freshing">
      <view class="dimple-uni-scroll-refresher" :class="{ animate: dy === 0 || -dy === threshold }" :style="{ height: -dy + 'px' }">
        <view
          class="u-loading-flower"
          :class="{ spin: freshing }"
          :style="{
            height: loaderSize + 'px',
            width: loaderSize + 'px',
            transform: `rotate(${(360 / 100) * (freshing ? 100 : (-dy / threshold) * 100)}deg)`,
          }"
        ></view>
        <view style="width: 5px"></view>
        <view v-if="freshing">{{ loadingText }}</view>
        <view v-else>{{ refresherActived ? refresherActivedText : refresherText }}</view>
      </view>
    </slot>
    <view class="dimple-uni-scroll-content">
      <slot v-if="isNoData" name="noData">
        <view class="dimple-uni-scroll-no-data">{{ noDataText }}</view>
      </slot>
      <slot></slot>

      <slot v-if="isNoMore" name="noMore">
        <view class="dimple-uni-scroll-no-more">{{ noMoreText }}</view>
      </slot>
    </view>
    <view class="dimple-uni-scroll-autoload-flag"></view>

    <slot name="loadmorer" :threshold="threshold" :loading="loading">
      <view class="dimple-uni-scroll-loadmorer" :class="{ hidden: !loading }" :style="{ height: threshold + 'px', top: loadmorerTop + 'px', background: background }">
        <view v-if="loading" class="u-loading-flower spin" :style="{ height: loaderSize + 'px', width: loaderSize + 'px' }"></view>
        <view v-if="loading" style="width: 5px"></view>
        <view v-if="loading">{{ loadingText }}</view>
      </view>
    </slot>
  </view>
</template>

<script>
export default {
  props: {
    height: { type: String, default: '100%' },
    background: { type: String, default: '#eeeeee' },
    threshold: { type: Number, default: 60 },
    maxThreshold: { type: Number, default: 120 },
    limit: { type: Number, default: 20 },
    skip: { type: Number, default: 0 },
    total: { type: Number, default: 0 },
    refresherText: { type: String, default: '下拉刷新' },
    refresherActivedText: { type: String, default: '松开刷新' },
    loadingText: { type: String, default: '加载中...' },
    noDataText: { type: String, default: '暂无数据' },
    noMoreText: { type: String, default: '没有更多数据了' },
    loaderSize: { type: Number, default: 25 },
    disabled: { type: Boolean, default: false },
  },
  data() {
    return {
      freshing: false,
      loading: false,
      touchStartPageY: 0,
      dy: 0,
      initialized: false, // 是否经过初始化
      refresherFlagVisible: false,
      autoloadFlagVisible: false,
      loadmorerTop: 0,
      containerHeight: 0,
      contentHeight: 0,
    }
  },
  computed: {
    refresherActived() {
      return this.threshold + this.dy <= 0
    },
    isNoData() {
      if (this.freshing || this.loading || this.disabled || !this.initialized) return false
      return this.total === 0
    },
    isNoMore() {
      if (this.isNoData || this.freshing || this.loading || this.disabled || !this.initialized) return false
      return this.total <= this.skip
    },
  },
  methods: {
    touchstart(e) {
      if (this.freshing || this.loading || this.disabled || !this.refresherFlagVisible) return
      this.touchStartPageY = e.changedTouches[0].pageY
    },
    touchmove(e) {
      if (this.freshing || this.loading || !this.refresherFlagVisible) return
      const pageY = e.changedTouches[0].pageY

      if (pageY > this.touchStartPageY) {
        const offset = pageY - this.touchStartPageY
        this.dy = -Math.min(this.maxThreshold, offset)
      }
    },
    touchend(e) {
      if (this.freshing || this.loading || this.disabled || !this.refresherFlagVisible) return this.stop()
      const pageY = e.changedTouches[0].pageY
      const offset = pageY - this.touchStartPageY
      if (offset < this.threshold) return this.stop()
      this.dy = -this.threshold
      this.fetch()
    },
    async fetch() {
      if (this.loading || this.disabled || !this.refresherFlagVisible) return
      this.initialized = false
      this.freshing = true
      const currentPage = 1
      this.$emit('fetch', {
        stop: this.stop,
        skip: this.skip,
        limit: this.limit,
        page: currentPage,
        total: this.total,
      })
    },
    async loadmore() {
      if (this.freshing || this.disabled || !this.autoloadFlagVisible) return
      if (this.isNoMore || this.isNoData) return this.stop()
      this.loading = true
      const currentPage = Math.ceil(this.skip / this.limit)
      this.$emit('fetch', {
        loadmore: true,
        stop: this.stop,
        skip: this.skip,
        limit: this.limit,
        page: currentPage + 1,
        total: this.total,
      })
    },
    stop() {
      this.touchStartPageY = 0
      this.dy = 0
      this.freshing = false
      this.loading = false
      this.initialized = true
      this.getScrollContainerRect()
      this.getScrollContentRect()
    },
    getScrollContainerRect() {
      uni
        .createSelectorQuery()
        .in(this)
        .select('.dimple-uni-scroll')
        .boundingClientRect((data) => {
          this.containerHeight = data.height
          this.loadmorerTop = data.top + data.height - this.threshold
        })
        .exec()
    },
    getScrollContentRect() {
      uni
        .createSelectorQuery()
        .in(this)
        .select('.dimple-uni-scroll-content')
        .boundingClientRect((data) => {
          this.contentHeight = data.height
        })
        .exec()
    },
    addObserver() {
      this.refresherObserver = uni.createIntersectionObserver(this)
      this.refresherObserver.relativeTo('.dimple-uni-scroll').observe('.dimple-uni-scroll-refresher-flag', ({ intersectionRatio, time }) => {
        this.refresherFlagVisible = intersectionRatio > 0
      })

      this.autoloadObserver = uni.createIntersectionObserver(this)
      this.autoloadObserver.relativeTo('.dimple-uni-scroll').observe('.dimple-uni-scroll-autoload-flag', ({ intersectionRatio, time }) => {
        if (!intersectionRatio) return (this.autoloadFlagVisible = false)
        this.autoloadFlagVisible = true
        intersectionRatio > 0 && this.contentHeight >= this.containerHeight && this.loadmore()
      })
    },
    removeObserver() {
      this.refresherObserver && this.refresherObserver.disconnect()
      this.autoloadObserver && this.autoloadObserver.disconnect()
    },
  },
  mounted() {
    this.getScrollContainerRect()
    this.getScrollContentRect()
    this.addObserver()
  },
  destroyed() {
    this.removeObserver()
  },
}
</script>

<style scoped>
.dimple-uni-scroll {
  background: #eeeeee;
  /* height: 100%; */
  overflow: auto;
  position: relative;
}
.dimple-uni-scroll::-webkit-scrollbar {
  display: none;
}

.dimple-uni-scroll-refresher-flag {
  height: 1px;
}

.dimple-uni-scroll-refresher {
  text-align: center;
  display: flex;
  align-items: center;
  justify-content: center;
  overflow: hidden;
  color: #aaa;
  font-size: 14px;
}
.dimple-uni-scroll-refresher.animate {
  transition: all 0.5s;
}

.dimple-uni-scroll-content {
  min-height: 100%;
}

.dimple-uni-scroll-autoload-flag {
  height: 1px;
  background: red;
}

.dimple-uni-scroll-loadmorer {
  position: fixed;
  left: 0;
  bottom: 0;
  width: 100%;
  z-index: 99999;
  text-align: center;
  display: flex;
  align-items: center;
  justify-content: center;
  overflow: hidden;
  color: #aaa;
  font-size: 14px;
  transition: all 0.2s;
}

.dimple-uni-scroll-loadmorer.hidden {
  /* height: 0px !important; */
  opacity: 0;
  z-index: -1;
}

.dimple-uni-scroll-no-data {
  text-align: center;
  padding: 20px;
  font-size: 14px;
  color: #aaa;
}
.dimple-uni-scroll-no-more {
  text-align: center;
  padding: 20px;
  padding-top: 0px;
  font-size: 14px;
  color: #aaa;
}

.u-loading-flower {
  width: 20px;
  height: 20px;
  display: inline-block;
  vertical-align: middle;
  /* animation: u-flower 1s steps(12) infinite; */
  background: transparent
    url(data:image/svg+xml;base64,PHN2ZyB4bWxucz0iaHR0cDovL3d3dy53My5vcmcvMjAwMC9zdmciIHdpZHRoPSIxMjAiIGhlaWdodD0iMTIwIiB2aWV3Qm94PSIwIDAgMTAwIDEwMCI+PHBhdGggZmlsbD0ibm9uZSIgZD0iTTAgMGgxMDB2MTAwSDB6Ii8+PHJlY3Qgd2lkdGg9IjciIGhlaWdodD0iMjAiIHg9IjQ2LjUiIHk9IjQwIiBmaWxsPSIjRTlFOUU5IiByeD0iNSIgcnk9IjUiIHRyYW5zZm9ybT0idHJhbnNsYXRlKDAgLTMwKSIvPjxyZWN0IHdpZHRoPSI3IiBoZWlnaHQ9IjIwIiB4PSI0Ni41IiB5PSI0MCIgZmlsbD0iIzk4OTY5NyIgcng9IjUiIHJ5PSI1IiB0cmFuc2Zvcm09InJvdGF0ZSgzMCAxMDUuOTggNjUpIi8+PHJlY3Qgd2lkdGg9IjciIGhlaWdodD0iMjAiIHg9IjQ2LjUiIHk9IjQwIiBmaWxsPSIjOUI5OTlBIiByeD0iNSIgcnk9IjUiIHRyYW5zZm9ybT0icm90YXRlKDYwIDc1Ljk4IDY1KSIvPjxyZWN0IHdpZHRoPSI3IiBoZWlnaHQ9IjIwIiB4PSI0Ni41IiB5PSI0MCIgZmlsbD0iI0EzQTFBMiIgcng9IjUiIHJ5PSI1IiB0cmFuc2Zvcm09InJvdGF0ZSg5MCA2NSA2NSkiLz48cmVjdCB3aWR0aD0iNyIgaGVpZ2h0PSIyMCIgeD0iNDYuNSIgeT0iNDAiIGZpbGw9IiNBQkE5QUEiIHJ4PSI1IiByeT0iNSIgdHJhbnNmb3JtPSJyb3RhdGUoMTIwIDU4LjY2IDY1KSIvPjxyZWN0IHdpZHRoPSI3IiBoZWlnaHQ9IjIwIiB4PSI0Ni41IiB5PSI0MCIgZmlsbD0iI0IyQjJCMiIgcng9IjUiIHJ5PSI1IiB0cmFuc2Zvcm09InJvdGF0ZSgxNTAgNTQuMDIgNjUpIi8+PHJlY3Qgd2lkdGg9IjciIGhlaWdodD0iMjAiIHg9IjQ2LjUiIHk9IjQwIiBmaWxsPSIjQkFCOEI5IiByeD0iNSIgcnk9IjUiIHRyYW5zZm9ybT0icm90YXRlKDE4MCA1MCA2NSkiLz48cmVjdCB3aWR0aD0iNyIgaGVpZ2h0PSIyMCIgeD0iNDYuNSIgeT0iNDAiIGZpbGw9IiNDMkMwQzEiIHJ4PSI1IiByeT0iNSIgdHJhbnNmb3JtPSJyb3RhdGUoLTE1MCA0NS45OCA2NSkiLz48cmVjdCB3aWR0aD0iNyIgaGVpZ2h0PSIyMCIgeD0iNDYuNSIgeT0iNDAiIGZpbGw9IiNDQkNCQ0IiIHJ4PSI1IiByeT0iNSIgdHJhbnNmb3JtPSJyb3RhdGUoLTEyMCA0MS4zNCA2NSkiLz48cmVjdCB3aWR0aD0iNyIgaGVpZ2h0PSIyMCIgeD0iNDYuNSIgeT0iNDAiIGZpbGw9IiNEMkQyRDIiIHJ4PSI1IiByeT0iNSIgdHJhbnNmb3JtPSJyb3RhdGUoLTkwIDM1IDY1KSIvPjxyZWN0IHdpZHRoPSI3IiBoZWlnaHQ9IjIwIiB4PSI0Ni41IiB5PSI0MCIgZmlsbD0iI0RBREFEQSIgcng9IjUiIHJ5PSI1IiB0cmFuc2Zvcm09InJvdGF0ZSgtNjAgMjQuMDIgNjUpIi8+PHJlY3Qgd2lkdGg9IjciIGhlaWdodD0iMjAiIHg9IjQ2LjUiIHk9IjQwIiBmaWxsPSIjRTJFMkUyIiByeD0iNSIgcnk9IjUiIHRyYW5zZm9ybT0icm90YXRlKC0zMCAtNS45OCA2NSkiLz48L3N2Zz4=)
    no-repeat;
  background-size: 100%;
}
.u-loading-flower.spin {
  animation: u-flower 1s steps(12) infinite;
}
@keyframes u-flower {
  0% {
    -webkit-transform: rotate(0deg);
    transform: rotate(0deg);
  }
  to {
    -webkit-transform: rotate(1turn);
    transform: rotate(1turn);
  }
}
@-webkit-keyframes u-circle {
  0% {
    transform: rotate(0);
  }
  100% {
    transform: rotate(360deg);
  }
}
</style>
