<script module="refresh" lang="wxs">
function onPulling(e, ins) {
	var dy = e.detail.dy || e.detail.deltaY
	var dataset = e.currentTarget.dataset
	var threshold = dataset.threshold
	ins.selectComponent('.dimple-uni-scroll-refresher').setStyle({
		height: Math.max(0, dy) + 'px'
	})
	ins.selectComponent('.refresher-icon').setStyle({
		transform: 'rotate(' + (360 / 100) * (dy / threshold) * 100 + 'deg)',
	})
	var active = dy >= threshold
	ins.selectComponent('.dimple-uni-scroll-refresher-text').setStyle({
		display: active ? 'none' : 'inline',
	})
	ins.selectComponent('.dimple-uni-scroll-refresher-active-text').setStyle({
		display: !active ? 'none' : 'inline',
	})
}

function onRefresh(e, ins) {
	var dataset = e.currentTarget.dataset
	var threshold = dataset.threshold
	ins.selectComponent('.dimple-uni-scroll-refresher').setStyle({
		height: threshold + 'px',
	})
	ins.selectComponent('.dimple-uni-scroll-refresher-text').setStyle({
		display: 'none',
	})
	ins.selectComponent('.dimple-uni-scroll-refresher-active-text').setStyle({
		display: 'none',
	})
	ins.callMethod('fetch')
}

function onAbort(e, ins) {
	var dataset = e.currentTarget.dataset
	var threshold = dataset.threshold
	ins.selectComponent('.dimple-uni-scroll-refresher').setStyle({
		height: 0 + 'px',
	})
}

module.exports = {
	onPulling: onPulling,
	onRefresh: onRefresh,
	onAbort: onAbort
}
</script>

<template>
  <scroll-view
    class="dimple-uni-scroll"
    :style="{ height: height, background: background }"
    scroll-y
    :refresher-enabled="refreshDisabled ? false : !disabled"
    :refresher-threshold="threshold"
    :refresher-triggered="triggered"
    :refresher-background="refresherBackground"
    refresher-default-style="none"
    :data-threshold="threshold"
    :scroll-top="scrollTop"
    @refresherpulling="refresh.onPulling"
    @refresherrefresh="refresh.onRefresh"
    @refresherrestore="onRestore"
    @refresherabort="refresh.onAbort"
    @scroll="handleScroll"
  >
    <view
      slot="refresher"
      class="dimple-uni-scroll-refresher"
      :style="{ background: refresherBackground }"
    >
      <view
        class="u-loading-flower refresher-icon"
        :class="{ spin: freshing }"
        :style="{ height: loaderSize + 'px', width: loaderSize + 'px' }"
      ></view>
      <view style="width: 5px"></view>
      <view v-if="freshing">{{ freshingText }}</view>
      <view class="dimple-uni-scroll-refresher-text">
        {{ refresherText }}
      </view>
      <view class="dimple-uni-scroll-refresher-active-text">
        {{ refresherActivedText }}
      </view>
    </view>
    <view class="dimple-uni-scroll-content">
      <slot v-if="error && !freshing" name="error">
        <view class="dimple-uni-scroll-no-data"> {{ errorText }} </view>
      </slot>
      <slot v-if="isNoData" name="noData">
        <view
          class="dimple-uni-scroll-no-data"
          :style="[computedNoDataStyle]"
          >{{ noDataText }}</view
        >
      </slot>

      <view v-if="!error" class="dimple-uni-scroll-content-slot">
        <slot></slot>
      </view>
      <slot v-if="isNoMore" name="noMore">
        <view
          class="dimple-uni-scroll-no-more"
          :style="[computedNoMoreStyle]"
          >{{ noMoreText }}</view
        >
      </slot>
    </view>
    <view
      class="dimple-uni-scroll-loadmorer"
      :class="{
        'dimple-uni-scroll-loadmorer-hidden':
          isNoData || isNoMore || error || disabled || loadmoreDisabled,
      }"
    >
      <slot
        v-if="!isNoMore && loading && !error"
        name="loadmorer"
        :threshold="threshold"
        :loading="loading"
      >
        <view
          class="u-loading-flower spin"
          :style="{ height: loaderSize + 'px', width: loaderSize + 'px' }"
        >
        </view>
        <view style="width: 5px"></view>
        <view>{{ loadingText }}</view>
      </slot>
    </view>
  </scroll-view>
</template>

