<template>
	<view class="music-head" :style="{color:color}">
		<view class="music-head-icon" v-if="icon">
			<text class="iconfont iconzuojiantou-copy" @tap="handleToBack"></text> |<text class="iconfont iconshouye" @tap="handleToHome"></text>
		</view>
		{{title}}
	</view>
</template>

<script>
	import '@/common/iconfont.css'
	export default {
		name:"musichead",
		data() {
			return {
				handleToBack(){
					uni.navigateBack()
				},
				handleToHome(){
					uni.navigateTo({
						url:'/pages/index/index'
					})
				}
			};
		},
		methods:{
			
		},
		props:['title','icon','color',]
	}
</script>

<style scoped>
	.music-head{
		position: relative;
		width: 100%;
		height: 75px;
		font-size: 16px;
		line-height: 80px;
		text-align: center;
		color: block;
	}
	.music-head-icon{
		position: absolute;
		left: 8px;
		top: 28px;
		width: 90px;
		height: 30px;
		line-height: 31px;
		background-color: rgba(0,0,0,0.4);
		border-radius: 15px;
		display: flex;
		justify-content: space-evenly;
	}
	
</style>
