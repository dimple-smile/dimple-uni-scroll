<template>
	<view class="dimple-uni-scroll-loader-container" :class="{ spin: spin }" :style="{ height: size + 'px', width: size + 'px' }">
		<template v-for="item in density">
			<view v-if="item <= (progress * density) / 100" :key="item" class="dimple-uni-scroll-loader-item" :style="itemStyle">
				<view
					class="dimple-uni-scroll-loader"
					:style="{ width: itemSize + 'px', height: itemSize + 'px', background: color, transform: `rotate(${(360 / density) * item}deg)` }"
				></view>
			</view>
		</template>
	</view>
</template>

<script>
export default {
	props: {
		size: { type: Number, default: 25 },
		density: { type: Number, default: 12 },
		progress: { type: Number, default: 0 },
		color: { type: String, default: 'rgba(0,0,0,0.05)' },
		spin: { type: Boolean, default: false }
	},
	computed: {
		itemSize() {
			const halfSize = this.size / 2;
			return Math.sqrt((halfSize * halfSize) / 2);
		},
		itemStyle() {
			const halfSize = this.size / 2;
			let itemSize = this.itemSize;
			return this.makeStyle({
				height: itemSize + 'px',
				width: itemSize + 'px',
				transform: `translateX(${halfSize - itemSize}px) translateY(${halfSize - itemSize}px) rotate(45deg)`
			});
		}
	},
	methods: {
		makeStyle(styles = {}) {
			return Object.keys(styles)
				.map(key => `${key}:${styles[key]}`)
				.join(';');
		}
	}
};
</script>

<style scoped>
.dimple-uni-scroll-loader-container {
	position: relative;
}
.dimple-uni-scroll-loader-container.spin {
	animation: dimple-uni-scroll-loader-container-spin 3s linear infinite;
}
.dimple-uni-scroll-loader-item {
	position: absolute;
	left: 0;
	top: 0;
	transform-origin: 100% 100%;
}
.dimple-uni-scroll-loader {
	border-radius: 0 100%;
	transform-origin: 100% 100%;
}
@keyframes dimple-uni-scroll-loader-container-spin {
	from {
		transform: rotate(0deg);
	}
	to {
		transform: rotate(360deg);
	}
}
</style>