<script>
export default {
  props: {
    height: { type: String, default: "100%" },
    background: { type: String, default: "#eeeeee" },
    threshold: { type: Number, default: 80 },
    limit: { type: Number, default: 20 },
    skip: { type: Number, default: -1 },
    total: { type: Number, default: -1 },
    refresherBackground: { type: String, default: "#eeeeee" },
    refresherText: { type: String, default: "下拉刷新" },
    refresherActivedText: { type: String, default: "松开刷新" },
    freshingText: { type: String, default: "刷新中..." },
    loadingText: { type: String, default: "加载中..." },
    noDataText: { type: String, default: "暂无数据" },
    noMoreText: { type: String, default: "没有更多数据了" },
    loaderSize: { type: Number, default: 25 },
    disabled: { type: Boolean, default: false },
    refreshDisabled: { type: Boolean, default: false },
    loadmoreDisabled: { type: Boolean, default: false },
    noData: { type: [Boolean, Number], default: -1 },
    noMore: { type: [Boolean, Number], default: -1 },
    error: { type: Boolean, default: false },
    errorText: { type: String, default: "服务异常，请稍后刷新重试" },
  },
  data() {
    return {
      triggered: false,
      freshing: false,
      loading: false,
      scrollTop: 0,
      oldScrollTop: 0,
    };
  },
  computed: {
    isNoData() {
      if (this.error) return false;
      if (this.noData === true || this.noData === false) return this.noData;
      if (
        this.freshing ||
        this.loading ||
        this.disabled ||
        this.refreshDisabled
      )
        return false;
      if (this.total < 0 || this.skip < 0) return false;
      return this.total === 0;
    },
    isNoMore() {
      if (this.error) return false;
      if (this.noMore === true || this.noMore === false) return this.noMore;
      if (
        this.isNoData ||
        this.freshing ||
        this.loading ||
        this.disabled ||
        this.loadmoreDisabled
      )
        return false;
      if (this.total < 0 || this.skip < 0) return false;
      return this.total <= this.skip;
    },
    computedNoDataStyle() {
      return {
        "text-align": "center",
        padding: "20px",
        "font-size": "14px",
        color: "#aaa",
      };
    },
    computedNoMoreStyle() {
      return {
        "text-align": "center",
        padding: "20px",
        "font-size": "14px",
        color: "#aaa",
        "padding-top": "0px",
      };
    },
  },
  watch: {
    total(newValue, oldValue) {
      if (oldValue > -1 && newValue === -1) this.setScrollTop(0);
    },
  },
  methods: {
    onRestore() {
      this.triggered = "restore"; // 需要重置
    },
    handleScroll(e) {
      this.$emit("scroll", e);
      this.oldScrollTop = e.detail.scrollTop;
      // this.setScrollTop(e.detail.scrollTop)
    },
    async fetch() {
      if (this.loading || this.disabled || this.refreshDisabled) return;
      this.triggered = true;
      this.freshing = true;
      const currentPage = 1;
      this.$emit("fetch", {
        stop: this.stop,
        skip: this.skip,
        limit: this.limit,
        page: currentPage,
        total: this.total,
      });
    },
    async loadmore() {
      if (this.freshing || this.disabled || this.loadmoreDisabled) return;
      if (this.isNoMore || this.isNoData) return this.stop();
      this.loading = true;
      const currentPage = Math.ceil(this.skip / this.limit);
      this.$emit("fetch", {
        loadmore: true,
        stop: this.stop,
        skip: this.skip,
        limit: this.limit,
        page: currentPage + 1,
        total: this.total,
      });
    },
    stop() {
      setTimeout(() => {
        this.freshing = false;
        this.loading = false;
        this.triggered = false;
      }, 0);

      Promise.all([
        this.getContainerRect(),
        this.getScrollSlotContentRect(),
      ]).then((result) => {
        const [containerRect, contentRect] = result;
        const containerHeight = containerRect.height;
        const contentHeight = contentRect.height;
        if (
          contentHeight <= containerHeight &&
          !this.isNoData &&
          !this.isNoMore
        ) {
          this.setScrollTop(0);
        }
      });
    },
    setScrollTop(value) {
      this.scrollTop = this.oldScrollTop;
      this.$nextTick(() => {
        this.scrollTop = value;
      });
    },
    getContainerRect() {
      const query = uni.createSelectorQuery().in(this);
      return new Promise((res) => {
        query
          .select(".dimple-uni-scroll")
          .boundingClientRect((data) => res(data))
          .exec();
      });
    },
    getScrollSlotContentRect() {
      const query = uni.createSelectorQuery().in(this);
      return new Promise((res) => {
        query
          .select(".dimple-uni-scroll-content-slot")
          .boundingClientRect((data) => res(data))
          .exec();
      });
    },
    addObserver() {
      // loadmorer监听
      this.autoloadObserver = uni.createIntersectionObserver(this);
      this.autoloadObserver
        .relativeTo(".dimple-uni-scroll")
        .observe(
          ".dimple-uni-scroll-loadmorer",
          ({ intersectionRatio, time }) => {
            if (this.isNoMore || this.isNoData) return;
            const visible = intersectionRatio > 0;
            visible && this.loadmore();
          }
        );
    },
    removeObserver() {
      this.autoloadObserver && this.autoloadObserver.disconnect();
    },
  },
  mounted() {
    this.addObserver();
  },
  destroyed() {
    this.removeObserver();
  },
};
</script>

<style scoped>
.dimple-uni-scroll {
  overflow: auto;
  position: relative;
}

.dimple-uni-scroll::-webkit-scrollbar {
  display: none;
}

.dimple-uni-scroll-refresher {
  width: 100%;
  text-align: center;
  display: flex;
  align-items: center;
  justify-content: center;
  overflow: hidden;
  color: #aaa;
  font-size: 14px;
}

.dimple-uni-scroll-content {
  min-height: 100%;
}

.dimple-uni-scroll-content-slot {
}

.dimple-uni-scroll-loadmorer {
  width: 100%;
  text-align: center;
  display: flex;
  align-items: center;
  justify-content: center;
  overflow: hidden;
  color: #aaa;
  font-size: 14px;
  min-height: 50px;
  margin-top: 10px;

  /* background: red; */
}

.dimple-uni-scroll-loadmorer-hidden {
  height: 0px;
  min-height: 0px;
  margin-top: 0px;
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
