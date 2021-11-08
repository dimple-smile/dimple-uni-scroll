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
                <scroll-loader :progress="freshing ? 100 : (-dy / threshold) * 100" :spin="freshing"></scroll-loader>
                <view style="width: 5px"></view>
                <view>
                  <template v-if="freshing"> {{ loadingText }} </template>
                  <template v-else>
                    {{ refresherActived ? refresherActivedText : refresherText }}
                  </template>
                </view>
              </view>
            </view>
          </slot>
        </view>
        <scroll-view class="dimple-uni-scroll" scroll-y :style="{ background: background }" @scroll="handleScroll">
          <view class="dimple-uni-scroll-content">
            <slot v-if="isNoData" name="no-data">
              <view class="dimple-uni-scroll-no-data">{{ noDataText }}</view>
            </slot>
            <slot></slot>
            <slot v-if="isNoMore" name="no-more">
              <view class="dimple-uni-scroll-no-more">{{ noMoreText }}</view>
            </slot>
          </view>
          <view class="dimple-uni-scroll-autoload-flag"></view>
        </scroll-view>

        <view v-if="!isNoMore && !isNoData" class="dimple-uni-scroll-loadmorer" :style="{ height: threshold + 'px' }">
          <slot name="loadmorer" :dy="dy" :threshold="threshold" :loading="loading">
            <view class="dimple-uni-scroll-loadmorer-slot">
              <view class="dimple-uni-scroll-loadmorer-slot-content">
                <scroll-loader :progress="loading ? 100 : (dy / threshold) * 100" :spin="loading"></scroll-loader>
                <view style="width: 5px"></view>
                <view>
                  <template v-if="loading"> {{ loadingText }} </template>
                  <template v-else>
                    {{ loadmorerActived ? loadmorerActivedText : loadmorerText }}
                  </template>
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
import ScrollLoader from './dimple-uni-scroll-loader.vue'
export default {
  components: { ScrollLoader },
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
    noMoreText: { type: String, default: '到底了~' },
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
        page: currentPage + 1,
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

<style scoped lang="scss">
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
</style>
