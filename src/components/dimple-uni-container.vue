<template>
  <view class="container" :style="{ background: bgColor }">
    <view class="header">
      <slot name="header"></slot>
    </view>

    <view class="content" >
      <view class="header-placeholder" :style="{ height: headerPlaceholderHeight + 'px' }"> </view>

      <view class="content-slot" v-if="!loading">
        <slot></slot>

				<view v-if="tabs.length>0">

				</view>
      </view>

      <view class="footer-placeholder" :style="{ height: footerPlaceholderHeight + 'px' }"> </view>
      <view v-if="footerPlaceholderHeight > 0" class="safe-bottom-placeholder"> </view>
    </view>

    <view class="footer" :style="{ background: footerBgColor }">
      <slot name="footer"></slot>
      <view v-if="footerPlaceholderHeight > 0" class="safe-bottom-placeholder"> </view>
    </view>
  </view>
</template>

<script>
export default {
  name: 'dimple-uni-container',
  props: {
    bgColor: {
      type: String,
      default: '#eeeeee',
    },
    footerBgColor: {
      type: String,
      default: '#ffffff',
    },
    tabs: {
      type: Array,
      default: [],
    },
  },
  data() {
    return {
      loading: true,
      headerPlaceholderHeight: 0,
      footerPlaceholderHeight: 0,
    }
  },
  methods: {
    getDomRect(className) {
      const query = uni.createSelectorQuery().in(this)
      return new Promise((res) => {
        query
          .select(className)
          .boundingClientRect((data) => res(data))
          .exec()
      })
    },
    async setHeaderFooterInfo() {
      this.loading = true
      await this.$nextTick()
      Promise.all([this.getDomRect('.header'), this.getDomRect('.footer')]).then(([headerRect, footerRect]) => {
        this.headerPlaceholderHeight = headerRect.height
        this.footerPlaceholderHeight = footerRect.height
        this.$nextTick(() => (this.loading = false))
      })
    },
  },
  async mounted() {
    this.setHeaderFooterInfo()
  },
}
</script>

<style scoped>
.container {
  height: 100%;
}

.header {
  position: fixed;
  left: 0;
  top: 0;
  width: 100%;
}

.footer {
  position: fixed;
  left: 0;
  bottom: 0;
  width: 100%;
}

.content {
  display: flex;
  flex-direction: column;
  height: 100%;
}

.content-slot {
  flex: 1;
  overflow: auto;
}

.safe-bottom-placeholder {
  height: constant(safe-area-inset-bottom);
  /* 兼容 iOS < 11.2 */
  height: env(safe-area-inset-bottom);
}
</style>
