<template>
	<view>
		<!-- #ifdef APP-->
		<view class='status'></view>
		<!-- #endif -->
		<form @submit="formSubmit" report-submit='true'>
			<view class='addAddress pad30'>
				<view class='list borRadius14'>
					<view class='item acea-row row-between-wrapper' style="border: none;">
						<view class='name'>姓名</view>
						<input type='text' placeholder='请输入姓名' placeholder-style="color:#ccc;" name='realName' :value="userAddress.realName"
							placeholder-class='placeholder' maxlength="4"></input>
					</view>
					<view class='item acea-row row-between-wrapper'>
						<view class='name'>联系电话</view>
						<input type='number' placeholder='请输入联系电话' placeholder-style="color:#ccc;" name="phone" :value='userAddress.phone'
							placeholder-class='placeholder' maxlength="11"></input>
					</view>
					<view class='item acea-row row-between-wrapper relative'>
						<view class='name'>送货地址</view>
						<view class="address">
							<picker class='num' @change="bindAddresChange" :value="index" :range="addressArray">
							    <view class="picker acea-row row-between-wrapper">
							      <view class='address'>{{addressArray[index]}}</view>
							      <view class='iconfont icon-xiangyou abs_right'></view>
							    </view>
							</picker>
						</view>
					</view>
					<!--<view class='item acea-row row-between-wrapper relative'>
						<view class='name'>所在地区</view>
						<view class="address">
							<picker mode="multiSelector" @change="bindRegionChange"
								@columnchange="bindMultiPickerColumnChange" :value="valueRegion" :range="multiArray">
								<view class='acea-row'>
									<view class="picker line1">{{region[0]}}，{{region[1]}}，{{region[2]}}</view>
									<view class='iconfont icon-xiangyou abs_right'></view>
								</view>
							</picker>
						</view>
					</view>
					<view class='item acea-row row-between-wrapper relative'>
						<view class='name'>详细地址</view>
						<input type='text' placeholder='请填写具体地址' placeholder-style="color:#ccc;" name='detail' placeholder-class='placeholder'
							v-model='userAddress.detail' maxlength="18"></input>
							<view class='iconfont icon-dizhi font-color abs_right' @tap="chooseLocation"></view>
					</view>-->
				</view>
				<view class='default acea-row row-middle borRadius14'>
					<checkbox-group @change='ChangeIsDefault'>
						<checkbox :checked="userAddress.isDefault" />设置为默认地址
					</checkbox-group>
				</view>

				<button class='keepBnt bg-color' form-type="submit">立即保存</button>
				<!-- #ifdef MP -->
				<view class="wechatAddress" v-if="!id" @click="getWxAddress">导入微信地址</view>
				<!-- #endif -->
				<!-- #ifdef H5 -->
				<view class="wechatAddress" v-if="this.$wechat.isWeixin() && !id" @click="getAddress">导入微信地址</view>
				<!-- #endif -->
			</view>
		</form>
		<!-- #ifdef MP -->
		<!-- <authorize @onLoadFun="onLoadFun" :isAuto="isAuto" :isShowAuth="isShowAuth" @authColse="authColse"></authorize> -->
		<!-- #endif -->
		<!-- <home></home> -->
	</view>
</template>

