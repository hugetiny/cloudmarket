<script>
//#ifdef APP-PLUS
//极光登录插件,提前预取号，提高登录速度
const jv = uni.requireNativePlugin('JG-JVerification');
//#endif
/**
 * vuex管理登陆状态，具体可以参考官方登陆模板示例
 */
import { mapMutations } from 'vuex';
import { client, auth } from '@/common/cloud.js';
const pushClientInfoKey = 'pushClientInfo';
export default {
	methods: {
		...mapMutations(['login', 'setUserLocation']),
		/**
		 * 初始化极光一键登录
		 */
		async initLogin() {
			if (!jv) {
				console.log('不支持jv');
				return;
			}
			jv.init(
				{
					timeout: 7000,
					isProduction: true
				},
				result => {
					console.log('jv.init', result);
				}
			);
			jv.isInitSuccess(result => {
				console.log('jv.isInitSuccess', result);
			});
			jv.checkVerifyEnable(result => {
				console.log('jv.checkVerifyEnable', result);
			});
			//预取号
			jv.preLogin(7000, result => {
				console.log('jv.preLogin', result);
			});
		},
		/**
		 * 初始化push相关
		 */
		async initPush() {
			console.log('初始化push message');
			plus.push.addEventListener('receive', msg => {
				console.log('receive push message', msg);
			});
			plus.push.addEventListener('click', msg => {
				console.log('click push message', msg);
			});
			plus.push.getClientInfoAsync(
				info => {
					console.log('push.getClientInfoAsync', info);
					//提交到服务器保存，第一步存到本地缓存
					uni.setStorage({
						key: pushClientInfoKey,
						data: info
					});
				},
				err => {
					console.log('push.getClientInfoAsync', err);
				}
			);
		}
	},
	onLaunch: function() {
		console.log('App.vue 启动');
		//检查登录状态,如果有效，则刷新accessToken和refreshToken，这个是unicloud自己存的，每次返回不一样
		/* auth.getLoginState().then(state => {
			console.log('auth.getLoginState', state);
		}); */
		let userInfoKey = 'userInfo';
		//根据返回值决定是否要刷新短期访问令牌。
		auth.shouldRefreshAccessToken(() => {
			let userInfo = uni.getStorageSync(userInfoKey);
			console.log('auth.shouldRefreshAccessToken 判断是否可以刷新令牌', userInfo);
			return !userInfo || userInfo.id > 0;
		});
		uniCloud.on('loginStateExpire', () => {
			// 尝试重新登录
			console.error('loginStateExpire,尝试重新登录');
			let userInfo = uni.getStorageSync(userInfoKey) || {};
			if (userInfo.id) {
				//更新登陆状态,检查tokenExpire是否在有效期内
				let time = new Date().getTime();
				if (userInfo.tokenExpire < time) {
					console.log('登录信息过期了');
					uni.removeStorage({
						key: userInfoKey
					});
					return;
				}
				client
					.callFunction({
						name: 'token',
						data: {
							uid: userInfo.id,
							token: userInfo.token
						}
					})
					.then(
						res => {
							console.log(res);
							//写入缓存
							uni.setStorage({
								key: userInfoKey,
								data: res
							});
							auth.signInWithTicket(res.ticket).then(() => {
								// 登录成功
								console.log('客户端登录成功');
							});
						},
						err => {
							//登录信息失效
							uni.removeStorage({
								key: userInfoKey
							});
						}
					);
			} else {
				console.log('本地没有历史登录信息');
			}
		});
		uniCloud.on('refreshAccessToken', () => {
			// 此时 uniCloud 短期访问令牌已经刷新，可以尝试刷新自身业务的登录态
			let time = new Date().getTime();
			console.log('uniCloud 短期访问令牌已经刷新，可以尝试刷新自身业务的登录态', time);
			/* auth.getAccessToken().then(res => {
				console.log('getAccessToken', time, res);
				//暂时没用
				uni.setStorage({
					key: 'userAccessToken',
					data: res.accessToken
				});
			});
			 */
			auth.getUserInfo().then(res => {
				console.log('getUserInfo', time, res);
			});
			let userInfo = uni.getStorageSync(userInfoKey) || {};
			if (userInfo.id) {
				console.log('设置登录缓存信息到store', time);
				//兼容服务器无法获取用户id,将来可以删除掉
				this.login(userInfo);
				//提交设备信息到服务器
				uni.getStorage({
					key: pushClientInfoKey,
					success: info => {
						client.callFunction({
							name: 'unipush',
							data: {
								type: 'regist',
								...info.data
							}
						});
						console.log('regist push to uid');
					}
				});
			} else {
				//用户信息失效
				console.log('用户信息失效');
				/* auth.signOut().then(res=>{
					console.log("App.vue signOut",res);
				}); */
			}
		});
		let userLocationInfo = uni.getStorageSync('userLocationInfo') || {};
		if (userLocationInfo.latitude > 0) {
			this.setUserLocation(userLocationInfo);
		}
		//#ifdef APP-PLUS
		this.initLogin();
		this.initPush();
		//#endif
	},
	onShow: function() {
		console.log('App Show');
	},
	onHide: function() {
		console.log('App Hide');
	}
};
</script>

