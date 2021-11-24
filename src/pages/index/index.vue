<template>
  <view class="dimple-uni-scroll-demo">
    <dimple-uni-scroll :total="total" :skip="skip" :limit="20" @fetch="fetch">
      <view class="scroll-content">
        <view v-for="(item, index) in list" :key="index" class="scroll-item">{{ index + 1 }}</view>
      </view>
    </dimple-uni-scroll>
  </view>
</template>

<script>
import DimpleUniScroll from '../../components/dimple-uni-scroll.vue'
export default {
  components: { DimpleUniScroll },
  data() {
    return {
      list: [],
      total: -1,
      skip: -1,
    }
  },
  methods: {
    async fetch(e) {
      const { stop } = e
      try {
        await this.getData(e)
      } catch {}
      stop()
    },
    async getData(options = {}) {
      const { page = 1, skip = 0, limit = 20 } = options
      const res = await new Promise((res) =>
        setTimeout(() => {
          const data = Array(limit)
            .fill('')
            .map((item, index) => (page === 1 ? index : skip + index))
          res(data)
        }, 2000)
      )
      this.total = 100
      if (page === 1) this.list = []
      this.list.push(...res)
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

<style scoped>
.dimple-uni-scroll-demo {
  /* 如果这里设置了继承page的100%，需要保证page设置了height: 100% */
  height: 100%;
}
.scroll-content {
  padding: 20px;
}
.scroll-item {
  background: #ffffff;
  border-radius: 2px;
  padding: 20px;
  margin-bottom: 20px;
}
</style>
