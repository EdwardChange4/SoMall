
<template>
   <view class="content">
      <view class="navbar">
         <view v-for="(item, index) in navList" :key="index" class="nav-item" :class="{current: tabCurrentIndex === index}" @click="tabClick(index)">
            {{item.text}}
         </view>
      </view>

      <swiper :current="tabCurrentIndex" class="swiper-box" duration="300" @change="changeTab">
         <swiper-item class="tab-content" v-for="(tabItem,tabIndex) in navList" :key="tabIndex">
            <scroll-view class="list-scroll-content" scroll-y @scrolltolower="loadData">
               <!-- 空白页 -->
               <empty v-if="tabItem.loaded === true && tabItem.orderList.length === 0"></empty>

               <!-- 订单列表 -->
               <view v-for="(item,index) in tabItem.orderList" :key="index" class="order-item">
                  <view class="i-top b-b">
                     <text class="time">{{item.creationTime}}</text>
                     <text class="state" :style="{color: item.stateTipColor}">{{item.state}}</text>
                  </view>

                  <scroll-view v-if="item.orderItems.length > 1" class="goods-box" scroll-x>
                     <view v-for="(goodsItem, goodsIndex) in item.orderItems" :key="goodsIndex" class="goods-item">
                        <image class="goods-img" :src="goodsItem.skuCoverImageUrl" mode="aspectFill"></image>
                     </view>
                  </scroll-view>
                  <template v-if="item.orderItems.length === 1">
                     <view class="goods-box-single" v-for="(goodsItem, goodsIndex) in item.orderItems" :key="goodsIndex">
                        <image class="goods-img" :src="goodsItem.skuCoverImageUrl" mode="aspectFill"></image>
                        <view class="right">
                           <text class="title clamp">{{goodsItem.spuName}}</text>
                           <text class="attr-box">{{goodsItem.skuName}} x {{goodsItem.num}}</text>
                           <text class="price">{{goodsItem.skuPrice}}</text>
                        </view>
                     </view>
                  </template>
                  <view class="price-box">
                     共
                     <text class="num"> {{item.orderItems.length}}</text>
                     件商品 实付款
                     <text class="price">143.7</text>
                  </view>
                  <view class="action-box b-t" v-if="item.state != 9">
                     <button class="action-btn" @click="cancelOrder(item)">取消订单</button>
                     <button class="action-btn recom" @tap="rePay(item)">重新支付</button>
                  </view>
               </view>

               <uni-load-more :status="tabItem.loadingType"></uni-load-more>

            </scroll-view>
         </swiper-item>
      </swiper>
   </view>
</template>

<script lang="ts">
import { Component, Vue, Inject, Watch, Ref } from "vue-property-decorator";
import { BaseView } from "../baseView";
import empty from "@/components/empty.vue";
import api from "@/utils/api";
import { Tips } from "@/utils/tips";

@Component({ components: { empty } })
export default class Orders extends BaseView {
   async onLoad(options: any) {
      /**
       * 修复app端点击除全部订单外的按钮进入时不加载数据的问题
       * 替换onLoad下代码即可
       */
      // this.tabCurrentIndex = +options.state || 0;
   }

   onShow() {
      if (uni.getStorageSync("Order_Select_Index")) {
         this.tabCurrentIndex = uni.getStorageSync("Order_Select_Index");
         uni.removeStorageSync("Order_Select_Index");
      }
   }

   async created() {
      // #ifndef MP
      await this.loadData();
      // #endif

      // #ifdef MP
      if (this.tabCurrentIndex == 0) {
         await this.loadData();
      }
      // #endif
   }

   tabCurrentIndex = 0;
   navList = [
      {
         state: 0,
         text: "全部",
         loadingType: "more",
         orderList: [],
         loaded: false,
         total: 0,
         page: 0
      },
      {
         state: 1,
         text: "待付款",
         loadingType: "more",
         orderList: [],
         loaded: false,
         total: 0,
         page: 0
      },
      {
         state: 2,
         text: "待收货",
         loadingType: "more",
         orderList: [],
         loaded: false,
         total: 0,
         page: 0
      },
      {
         state: 3,
         text: "待评价",
         loadingType: "more",
         orderList: [],
         loaded: false,
         total: 0,
         page: 0
      },
      {
         state: 4,
         text: "售后",
         loadingType: "more",
         orderList: [],
         loaded: false,
         total: 0,
         page: 0
      }
   ];

