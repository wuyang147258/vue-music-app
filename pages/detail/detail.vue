<template>
	<view class="detail">
		<view class="fixbg"  :style="{ 'background-image' : 'url('+songdetail.al.picUrl+')' }"></view>
		<musichead :title="songdetail.name" :icon="true" color="white"></musichead>
		<view class="container"  v-show="!isLoading">
			<scroll-view scroll-y="true">
				<view class="detail-play" @tap="handlePlay">
					<image :src="songdetail.al.picUrl" :class="{'detail-play-run':isPlayRotate}"></image>
					<text class="iconfont" :class="iconplay"></text>
					<view></view>
				</view>
				<!-- 歌词部分 内框要比外框大  实现歌词流动 -->
				<view class="detail-lyric">
					<view class="detail-lyric-wrap" :style="{transform:'translateY('+-(lyricIndex-1)*82+'rpx)'}">
						<view class="detail-lyric-item" :class="{active:lyricIndex===index}" v-for="(item,index) in songLyric" :key="index">{{item.lyric}}</view>
						
					</view>
				</view>
				<!-- 有多少人喜欢部分 -->
				<view class="detail-like">
					<view class="detail-like-head">喜欢这首歌的人也听</view>
					<view class="detail-like-item"  v-for="(item,index) in songSimi" :key="index" @tap="handleToSimi(item.id)">
						<view class="detail-like-img">
							<image :src="item.album.blurPicUrl"></image>
						</view>
						<view class="detail-like-song">
							<view>{{item.name}}</view>
							<view>
								<image v-if="item.privilege.flag >60&&item.privilege.flag<70" src="../../static/dujia.png"></image>
								<image v-if="item.privilege.maxbr==999000" src="../../static/sq.png"></image>
								{{item.album.artists[0].name}}-{{item.name}}
							</view>
						</view>
						<text class="iconfont iconbofang"></text>
					</view>
				</view>
				<!-- 评论区 -->
				<view class="detail-comment">
					<view class="detail-comment-head">精彩评论</view>
					<view class="detail-comment-item" v-for="(item ,index) in songComment" :key="index">
						<!-- 左右布局 -->
						<view class="detail-comment-img">
							<image :src="item.user.avatarUrl"></image>
						</view>
						<view class="detail-comment-content">
							<view class="detail-common-title">
								<view class="detail-comment-name">
									<view>{{item.user.nickname}}</view>
									<view>{{item.time | formatTime}}</view>
								</view>
								<view class="detail-comment-like">{{item.likedCount | formatCount}}<text class="iconfont iconlike"></text></view>
							</view>
							<div class="clear"></div>
							<view class="detail-comment-text">{{item.content}}</view>
						</view>
					</view>
				</view>
			</scroll-view>
		</view>
	</view>
</template>

<script>
	import {songDetail,songSimi,songComment,songLyric,songUrl} from '../../common/api.js'
	import musichead from '../../components/musichead/musichead.vue'
	import '@/common/iconfont.css'
	export default {
	components:{musichead},
		data() {
			return {
				//歌曲基本信息数据
				songdetail:{
					al:{
						picUrl:''
					}
				},
				//相似歌曲数据
				songSimi:[],
				//评论数据
				songComment:[],
				//歌词数据
				songLyric:[],
				//控制歌词的显示
				lyricIndex:0,
				//控制播放暂停小图标的状态
				iconplay:'iconpause',
				isPlayRotate:true,
				isLoading:true
				
			}
		},
		onLoad(options) {
			// console.log(options.songId)
			this.getMusic(options.songId)
			
		},
		onUnload() {
			this.cancelLyriIdex()
			// #ifdef H5
			this.bgAudioManager.destroy()
			// #endif
		},
		onHide() {
			this.cancelLyriIdex()
			// #ifdef H5
			this.bgAudioManager.destroy()
			// #endif
		}
		,
		methods: {
			getMusic(songId){
				this.$store.commit('NEXT_ID',songId)
				uni.showLoading({
					title:'加载中'
				})
				this.isLoading=true
				Promise.all([songDetail(songId),songSimi(songId),songComment(songId),songLyric(songId),songUrl(songId)]).then((res)=>{
					if(res[0][1].data.code=='200'){
						this.songdetail=res[0][1].data.songs[0]
						console.log(this.songdetail)
					}
					if(res[1][1].data.code=='200'){
						this.songSimi=res[1][1].data.songs
					}
					if(res[2][1].data.code=='200'){
						this.songComment=res[2][1].data.hotComments
					}
					if(res[3][1].data.code=='200'){
						//利用正则处理歌词
						let lyric=res[3][1].data.lrc.lyric
						// console.log(lyric)
						let re=/\[([^\]]+)\]([^\[]+)/g
						var result=[];
						lyric.replace(re,($0,$1,$2)=>{
							result.push({"time":this.formatTimeToSec($1),"lyric":$2})
						})
						this.songLyric=result
					}
					if(res[4][1].data.code=='200'){
						//#ifdef MP-WEIXIN
						this.bgAudioManager = uni.getBackgroundAudioManager();
						this.bgAudioManager.title = this.songdetail.name;
						// #endif
						
						// #ifdef H5
						if(!this.bgAudioManager ){
						this.bgAudioManager = uni.createInnerAudioContext();
						}
						
						this.iconplay='iconbofang1'
						this.isPlayRotate=false
						// #endif
					
						this.bgAudioManager.src = res[4][1].data.data[0].url || '';
						this.listenLyricIndex()
						this.bgAudioManager.onPlay(()=>{
							this.iconplay='iconpause'	
							this.isPlayRotate=true
							this.listenLyricIndex()
						})
						this.bgAudioManager.onPause(()=>{
							this.iconplay='iconbofang1'
							this.isPlayRotate=false
							this.cancelLyriIdex()
						})
						this.bgAudioManager.onEnded(()=>{
							this.getMusic(this.$store.state.nextId)
						})
					}
						this.isLoading=false
						uni.hideLoading()
	
				})
			},
			formatTimeToSec(value){
					let arr = value.split(':');
					return (Number(arr[0]*60) +  Number(arr[1])).toFixed(1);
			},
			handlePlay(){
				if(this.bgAudioManager.paused){
					this.bgAudioManager.play()
				}else{
					this.bgAudioManager.pause()
				}
			},
			//实现歌词滚动
			listenLyricIndex(){
				//清除之前存在的计时器
				clearInterval(this.timer)
				this.timer=setInterval(()=>{
					for(var i=0;i<this.songLyric.length;i++){
						if(this.bgAudioManager.currentTime>this.songLyric[this.songLyric.length-1].time){
							this.lyricIndex=this.songLyric.length-1
							break;
						}
						
						if(this.bgAudioManager.currentTime>this.songLyric[i].time && this.bgAudioManager.currentTime<this.songLyric[i+1].time){
							this.lyricIndex=i
						}
						
					}
					console.log('歌曲时间'+this.bgAudioManager.currentTime)
					
				
				},500)
			},
			//关闭定时器
			cancelLyriIdex(){
				clearInterval(this.timer)
			},
			//相似歌曲
			handleToSimi(songId){
				this.getMusic(songId)
			}
		}
	}
