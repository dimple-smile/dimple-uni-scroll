<template>
  <view class="dimple-uni-scroll-container" :style="{ height: height }" @touchend="handleTouchEnd">
    <view class="dimple-uni-scroll-mask" v-show="freshing || loading"></view>
    <swiper class="dimple-uni-scroll-swiper" :style="{ background: background }" vertical touchable @transition="handleTransition" @animationfinish="handleAnimationFinish">
      <swiper-item class="dimple-uni-scroll-swiper-item" :style="swiperItemStyle">
        <view class="dimple-uni-scroll-refresher" :style="{ height: threshold + 'px' }">
          <slot name="refresher" :dy="dy" :threshold="threshold" :loading="freshing">
            <view class="dimple-uni-scroll-refresher-slot">
              <view></view>
              <view class="dimple-uni-scroll-refresher-slot-content">
                <view
                  class="u-loading-flower"
                  :class="{ spin: freshing }"
                  :style="{ height: loaderSize + 'px', width: loaderSize + 'px', transform: `rotate(${(360 / 100) * (freshing ? 100 : (-dy / threshold) * 100)}deg)` }"
                >
                </view>
                <view style="width: 5px"></view>
                <view v-if="freshing"> {{ loadingText }} </view>
                <view v-else>
                  {{ refresherActived ? refresherActivedText : refresherText }}
                </view>
              </view>
            </view>
          </slot>
        </view>
        <scroll-view class="dimple-uni-scroll" scroll-y :style="{ background: background }" @scroll="handleScroll">
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
        </scroll-view>

        <view v-if="!isNoMore && !isNoData" class="dimple-uni-scroll-loadmorer" :style="{ height: threshold + 'px' }">
          <slot name="loadmorer" :dy="dy" :threshold="threshold" :loading="loading">
            <view class="dimple-uni-scroll-loadmorer-slot">
              <view class="dimple-uni-scroll-loadmorer-slot-content">
                <view
                  class="u-loading-flower"
                  :class="{ spin: loading }"
                  :style="{ height: loaderSize + 'px', width: loaderSize + 'px', transform: `rotate(${(360 / 100) * (loading ? 100 : (dy / threshold) * 100)}deg)` }"
                >
                </view>
                <view style="width: 5px"></view>
                <view v-if="loading"> {{ loadingText }} </view>
                <view v-else>
                  {{ loadmorerActived ? loadmorerActivedText : loadmorerText }}
                </view>
              </view>
              <view></view>
            </view>
          </slot>
        </view>
      </swiper-item>
    </swiper>
  </view>
</template>