<style lang="scss">
/*
		全局公共样式和字体图标
	*/
@font-face {
	font-family: yticon;
	font-weight: normal;
	font-style: normal;
	src: url('/static/yticon2.ttf') format('truetype');
}

.yticon {
	font-family: 'yticon' !important;
	font-size: $font-lg;
	font-style: normal;
	-webkit-font-smoothing: antialiased;
	-moz-osx-font-smoothing: grayscale;
	transition: 0.4s;
}

.yticon.checked {
	color: $base-color;
}

.icon-yiguoqi1:before {
	content: '\e700';
}

.icon-iconfontshanchu1:before {
	content: '\e619';
}

.icon-iconfontweixin:before {
	content: '\e611';
}

.icon-alipay:before {
	content: '\e636';
}

.icon-shang:before {
	content: '\e624';
}

.icon-shouye:before {
	content: '\e626';
}

.icon-shanchu4:before {
	content: '\e622';
}

.icon-xiaoxi:before {
	content: '\e618';
}

.icon-jiantour-copy:before {
	content: '\e600';
}

.icon-fenxiang2:before {
	content: '\e61e';
}

.icon-pingjia:before {
	content: '\e67b';
}

.icon-daifukuan:before {
	content: '\e68f';
}

.icon-pinglun-copy:before {
	content: '\e612';
}

.icon-dianhua-copy:before {
	content: '\e621';
}

.icon-shoucang:before {
	content: '\e645';
}

.icon-xuanzhong2 {
	color: $font-color-disabled;
}

.icon-xuanzhong2:before {
	content: '\e62f';
}

.icon-gouwuche_:before {
	content: '\e630';
}

.icon-icon-test:before {
	content: '\e60c';
}

.icon-icon-test1:before {
	content: '\e632';
}

.icon-bianji:before {
	content: '\e646';
}

.icon-jiazailoading-A:before {
	content: '\e8fc';
}

.icon-zuoshang:before {
	content: '\e613';
}

.icon-jia2:before {
	content: '\e60a';
}

.icon-huifu:before {
	content: '\e68b';
}

.icon-sousuo:before {
	content: '\e7ce';
}

.icon-arrow-fine-up:before {
	content: '\e601';
}

.icon-hot:before {
	content: '\e60e';
}

.icon-lishijilu:before {
	content: '\e6b9';
}

.icon-zhengxinchaxun-zhifubaoceping-:before {
	content: '\e616';
}

.icon-naozhong:before {
	content: '\e64a';
}

.icon-xiatubiao--copy:before {
	content: '\e608';
}

.icon-shoucang_xuanzhongzhuangtai:before {
	content: '\e6a9';
}

.icon-jia1:before {
	content: '\e61c';
}

.icon-bangzhu1:before {
	content: '\e63d';
}

.icon-arrow-left-bottom:before {
	content: '\e602';
}

.icon-arrow-right-bottom:before {
	content: '\e603';
}

.icon-arrow-left-top:before {
	content: '\e604';
}

.icon-icon--:before {
	content: '\e744';
}

.icon-zuojiantou-up:before {
	content: '\e605';
}

.icon-xia:before {
	content: '\e62d';
}

.icon--jianhao:before {
	content: '\e60b';
}

.icon-weixinzhifu:before {
	content: '\e61a';
}

.icon-comment:before {
	content: '\e64f';
}

.icon-weixin:before {
	content: '\e61f';
}

.icon-fenlei1:before {
	content: '\e620';
}

.icon-erjiye-yucunkuan:before {
	content: '\e623';
}

.icon-Group-:before {
	content: '\e688';
}