</script>

<style scoped>
.detail-play{ width: 580rpx; height: 580rpx; background: url(~@/static/disc.png); background-size: cover; margin: 214rpx auto 0 auto; position: relative;}
.detail-play image{width: 370rpx; height: 370rpx; border-radius: 50%; position: absolute; left: 0; top: 0; right: 0; bottom: 0; margin: auto; animation: 10s linear move infinite;animation-play-state: paused;}
.detail-play .detail-play-run{animation-play-state: running;}
@keyframes move{
	from{transform: rotate(0deg);}
	to{transform: rotate(360deg);}
}
.detail-play text{width: 100rpx; height: 100rpx; font-size: 100rpx; color: white; position: absolute; left: 0; top: 0; right: 0; bottom: 0; margin: auto;}
.detail-play view{width: 230rpx; height: 360rpx; background: url(../../static/needle.png); position: absolute; left: 100rpx; right: 0; margin: auto;background-size: cover; top: -200rpx;}
.detail-lyric{font-size: 32rpx; line-height: 82rpx; height: 246rpx; text-align: center; overflow: hidden; color: #6f6793;}
.detail-lyric-wrap{transition: .5s;}
.detail-lyric-item{height: 82rpx;}
.detail-lyric-item.active{color: white;}


.detail-like{margin: 0 30rpx;}
.detail-like-head{font-size: 36rpx; color: white; margin: 50rpx 0;}
.detail-like-item{display: flex; align-items: center; margin-bottom: 28rpx;}
.detail-like-item text{font-size: 50rpx; color:#c6c2bf;}
.detail-like-img{width: 82rpx; height: 82rpx;border-radius: 20rpx; overflow:hidden;margin-right: 20rpx;}
.detail-like-img image{width: 100%;height: 100%;}
.detail-like-song{flex: 1; color:#c6c2bf;}
.detail-like-song view:nth-child(1){font-size: 28rpx; color: white; margin-bottom: 12rpx;}
.detail-like-song view:nth-child(2){font-size: 22rpx}
.detail-like-song image{width: 26rpx; height: 20rpx; margin-right: 10rpx;}


.detail-comment{margin:0 30rpx;}
.detail-comment-head{font-size: 36rpx; color: white; margin:50rpx 0;}
.detail-comment-item{margin-bottom: 28rpx; display:flex;}
.detail-comment-img{width:64rpx; height:64rpx; border-radius: 50%; overflow:hidden; margin-right:18rpx;}
.detail-comment-img image{width: 100%; height: 100%;}
.detail-comment-content{flex:1; color: #cbcacf;}
.detail-comment-title{ height: 30rpx!important;}
.detail-comment-name{display: inline-block; float: left;}
.detail-comment-name view:nth-child(1){font-size: 26rpx;}
.detail-comment-name view:nth-child(2){font-size: 20rpx;}
.detail-comment-like{display:inline-block; font-size: 28rpx;float: right;}
.detail-comment-text{font-size: 28rpx; line-height: 40rpx; color: white; margin-top: 15rpx; border-bottom: 1rpx solid #e0e0e0; padding-bottom: 40rpx;}
.clear{clear:both}

</style>