<script>
	import {
		editAddress,
		getAddressDetail
	} from '@/api/user.js';
	import {
		getCity
	} from '@/api/api.js';
	import {
		toLogin
	} from '@/libs/login.js';
	import {
		mapGetters
	} from "vuex";
	// #ifdef MP
	import authorize from '@/components/Authorize';
	// #endif
	import home from '@/components/home';
	// import city from '@/utils/cityData';
	let app = getApp();
	export default {
		components: {
			// #ifdef MP
			authorize,
			// #endif
			home
		},
		data() {
			return {
				regionDval: ['浙江省', '杭州市', '滨江区'],
				cartId: '', //购物车id
				pinkId: 0, //拼团id
				couponId: 0, //优惠券id
				id: 0, //地址id
				userAddress: {
					isDefault: false
				}, //地址详情
				region: ['省', '市', '区'],
				valueRegion: [0, 0, 0],
				isAuto: false, //没有授权的不会自动授权
				isShowAuth: false, //是否隐藏授权
				district: [],
				multiArray: [],
				multiIndex: [0, 0, 0],
				cityId: 0,
				address: '',
				addressArray: ['安徽省马鞍山市第八高级中学','安徽省马鞍山市第二中学'],
				index: 0,
				defaultRegion: ['广东省', '广州市', '番禺区'],
				defaultRegionCode: '440113',
				bargain: false, //是否是砍价
				combination: false, //是否是拼团
				secKill: false, //是否是秒杀
			};
		},
		computed: mapGetters(['isLogin']),
		watch: {
			isLogin: {
				handler: function(newV, oldV) {
					if (newV) {
						this.getUserAddress();
						this.getCityList();
					}
				},
				deep: true
			}
		},
		onLoad(options) {
			if (this.isLogin) {
				this.preOrderNo = options.preOrderNo || 0;
				this.id = options.id || 0;
				uni.setNavigationBarTitle({
					title: options.id ? '修改地址' : '添加地址'
				})
				this.getUserAddress();
				if(this.$Cache.has('cityList')){
					//检测城市数据是否存在缓存，有的话从缓存取，没有的话请求接口
					this.district = this.$Cache.getItem('cityList')
					this.initialize();
				}else{
					this.getCityList();
				}
			} else {
				toLogin();
			}
		},
		methods: {
			// #ifdef APP-PLUS
			// 获取选择的地区
			handleGetRegion(region) {
				this.region = region
			},
			// #endif
			// 获取地址数据
			getCityList: function() {
				let that = this;
				getCity().then(res => {
					this.district = res.data;
					let oneDay = 24 * 3600 * 1000;
					// this.$Cache.set('cityList', JSON.stringify(res.data)); //设置不过期时间的方法 
					this.$Cache.setItem({name:'cityList',value:res.data,expires:oneDay * 7});  //设置七天过期时间
					that.initialize();
				})
			},
			initialize: function() {
				let that = this,province = [],city = [],area = [];
				if (that.district.length) {
					let cityChildren = that.district[0].child || [];
					let areaChildren = cityChildren.length ? (cityChildren[0].child || []) : [];
					that.district.forEach(function(item) {
						province.push(item.name);
					});
					cityChildren.forEach(function(item) {
						city.push(item.name);
					});
					areaChildren.forEach(function(item) {
						area.push(item.name);
					});
					this.multiArray = [province, city, area]
				}
			},
			bindRegionChange: function(e) {
				let multiIndex = this.multiIndex,
					province = this.district[multiIndex[0]] || {
						child: []
					},
					city = province.child[multiIndex[1]] || {
						cityId: 0
					},
					multiArray = this.multiArray,
					value = e.detail.value;

				this.region = [multiArray[0][value[0]], multiArray[1][value[1]], multiArray[2][value[2]]]
				this.cityId = city.cityId
				this.valueRegion = [0, 0, 0]
				this.initialize();
			},
			bindMultiPickerColumnChange: function(e) {
				let that = this,
					column = e.detail.column,
					value = e.detail.value,
					currentCity = this.district[value] || {
						child: []
					},
					multiArray = that.multiArray,
					multiIndex = that.multiIndex;
				multiIndex[column] = value;
				switch (column) {
					case 0:
						let areaList = currentCity.child[0] || {
							child: []
						};
						multiArray[1] = currentCity.child.map((item) => {
							return item.name;
						});
						multiArray[2] = areaList.child.map((item) => {
							return item.name;
						});
						break;
					case 1:
						let cityList = that.district[multiIndex[0]].child[multiIndex[1]].child || [];
						multiArray[2] = cityList.map((item) => {
							return item.name;
						});
						break;
					case 2:

						break;
				}
				// #ifdef MP || APP-PLUS
				this.$set(this.multiArray, 0, multiArray[0]);
				this.$set(this.multiArray, 1, multiArray[1]);
				this.$set(this.multiArray, 2, multiArray[2]);
				// #endif
				// #ifdef H5
				this.multiArray = multiArray;
				// #endif
				this.multiIndex = multiIndex
				// this.setData({ multiArray: multiArray, multiIndex: multiIndex});
			},
			// 授权回调
			onLoadFun: function() {
				this.getUserAddress();
			},
			// 授权关闭
			authColse: function(e) {
				this.isShowAuth = e
			},
			toggleTab(str) {
				this.$refs[str].show();
			},
			onConfirm(val) {
				this.region = val.checkArr[0] + '-' + val.checkArr[1] + '-' + val.checkArr[2];
			},
			getUserAddress: function() {
				if (!this.id) return false;
				let that = this;
				getAddressDetail(this.id).then(res => {
					let region = [res.data.province, res.data.city, res.data.district];
					that.$set(that, 'userAddress', res.data);
					that.$set(that, 'region', region);
					that.city_id = res.data.cityId
				});
			},
			chooseLocation: function () {
				uni.chooseLocation({
					success: (res) => {
						this.$set(this.userAddress,'detail',res.address.replace(/.+?(省|市|自治区|自治州|县|区)/g,''));
					}
				})
			},
			// 导入共享地址（小程序）
			getWxAddress: function() {
				let that = this;
				uni.authorize({
					scope: 'scope.address',
					success: function(res) {
						uni.chooseAddress({
							success: function(res) {
								let addressP = {};
								addressP.province = res.provinceName;
								addressP.city = res.cityName;
								addressP.district = res.countyName;
								addressP.cityId = 0;
								editAddress({
									address: addressP,
									isDefault: 1,
									realName: res.userName,
									postCode: res.postalCode,
									phone: res.telNumber,
									detail: res.detailInfo,
									id: 0
								}).then(res => {
									setTimeout(function() {
										if (that.cartId) {
											let cartId = that.cartId;
											let pinkId = that.pinkId;
											let couponId = that.couponId;
											that.cartId = '';
											that.pinkId = '';
											that.couponId = '';
											uni.navigateTo({
												url: '/pages/users/order_confirm/index?cartId=' +
													cartId +
													'&addressId=' + (
														that.id ? that
														.id :
														res.data
														.id) +
													'&pinkId=' +
													pinkId +
													'&couponId=' +
													couponId +
													'&secKill=' + that
													.secKill +
													'&combination=' +
													that.combination +
													'&bargain=' + that
													.bargain
											});
										} else {
											uni.navigateBack({
												delta: 1
											});
										}
									}, 1000);
									return that.$util.Tips({
										title: "添加成功",
										icon: 'success'
									});
								}).catch(err => {
									return that.$util.Tips({
										title: err
									});
								});
							},
							fail: function(res) {
								if (res.errMsg == 'chooseAddress:cancel') return that.$util
									.Tips({
										title: '取消选择'
									});
							},
						})
					},
					fail: function(res) {
						uni.showModal({
							title: '您已拒绝导入微信地址权限',
							content: '是否进入权限管理，调整授权？',
							success(res) {
								if (res.confirm) {
									uni.openSetting({
										success: function(res) {}
									});
								} else if (res.cancel) {
									return that.$util.Tips({
										title: '已取消！'
									});
								}
							}
						})
					},
				})
			},
			// 导入共享地址（微信）；
			getAddress() {
				let that = this;
				that.$wechat.openAddress().then(userInfo => {
					// open();
					editAddress({
							id: this.id,
							realName: userInfo.userName,
							phone: userInfo.telNumber,
							address: {
								province: userInfo.provinceName,
								city: userInfo.cityName,
								district: userInfo.countryName,
								cityId: 0
							},
							detail: userInfo.detailInfo,
							isDefault: 1,
							postCode: userInfo.postalCode
						})
						.then(() => {
							setTimeout(function() {
								if (that.cartId) {
									let cartId = that.cartId;
									let pinkId = that.pinkId;
									let couponId = that.couponId;
									that.cartId = '';
									that.pinkId = '';
									that.couponId = '';
									uni.navigateTo({
										url: '/pages/users/order_confirm/index?cartId=' +
											cartId + '&addressId=' + (that.id ? that.id :
												res.data
												.id) + '&pinkId=' + pinkId + '&couponId=' +
											couponId + '&secKill=' + that.secKill +
											'&combination=' + that.combination + '&bargain=' +
											that.bargain
									});
								} else {
									uni.navigateTo({
										url: '/pages/users/user_address_list/index'
									})
									// history.back();
								}
							}, 1000);
							// close();
							that.$util.Tips({
								title: "添加成功",
								icon: 'success'
							});
						})
						.catch(err => {
							// close();
							return that.$util.Tips({
								title: err || "添加失败"
							});
						});
				}).catch(err => {
					console.log(err);
				});
			},
			/**
			 * 提交用户添加地址
			 * 
			 */
			formSubmit: function(e) {
				let that = this,
					value = e.detail.value;
				if (!value.realName) return that.$util.Tips({
					title: '请填写收货人姓名'
				});
				if (!value.phone) return that.$util.Tips({
					title: '请填写联系电话'
				});
				if (!/^1(3|4|5|7|8|9|6)\d{9}$/i.test(value.phone)) return that.$util.Tips({
					title: '请输入正确的手机号码'
				});
				if (that.region == '省-市-区') return that.$util.Tips({
					title: '请选择所在地区'
				});
				/*if (!value.detail) return that.$util.Tips({
					title: '请填写详细地址'
				});*/
				value.id = that.id;
				let regionArray = that.region;
				value.address = that.address;
				value.isDefault = that.userAddress.isDefault;

				uni.showLoading({
					title: '保存中',
					mask: true
				})
				editAddress(value).then(res => {
					if (that.id)
						that.$util.Tips({
							title: '修改成功',
							icon: 'success'
						});
					else
						that.$util.Tips({
							title: '添加成功',
							icon: 'success'
						});
					setTimeout(function() {
						if (that.preOrderNo>0) {
							uni.redirectTo({
								url: '/pages/users/order_confirm/index?preOrderNo=' + that.preOrderNo + '&addressId=' + (that.id ? that.id : res.data.id)
							})
						} else {
							// #ifdef H5
							return history.back();
							// #endif
							// #ifndef H5
							return uni.navigateBack({
								delta: 1,
							})
							// #endif
						}
					}, 1000);
				}).catch(err => {
					return that.$util.Tips({
						title: err
					});
				})
			},
			ChangeIsDefault: function(e) {
				this.$set(this.userAddress, 'isDefault', !this.userAddress.isDefault);
			},
			bindAddresChange: function (e) {
					this.$set(this,'index',e.detail.value);
					this.$set(this,'address',this.addressArray[e.detail.value]);
					
			}
		}
	}
