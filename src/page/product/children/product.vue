<template>
  <div class="shoplist_container">
    <ul v-load-more="loaderMore" v-if="productList.length" type="1">
      <li v-for="item in productList">
      <router-link :to="{path: 'shop', query:{geohash, id: item.id}}"  class="shop_li">
        <section>
          <img :src="item.image" class="shop_img">
        </section>
        <hgroup class="shop_right">
          <header class="shop_detail_header">
            <h4  class="" class="shop_title ellipsis">{{item.name}}</h4>
          </header>
          <h5 class="rating_order_num">
            <section class="rating_order_num_left">
              <section class="rating_section">
                <rating-star :rating='item.rating'></rating-star>
                <span class="rating_num">{{item.rating}}</span>
              </section>
              <section class="order_section">
                月售{{item.sellCount}}单
              </section>
            </section>
          </h5>
          <h5 class="fee_distance">
          <p class="fee">
            ¥{{item.price}}
          </p>
            <p class="distance_time">
            <buy-cart :product='item' @moveInCart="listenInCart" @showChooseList="showChooseList" @showReduceTip="showReduceTip" @showMoveDot="showMoveDotFun"></buy-cart>
          </p>
        </h5>

          <!--<footer class="menu_detail_footer">
            <section class="food_price">
              <span>¥</span>
              <span>{{item.price}}</span>
            </section>
            <buy-cart :shopId='shopId' :foods='foods' @moveInCart="listenInCart" @showChooseList="showChooseList" @showReduceTip="showReduceTip" @showMoveDot="showMoveDotFun"></buy-cart>
          </footer>-->
        </hgroup>
      </router-link>

      </li>
    </ul>
    <ul v-else class="animation_opactiy">
      <li class="list_back_li" v-for="item in 10" :key="item">
        <img src="../../../images/shopback.svg" class="list_back_svg">
      </li>
    </ul>
    <aside class="return_top" @click="backTop" v-if="showBackStatus">
      <svg class="back_top_svg">
        <use xmlns:xlink="http://www.w3.org/1999/xlink" xlink:href="#backtop"></use>
      </svg>
    </aside>
    <div ref="abc" style="background-color: red;"></div>
    <transition name="loading">
      <loading v-show="showLoading"></loading>
    </transition>
  </div>
</template>