   //获取订单列表
   async loadData(source: any = null) {
      //这里是将订单挂载到tab列表下
      let index = this.tabCurrentIndex;
      let navItem = this.navList[index];
      let state = navItem.state;

      if (source === "tabChange" && navItem.loaded === true) {
         //tab切换只有第一次需要加载数据
         return;
      }
      if (
         navItem.loadingType === "loading" ||
         navItem.loadingType === "noMore"
      ) {
         //防止重复加载
         return;
      }
      console.log("loadData");
      navItem.loadingType = "loading";
      await api
         .order_getList({ state, skipCount: navItem.page * 10 })
         .then((res: any) => {
            //loaded新字段用于表示数据加载完毕，如果为空可以显示空白页
            this.$set(navItem, "loaded", true);
            this.$set(navItem, "orderList", [
               ...navItem.orderList,
               ...res.items
            ]);
            this.$set(navItem, "page", navItem.page + 1);
            this.$set(navItem, "total", res.totalCount);
            //判断是否还有数据， 有改为 more， 没有改为noMore
            if (navItem.total > navItem.page * 10) navItem.loadingType = "more";
            else navItem.loadingType = "noMore";
         });
   }

   //swiper 切换
   changeTab(e: any) {
      this.tabCurrentIndex = +e.target.current;
      this.loadData("tabChange");
   }
   //顶部tab点击
   tabClick(index: number) {
      console.log("index:" + index);
      this.tabCurrentIndex = +index;
   }
   //删除订单
   deleteOrder(index: any) {
      this.navList[this.tabCurrentIndex].orderList.splice(index, 1);
   }
   //取消订单
   cancelOrder(item: any) {
      uni.showLoading({
         title: "请稍后"
      });

      //取消订单后删除待付款中该项
      let list = this.navList[this.tabCurrentIndex].orderList;
      let index = list.findIndex((val: any) => val.id === item.id);
      index !== -1 && list.splice(index, 1);

      uni.hideLoading();
   }

   // repay
   async rePay(item: any) {
      await api
         .tenpay({
            orderId: item.id,
            openid: this.openid
         })
         .then((obj: any) => {
            console.log(obj);

            uni.requestPayment({
               timeStamp: obj.timeStamp,
               nonceStr: obj.nonce_str,
               package: "prepay_id=" + obj.prepay_id,
               signType: "MD5",
               paySign: obj.paySign,
               success: res => {
                  Tips.info("支付成功!");
               },
               fail: res => {
                  //{errMsg: "requestPayment:fail cancel"} 用户取消
                  if (res.errMsg === "requestPayment:fail cancel") {
                     Tips.info("用户取消了支付!");
                     return;
                  }
                  console.log("pay fail:", res);
                  Tips.info("支付失败:" + res.errMsg);
               }
            });
         });
   }
}
</script>


<style lang="scss">
page,
.content {
   background: $page-color-base;
   height: 100%;
}

.swiper-box {
   height: calc(100% - 40px);
}
.list-scroll-content {
   height: 100%;
}

.navbar {
   display: flex;
   height: 40px;
   padding: 0 5px;
   background: #fff;
   box-shadow: 0 1px 5px rgba(0, 0, 0, 0.06);
   position: relative;
   z-index: 10;
   .nav-item {
      flex: 1;
      display: flex;
      justify-content: center;
      align-items: center;
      height: 100%;
      font-size: 15px;
      color: $font-color-dark;
      position: relative;
      &.current {
         color: $base-color;
         &:after {
            content: "";
            position: absolute;
            left: 50%;
            bottom: 0;
            transform: translateX(-50%);
            width: 44px;
            height: 0;
            border-bottom: 2px solid $base-color;
         }
      }
   }
}