</script>

<style scoped lang="scss">
	.addAddress {
		padding-top: 20rpx;
	}
	.addAddress .list {
		background-color: #fff;
		padding: 0 24rpx;
	}

	.addAddress .list .item {
		border-top: 1rpx solid #eee;
		height: 90rpx;
		line-height: 90rpx;
	}

	.addAddress .list .item .name {
		// width: 195rpx;
		font-size: 30rpx;
		color: #333;
	}

	.addAddress .list .item .address {
		flex: 1;
		margin-left: 32rpx;
	}

	.addAddress .list .item input {
		width: 475rpx;
		font-size: 30rpx;
		font-weight: 400;
	}

	.addAddress .list .item .placeholder {
		color: #ccc;
	}

	.addAddress .list .item picker .picker {
		width: 475rpx;
		font-size: 30rpx;
	}

	.addAddress .default {
		padding: 0 30rpx;
		height: 90rpx;
		background-color: #fff;
		margin-top: 23rpx;
	}

	.addAddress .default checkbox {
		margin-right: 15rpx;
	}

	.addAddress .keepBnt {
		width: 690rpx;
		height: 86rpx;
		border-radius: 50rpx;
		text-align: center;
		line-height: 86rpx;
		margin: 80rpx auto 24rpx auto;
		font-size: 32rpx;
		color: #fff;
	}

	.addAddress .wechatAddress {
		width: 690rpx;
		height: 86rpx;
		border-radius: 50rpx;
		text-align: center;
		line-height: 86rpx;
		margin: 0 auto;
		font-size: 32rpx;
		color: #E93323 ;
		border: 1px solid #E93323;
	}
	.relative{
		position: relative;
	}
	.icon-dizhi{
		font-size: 44rpx;
		z-index: 100;
	}
	.abs_right{
		position: absolute;
		right:0;
	}
	
	.status{
		display: flex;
		width: 750rpx;
		// background-color: #E93323;
		height: var(--status-bar-height);
	}
	
</style>