<script>

  import {mapState} from 'vuex'
  import {shopList,getProductList} from 'src/service/getData'
  import {imgBaseUrl} from 'src/config/env'
  import {showBack, animate} from 'src/config/mUtils'
  import {loadMore, getImgPath} from 'src/components/common/mixin'
  import loading from 'src/components/common/loading'
  import ratingStar from 'src/components/common/ratingStar'
  import buyCart from 'src/page/product/buyCart'



  export default {
    data(){
      return {
        cartFoodList: [], //购物车商品列表
        productList:[],
        offset: 0, // 批次加载店铺列表，每次加载20个 limit = 20
        shopListArr:[], // 店铺列表数据
        preventRepeatReuqest: false, //到达底部加载数据，防止重复加载
        showBackStatus: false, //显示返回顶部按钮
        showLoading: true, //显示加载动画
        touchend: false, //没有更多数据
        imgBaseUrl,
      }
    },
    mounted(){
      this.initData();
    },
    components: {
      loading,
      ratingStar,
      buyCart

    },
    props: ['restaurantCategoryId', 'restaurantCategoryIds', 'sortByType', 'deliveryMode', 'supportIds', 'confirmSelect', 'geohash'],
    mixins: [loadMore, getImgPath],
    computed: {
      ...mapState([
        'latitude','longitude'
      ]),
    },
    updated(){
      // console.log(this.supportIds, this.sortByType)
    },
    methods: {
      async initData(){
        this.productList=getProductList();
        //获取数据
        let res = await shopList(this.latitude, this.longitude, this.offset, this.restaurantCategoryId);
        this.shopListArr = [...res];
        if (res.length < 20) {
          this.touchend = true;
        }
        this.hideLoading();
        //开始监听scrollTop的值，达到一定程度后显示返回顶部按钮
        showBack(status => {
          this.showBackStatus = status;
        });
      },
      //到达底部加载更多数据
      async loaderMore(){
        if (this.touchend) {
          return
        }
        //防止重复请求
        if (this.preventRepeatReuqest) {
          return
        }
        this.showLoading = true;
        this.preventRepeatReuqest = true;

        //数据的定位加20位
        this.offset += 20;
        let res = await shopList(this.latitude, this.longitude, this.offset, this.restaurantCategoryId);
        this.hideLoading();
        this.shopListArr = [...this.shopListArr, ...res];
        //当获取数据小于20，说明没有更多数据，不需要再次请求数据
        if (res.length < 20) {
          this.touchend = true;
          return
        }
        this.preventRepeatReuqest = false;
      },
      //返回顶部
      backTop(){
        animate(document.body, {scrollTop: '0'}, 400,'ease-out');
      },
      //监听父级传来的数据发生变化时，触发此函数重新根据属性值获取数据
      async listenPropChange(){
        this.showLoading = true;
        this.offset = 0;
        let res = await shopList(this.latitude, this.longitude, this.offset, '', this.restaurantCategoryIds, this.sortByType, this.deliveryMode, this.supportIds);
        this.hideLoading();
        //考虑到本地模拟数据是引用类型，所以返回一个新的数组
        this.shopListArr = [...res];
      },
      //开发环境与编译环境loading隐藏方式不同
      hideLoading(){
        this.showLoading = false;
      },
      //监听圆点是否进入购物车
      listenInCart(){
        if (!this.receiveInCart) {
          this.receiveInCart = true;
          this.$refs.cartContainer.addEventListener('animationend', () => {
            this.receiveInCart = false;
          })
          this.$refs.cartContainer.addEventListener('webkitAnimationEnd', () => {
            this.receiveInCart = false;
          })
        }
      },
      //显示规格列表
      showChooseList(foods){
        if (foods) {
          this.choosedFoods = foods;
        }
        this.showSpecs = !this.showSpecs;
        this.specsIndex = 0;
      },
      //记录当前所选规格的索引值
      chooseSpecs(index){
        this.specsIndex = index;
      },
      //多规格商品加入购物车
      addSpecs(category_id, item_id, food_id, name, price, specs, packing_fee, sku_id, stock){
        this.ADD_CART({shopid: this.shopId, category_id, item_id, food_id, name, price, specs, packing_fee, sku_id, stock});
        this.showChooseList();
      },
      //显示提示，无法减去商品
      showReduceTip(){
        this.showDeleteTip = true;
        clearTimeout(this.timer);
        this.timer = setTimeout(() => {
          clearTimeout(this.timer);
          this.showDeleteTip = false;
        }, 3000);
      },
      //显示下落圆球
      showMoveDotFun(showMoveDot, elLeft, elBottom){
        this.showMoveDot = [...this.showMoveDot, ...showMoveDot];
        this.elLeft = elLeft;
        this.elBottom = elBottom;
      },
      zhunshi(supports){
        let zhunStatus;
        if ((supports instanceof Array) && supports.length) {
          supports.forEach(item => {
            if (item.icon_name === '准') {
              zhunStatus = true;
            }
          })
        }else{
          zhunStatus = false;
        }
        return zhunStatus
      },
    },
    watch: {
      //监听父级传来的restaurantCategoryIds，当值发生变化的时候重新获取餐馆数据，作用于排序和筛选
      restaurantCategoryIds: function (value){
        this.listenPropChange();
      },
      //监听父级传来的排序方式
      sortByType: function (value){
        this.listenPropChange();
      },
      //监听父级的确认按钮是否被点击，并且返回一个自定义事件通知父级，已经接收到数据，此时父级才可以清除已选状态
      confirmSelect: function (value){
        this.listenPropChange();
      }
    }
  }
</script>