.uni-swiper-item {
   height: auto;
}
.order-item {
   display: flex;
   flex-direction: column;
   padding-left: 30upx;
   background: #fff;
   margin-top: 16upx;
   .i-top {
      display: flex;
      align-items: center;
      height: 80upx;
      padding-right: 30upx;
      font-size: $font-base;
      color: $font-color-dark;
      position: relative;
      .time {
         flex: 1;
      }
      .state {
         color: $base-color;
      }
      .del-btn {
         padding: 10upx 0 10upx 36upx;
         font-size: $font-lg;
         color: $font-color-light;
         position: relative;
         &:after {
            content: "";
            width: 0;
            height: 30upx;
            border-left: 1px solid $border-color-dark;
            position: absolute;
            left: 20upx;
            top: 50%;
            transform: translateY(-50%);
         }
      }
   }
   /* 多条商品 */
   .goods-box {
      height: 160upx;
      padding: 20upx 0;
      white-space: nowrap;
      .goods-item {
         width: 120upx;
         height: 120upx;
         display: inline-block;
         margin-right: 24upx;
      }
      .goods-img {
         display: block;
         width: 100%;
         height: 100%;
      }
   }
   /* 单条商品 */
   .goods-box-single {
      display: flex;
      padding: 20upx 0;
      .goods-img {
         display: block;
         width: 120upx;
         height: 120upx;
      }
      .right {
         flex: 1;
         display: flex;
         flex-direction: column;
         padding: 0 30upx 0 24upx;
         overflow: hidden;
         .title {
            font-size: $font-base + 2upx;
            color: $font-color-dark;
            line-height: 1;
         }
         .attr-box {
            font-size: $font-sm + 2upx;
            color: $font-color-light;
            padding: 10upx 12upx;
         }
         .price {
            font-size: $font-base + 2upx;
            color: $font-color-dark;
            &:before {
               content: "￥";
               font-size: $font-sm;
               margin: 0 2upx 0 8upx;
            }
         }
      }
   }

   .price-box {
      display: flex;
      justify-content: flex-end;
      align-items: baseline;
      padding: 20upx 30upx;
      font-size: $font-sm + 2upx;
      color: $font-color-light;
      .num {
         margin: 0 8upx;
         color: $font-color-dark;
      }
      .price {
         font-size: $font-lg;
         color: $font-color-dark;
         &:before {
            content: "￥";
            font-size: $font-sm;
            margin: 0 2upx 0 8upx;
         }
      }
   }
   .action-box {
      display: flex;
      justify-content: flex-end;
      align-items: center;
      height: 100upx;
      position: relative;
      padding-right: 30upx;
   }
   .action-btn {
      width: 160upx;
      height: 60upx;
      margin: 0;
      margin-left: 24upx;
      padding: 0;
      text-align: center;
      line-height: 60upx;
      font-size: $font-sm + 2upx;
      color: $font-color-dark;
      background: #fff;
      border-radius: 100px;
      &:after {
         border-radius: 100px;
      }
      &.recom {
         background: #fff9f9;
         color: $base-color;
         &:after {
            border-color: #f7bcc8;
         }
      }
   }
}

/* load-more */
.uni-load-more {
   display: flex;
   flex-direction: row;
   height: 80upx;
   align-items: center;
   justify-content: center;
}

.uni-load-more__text {
   font-size: 28upx;
   color: #999;
}

.uni-load-more__img {
   height: 24px;
   width: 24px;
   margin-right: 10px;
}

.uni-load-more__img > view {
   position: absolute;
}

.uni-load-more__img > view view {
   width: 6px;
   height: 2px;
   border-top-left-radius: 1px;
   border-bottom-left-radius: 1px;
   background: #999;
   position: absolute;
   opacity: 0.2;
   transform-origin: 50%;
   animation: load 1.56s ease infinite;
}

.uni-load-more__img > view view:nth-child(1) {
   transform: rotate(90deg);
   top: 2px;
   left: 9px;
}

.uni-load-more__img > view view:nth-child(2) {
   transform: rotate(180deg);
   top: 11px;
   right: 0;
}

.uni-load-more__img > view view:nth-child(3) {
   transform: rotate(270deg);
   bottom: 2px;
   left: 9px;
}

.uni-load-more__img > view view:nth-child(4) {
   top: 11px;
   left: 0;
}

.load1,
.load2,
.load3 {
   height: 24px;
   width: 24px;
}

.load2 {
   transform: rotate(30deg);
}

.load3 {
   transform: rotate(60deg);
}

.load1 view:nth-child(1) {
   animation-delay: 0s;
}

.load2 view:nth-child(1) {
   animation-delay: 0.13s;
}

.load3 view:nth-child(1) {
   animation-delay: 0.26s;
}

.load1 view:nth-child(2) {
   animation-delay: 0.39s;
}

.load2 view:nth-child(2) {
   animation-delay: 0.52s;
}

.load3 view:nth-child(2) {
   animation-delay: 0.65s;
}

.load1 view:nth-child(3) {
   animation-delay: 0.78s;
}

.load2 view:nth-child(3) {
   animation-delay: 0.91s;
}

.load3 view:nth-child(3) {
   animation-delay: 1.04s;
}

.load1 view:nth-child(4) {
   animation-delay: 1.17s;
}

.load2 view:nth-child(4) {
   animation-delay: 1.3s;
}

.load3 view:nth-child(4) {
   animation-delay: 1.43s;
}

@-webkit-keyframes load {
   0% {
      opacity: 1;
   }

   100% {
      opacity: 0.2;
   }
}
</style>
