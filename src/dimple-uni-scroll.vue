<template>
  <view class="dimple-uni-scroll-container" :style="{ height: height }">
    <view class="dimple-uni-scroll-mask" v-show="freshing || loading"></view>
    <swiper vertical class="dimple-uni-scroll-swiper" :style="{ background: background }" @transition="handleTransition" @touchend="handleTouchEnd">
      <swiper-item class="dimple-uni-scroll-swiper-item" :style="swiperItemStyle">
        <view class="dimple-uni-scroll-refresher" :style="{ height: threshold + 'px' }">
          <slot name="refresher" :dy="dy" :threshold="threshold" :loading="freshing">
            <view class="dimple-uni-scroll-refresher-slot">
              <view></view>
              <view class="dimple-uni-scroll-refresher-slot-content">
                <scroll-loader :progress="freshing ? 100 : (-dy / threshold) * 100" :spin="freshing"></scroll-loader>
                <view style="width: 5px"></view>
                <view>
                  <template v-if="freshing"> 加载中... </template>
                  <template v-else>
                    {{ refresherActived ? '松开刷新' : '下拉刷新' }}
                  </template>
                </view>
              </view>
            </view>
          </slot>
        </view>
        <scroll-view class="dimple-uni-scroll" scroll-y :style="{ background: background }" @scroll="handleScroll">
          <slot v-if="isNoData" name="no-data"><view class="dimple-uni-scroll-no-data">暂无数据</view></slot>
          <slot></slot>
          <slot v-if="isNoMore" name="no-more"><view class="dimple-uni-scroll-no-more">到底了~</view></slot>
        </scroll-view>

        <view v-if="!isNoMore && !isNoData" class="dimple-uni-scroll-loadmorer" :style="{ height: threshold + 'px' }">
          <slot name="loadmorer" :dy="dy" :threshold="threshold" :loading="loading">
            <view class="dimple-uni-scroll-loadmorer-slot">
              <view class="dimple-uni-scroll-loadmorer-slot-content">
                <scroll-loader :progress="loading ? 100 : (dy / threshold) * 100" :spin="loading"></scroll-loader>
                <view style="width: 5px"></view>
                <view>
                  <template v-if="loading"> 加载中... </template>
                  <template v-else>
                    {{ loadmorerActived ? '松开加载' : '上拉加载' }}
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
    skip: { type: Number, default: 0 },
    total: { type: Number, default: -1 },
  },
  data() {
    return {
      dy: 0,
      freshing: false,
      loading: false,
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
      if (this.loading) return `top: ${-this.threshold}px`
      return `top: ${this.freshing ? this.threshold : 0}px`
    },
    isNoData() {
      return this.total === 0 && !this.freshing
    },
    isNoMore() {
      return this.total === this.skip && !this.loading
    },
  },
  methods: {
    handleTransition(e) {
      this.$emit('transition', e)
      const dy = e.detail.dy
      this.dy = dy
    },
    handleScroll(e) {
      this.$emit('scroll', e)
      this.dy = 0
    },
    async handleTouchEnd(e) {
      // 重置dy，防止意外触发（如swiper嵌套）刷新和加载，swiper动画时长默认是500ms
      if (this.timer) clearTimeout(this.timer)
      const time = 500 + 50
      this.timer = setTimeout(() => (this.dy = 0), time)

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
        pageIndex: currentPage,
        pageSize: this.limit,
        total: this.total,
      })
    },
    async loadmore() {
      if (this.isNoMore || this.isNoData) return this.stop()
      this.loading = true
      const currentPage = Math.ceil(this.skip / this.limit) + 1
      this.$emit('fetch', {
        loadmore: true,
        stop: this.stop,
        skip: this.skip,
        limit: this.limit,
        page: currentPage,
        pageIndex: currentPage,
        pageSize: this.limit,
        total: this.total,
      })
    },
    stop() {
      this.freshing = false
      this.loading = false
      this.dy = 0
    },
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
  font-size: 14px;
  color: #aaa;
}
</style>