<style lang="scss" scoped>
  @import 'src/style/mixin';
  .shoplist_container{
    background-color: #fff;
    margin-bottom: 2rem;
  }
  .shop_li{
    display: flex;
    border-bottom: 0.025rem solid #f1f1f1;
    padding: 0.7rem 0.4rem;
  }
  .shop_img{
    @include wh(2.7rem, 2.7rem);
    display: block;
    margin-right: 0.4rem;
  }
  .list_back_li{
    height: 4.85rem;
    .list_back_svg{
      @include wh(100%, 100%)
    }
  }
  .shop_right{
    flex: auto;
    .shop_detail_header{
      @include fj;
      align-items: center;
      .shop_title{
        width: 8.5rem;
        color: #333;
        padding-top: .01rem;
        @include font(0.65rem, 0.65rem, 'PingFangSC-Regular');
        font-weight: 700;
      }
      .premium::before{
        content: '品牌';
        display: inline-block;
        font-size: 0.5rem;
        line-height: .6rem;
        color: #333;
        background-color: #ffd930;
        padding: 0 0.1rem;
        border-radius: 0.1rem;
        margin-right: 0.2rem;
      }
      .shop_detail_ul{
        display: flex;
        transform: scale(.8);
        margin-right: -0.3rem;
        .supports{
          @include sc(0.5rem, #999);
          border: 0.025rem solid #f1f1f1;
          padding: 0 0.04rem;
          border-radius: 0.08rem;
          margin-left: 0.05rem;
        }
      }
    }
    .rating_order_num{
      @include fj(space-between);
      height: 0.6rem;
      margin-top: 0.52rem;
      .rating_order_num_left{
        @include fj(flex-start);
        .rating_section{
          display: flex;
          .rating_num{
            @include sc(0.4rem, #ff6000);
            margin: 0 0.2rem;
          }
        }
        .order_section{
          transform: scale(.8);
          margin-left: -0.2rem;
          @include sc(0.4rem, #666);
        }
      }
      .rating_order_num_right{
        display: flex;
        align-items: center;
        transform: scale(.7);
        min-width: 5rem;
        justify-content: flex-end;
        margin-right: -0.8rem;
        .delivery_style{
          font-size: 0.4rem;
          padding: 0.04rem 0.08rem 0;
          border-radius: 0.08rem;
          margin-left: 0.08rem;
          border: 1px;
        }
        .delivery_left{
          color: #fff;
          background-color: $blue;
          border: 0.025rem solid $blue;
        }
        .delivery_right{
          color: $blue;
          border: 0.025rem solid $blue;
        }
      }
    }
    .fee_distance{
      margin-top: 0.52rem;
      @include fj;
      @include sc(0.5rem, #333);
      .fee{
        transform: scale(.9);
        @include sc(0.5rem, #666);
      }
      .distance_time{
        transform: scale(.9);
        span{
          color: #999;
        }
        .order_time{
          color: $blue;
        }
        .segmentation{
          color: #ccc;
        }
      }
    }
  }
  .loader_more{
    @include font(0.6rem, 3);
    text-align: center;
    color: #999;
  }
  .empty_data{
    @include sc(0.5rem, #666);
    text-align: center;
    line-height: 2rem;
  }
  .return_top{
    position: fixed;
    bottom: 3rem;
    right: 1rem;
    .back_top_svg{
      @include wh(2rem, 2rem);
    }
  }
  .loading-enter-active, .loading-leave-active {
    transition: opacity 1s
  }
  .loading-enter, .loading-leave-active {
    opacity: 0
  }
  .menu_detail_footer{
    margin-left: 2.4rem;
    font-size: 0;
    margin-top: .3rem;
    @include fj;
    .food_price{
      span{
        font-family: 'Helvetica Neue',Tahoma,Arial;
      }
      span:nth-of-type(1){
        @include sc(.5rem, #f60);
        margin-right: .05rem;
      }
      span:nth-of-type(2){
        @include sc(.7rem, #f60);
        font-weight: bold;
        margin-right: .3rem;
      }
      span:nth-of-type(3){
        @include sc(.5rem, #666);
      }
    }
  }
</style>