<script>
export default {
  props: {
    height: { type: String, default: '100%' },
    background: { type: String, default: '#eeeeee' },
    threshold: { type: Number, default: 60 },
    limit: { type: Number, default: 20 },
    skip: { type: Number, default: -1 },
    total: { type: Number, default: -1 },
    autoload: { type: Boolean, default: true },
    refresherText: { type: String, default: '下拉刷新' },
    refresherActivedText: { type: String, default: '松开刷新' },
    loadmorerText: { type: String, default: '上拉加载' },
    loadmorerActivedText: { type: String, default: '松开加载' },
    loadingText: { type: String, default: '加载中...' },
    noDataText: { type: String, default: '暂无数据' },
    noMoreText: { type: String, default: '没有更多数据了' },
    loaderSize: { type: Number, default: 25 },
  },
  data() {
    return {
      dy: 0, // 下拉/上拉的偏移量
      freshing: false,
      loading: false,
      swiperHeight: 0,
      loadmoreFlagVisible: false,
    }
  },
  computed: {
    refresherActived() {
      return this.threshold + this.dy <= 0
    },
    loadmorerActived() {
      return this.dy - this.threshold >= 0
    },
    swiperItemStyle() {
      let top = 0
      if (this.freshing) top = this.threshold
      if (this.loading) top = -this.threshold
      return `top: ${top}px`
    },
    isNoData() {
      return this.total === 0 && !this.freshing
    },
    isNoMore() {
      if (this.isNoData || this.freshing) return false
      if (this.total < 0 || this.skip < 0) return false
      return this.total <= this.skip && !this.loading
    },
  },
  methods: {
    handleTransition(e) {
      this.$emit('transition', e)
      let dy = e.detail.dy
      this.dy = dy
    },
    handleScroll(e) {
      this.$emit('scroll', e)
    },
    handleAnimationFinish(e) {
      // swiper动画结束重置dy，防止下次操作误判和swiper嵌套时错误触发下拉/上拉
      this.dy = 0
    },
    async handleTouchEnd(e) {
      // 判断下拉/上拉距离是否过大，过大可能是因为下拉时突然上拉，触发上拉的dy值转换，导致dy值过大，上拉也一样有这个问题
      if (Math.abs(this.dy) > this.swiperHeight / 2) return
      if (this.dy === 0) return
      if (this.dy < 0) {
        if (this.dy + this.threshold <= 0) this.fetch()
      }
      if (this.dy > 0) {
        if (this.dy - this.threshold >= 0) this.loadmore()
      }
    },
    async fetch() {
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
      if (this.freshing) return
      if (this.isNoMore || this.isNoData) return this.stop()
      this.loading = true
      const currentPage = Math.ceil(this.skip / this.limit) + 1
      this.$emit('fetch', {
        loadmore: true,
        stop: this.stop,
        skip: this.skip,
        limit: this.limit,
        page: currentPage,
        total: this.total,
      })
    },
    stop() {
      this.freshing = false
      this.loading = false
      this.dy = 0
    },
    getSwiperHeight() {
      uni
        .createSelectorQuery()
        .in(this)
        .select('.dimple-uni-scroll-swiper')
        .fields({ size: true }, (data) => (this.swiperHeight = data.height))
        .exec()
    },
    setAutoloadObserver() {
      this.autoloadObserver = uni.createIntersectionObserver(this)
      this.autoloadObserver.relativeTo('.dimple-uni-scroll').observe('.dimple-uni-scroll-autoload-flag', ({ intersectionRatio, time }) => {
        if (!intersectionRatio) return
        if (!this.loadmoreFlagVisible) return (this.loadmoreFlagVisible = true)
        intersectionRatio > 0 && this.loadmore()
      })
    },
  },
  mounted() {
    this.getSwiperHeight()
    this.autoload && this.setAutoloadObserver()
  },
  onUnload() {
    this.autoloadObserver?.disconnect()
  },
}
</script>

<style scoped>
.dimple-uni-scroll-container {
  position: relative;
}

.dimple-uni-scroll-mask {
  background: rgba(0, 0, 0, 0);
  position: absolute;
  height: 100%;
  width: 100%;
  left: 0;
  top: 0;
  z-index: 9999;
}

.dimple-uni-scroll-swiper {
  position: relative;
  height: 100%;
}

.dimple-uni-scroll-swiper-item {
  position: absolute;
  width: 100%;
  height: 100%;
  transition: all 0.3s;
  position: relative;
  overflow: visible;
}

.dimple-uni-scroll {
  height: 100%;
}
.dimple-uni-scroll-autoload-flag {
  height: 1px;
}
.dimple-uni-scroll ::-webkit-scrollbar {
  display: none;
}

.dimple-uni-scroll-refresher {
  position: absolute;
  z-index: 0;
  bottom: 100%;
  left: 0;
  width: 100%;
}
.dimple-uni-scroll-refresher-slot {
  height: 100%;
  width: 100%;
  text-align: center;
  display: flex;
  flex-direction: column;
  justify-content: space-between;
}
.dimple-uni-scroll-refresher-slot-content {
  display: flex;
  width: 100%;
  display: flex;
  align-items: center;
  justify-content: center;
  color: #aaa;
  font-size: 14px;
}
.dimple-uni-scroll-loadmorer {
  position: absolute;
  z-index: 0;
  top: 100%;
  left: 0;
  width: 100%;
}

.dimple-uni-scroll-loadmorer-slot {
  height: 100%;
  width: 100%;
  text-align: center;
  display: flex;
  flex-direction: column;
  justify-content: space-between;
}
.dimple-uni-scroll-loadmorer-slot-content {
  display: flex;
  width: 100%;
  display: flex;
  align-items: center;
  justify-content: center;
  color: #aaa;
  font-size: 14px;
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