.icon-you:before {
	content: '\e606';
}

.icon-forward:before {
	content: '\e607';
}

.icon-tuijian:before {
	content: '\e610';
}

.icon-bangzhu:before {
	content: '\e679';
}

.icon-share:before {
	content: '\e656';
}

.icon-yiguoqi:before {
	content: '\e997';
}

.icon-shezhi1:before {
	content: '\e61d';
}

.icon-fork:before {
	content: '\e61b';
}

.icon-kafei:before {
	content: '\e66a';
}

.icon-iLinkapp-:before {
	content: '\e654';
}

.icon-saomiao:before {
	content: '\e60d';
}

.icon-shezhi:before {
	content: '\e60f';
}

.icon-shouhoutuikuan:before {
	content: '\e631';
}

.icon-gouwuche:before {
	content: '\e609';
}

.icon-dizhi:before {
	content: '\e614';
}

.icon-fenlei:before {
	content: '\e706';
}

.icon-xingxing:before {
	content: '\e70b';
}

.icon-tuandui:before {
	content: '\e633';
}

.icon-zuanshi:before {
	content: '\e615';
}

.icon-zuo:before {
	content: '\e63c';
}

.icon-shoucang2:before {
	content: '\e62e';
}

.icon-shouhuodizhi:before {
	content: '\e712';
}

.icon-yishouhuo:before {
	content: '\e71a';
}

.icon-dianzan-ash:before {
	content: '\e617';
}
.icon-right::before {
	content: '\e801';
}
.icon-money::before {
	content: '\e802';
}
.icon-chps::before {
	content: '\e803';
}
.icon-dingwei::before {
	content: '\e804';
}
.icon-dunpai::before {
	content: '\e805';
}
view,
scroll-view,
swiper,
swiper-item,
cover-view,
cover-image,
icon,
text,
rich-text,
progress,
button,
checkbox,
form,
input,
label,
radio,
slider,
switch,
textarea,
navigator,
audio,
camera,
image,
video {
	box-sizing: border-box;
}
/* 骨架屏替代方案 */
.Skeleton {
	background: #f3f3f3;
	padding: 20upx 0;
	border-radius: 8upx;
}

/* 图片载入替代方案 */
.image-wrapper {
	font-size: 0;
	background: #f3f3f3;
	border-radius: 4px;

	image {
		width: 100%;
		height: 100%;
		transition: 0.6s;
		opacity: 0;

		&.loaded {
			opacity: 1;
		}
	}
}

.clamp {
	overflow: hidden;
	text-overflow: ellipsis;
	white-space: nowrap;
	display: block;
}

.common-hover {
	background: #f5f5f5;
}

/*边框*/
.b-b:after,
.b-t:after {
	position: absolute;
	z-index: 3;
	left: 0;
	right: 0;
	height: 0;
	content: '';
	transform: scaleY(0.5);
	border-bottom: 1px solid $border-color-base;
}

.b-b:after {
	bottom: 0;
}

.b-t:after {
	top: 0;
}

/* button样式改写 */
uni-button,
button {
	height: 80upx;
	line-height: 80upx;
	font-size: $font-lg + 2upx;
	font-weight: normal;

	&.no-border:before,
	&.no-border:after {
		border: 0;
	}
}

uni-button[type='default'],
button[type='default'] {
	color: $font-color-dark;
}

/* input 样式 */
.input-placeholder {
	color: #999999;
}

.placeholder {
	color: #999999;
}
.tags {
	margin-top: 6upx;
}
/* 标签 */
.tag {
	font-size: $font-ssm;
	color: $base-color;
	margin-right: 10upx;
	border: 1px solid $base-color;
	border-radius: 4upx;
	padding: 2upx 6upx;
	line-height: 1;
	display: inline-block;
	&.info {
	}
	&.warning {
		color: $font-color-warning;
		border-color: $font-color-warning;
	}
}
/* 价格 */
.price {
	color: $base-color;
	font-size: $font-base;
	margin-right: 12upx;
	&:before {
		content: '￥';
		font-size: 0.85em;
	}
	&.del {
		font-size: $font-sm;
		color: $font-color-light;
		text-decoration: line-through;
		&:before {
			font-size: $font-ssm;
		}
	}
	&.emphasis {
		font-size: $font-sm;
		color: $font-color-warning;
	}
	&.warning {
		color: $font-color-warning;
	}
}
</style>
