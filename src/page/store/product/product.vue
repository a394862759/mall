<template>
  <div class="store_list_page">
    <div class="store_list_wrap">
      <div class="list_left">
        <scroll class="scroll">
          <ul>
            <li :class="{'active':calcIndex==index}" v-for="(item,index) in data" :key="index" @click="scrollToClass(index)">{{item.title}}</li>
          </ul>
        </scroll>
      </div>
      <div class="list_right">
        <scroll class="scroll" ref="rightScroll" :listenScroll="true" :probeType="3" @scroll="rightScroll">
          <ul class="outer_list" ref="productList">
            <li class="height_hook" v-for="(item,index) in data" :key="index">
              <div class="title">{{item.title}}
                <span>{{item.introduction}}</span>
              </div>
              <ul>
                <li v-for="(product,index2) in item.products" :key="index2">
                  <div class="img_wrap">
                    <img v-lazy="product.imgUrl" alt="">
                    <div v-if="product.is_new" class="new">新品</div>
                    <div v-if="product.is_sign" class="sign">招牌</div>
                  </div>
                  <div class="product_info">
                    <h4>{{product.name}}</h4>
                    <p class="introduction">{{product.introduction}}</p>
                    <div class="sales">
                      月售{{product.sale_month}} {{product.like_?'好评 '+product.like_*100+'%':''}}
                    </div>
                    <div class="discount" v-if="product.is_discount">
                      <span>{{product.discount}}折</span>
                      限量抢购
                    </div>
                    <div class="recommend" v-if="product.is_recommend">老板推荐</div>
                    <div class="price_box">
                      <div class="price">
                        ￥
                        <strong>{{product.is_discount?product.real_price:product.price[0].price}}</strong>
                        <span v-if="product.is_discount">￥{{product.price[0].price}}</span>
                        <em v-if="product.price.length>1">起</em>
                      </div>
                      <operating @ballDown="ballDown" :food="product"></operating>
                    </div>
                  </div>
                </li>
              </ul>
            </li>
          </ul>
        </scroll>
      </div>
    </div>
    <div class="shopCart_wrap">
      <cart ref="cart" :store_msg="store_msg" :foods="cartFoods" @cartChange="saveCart"></cart>
    </div>

  </div>
</template>

<script>
import scroll from "@/components/scroll";
import cart from "./shopCart";
import operating from "./cartOperating";
import { getStorage, setStorage } from "@/script/storage";
import { CartCtrl } from "@/api/store/cart";
import { mapState } from "vuex";
import { Toast, Indicator } from "mint-ui";
export default {
  data() {
    return {
      data: [],
      active_index: 0,
      scrollY: 0,
      height_arr: [],
      cart: getStorage("cart") || {},
      ctrl: null
    };
  },
  methods: {
    rightScroll(pos) {
      this.scrollY = Math.abs(Math.round(pos.y));
      this.$emit("scroll", pos);
    },
    scrollToClass(index) {
      this.$refs.rightScroll.refresh();
      var el = this.$refs.productList.querySelectorAll("li.height_hook")[index];
      this.$refs.rightScroll.scrollToElement(el, 500);
    },
    ballDown(el) {
      this.$refs.cart.ballDown(el);
    },
    saveCart(obj) {
      var id = this.$route.query.id;
      this.cart[id] = {};
      this.cart[id].data = this.cartFoods;
      this.cart[id].proto = obj.proto;
      this.cart[id].total = obj.total;
      setStorage("cart", this.cart);
    },
    init_() {
      this.ctrl
        .getGoodsAndCart()
        .then(
          this.$http.spread((d, cart) => {
            var d = d.data;
            var cart = cart.data;
            if (d.status == 1 && cart.status == 1) {
              this.data = d.data.map(item => {
                item.products.forEach(item2 => {
                  if (
                    Boolean(
                      cart.data[this.$route.query.id] &&
                        cart.data[this.$route.query.id][item2.category] &&
                        cart.data[this.$route.query.id][item2.category][
                          item2.product_id
                        ]
                    )
                  ) {
                    item2.cartCount =
                      cart.data[this.$route.query.id][item2.category][
                        item2.product_id
                      ] || 0;
                  } else {
                    item2.cartCount = 0;
                  }
                });
                return item;
              });
            } else if (d.status == 1) {
              this.data = d.data.map(item => {
                item.products.forEach(item2 => {
                  item2.cartCount = 0;
                });
                return item;
              });
            }
            this.$nextTick(() => {
              var Ali = this.$refs.productList.querySelectorAll(".height_hook");
              var h = 0;
              this.height_arr.push(h);
              for (let i = 0; i < Ali.length; i++) {
                h += Ali[i].offsetHeight;
                this.height_arr.push(h);
              }
            });
            //alert(JSON.stringify(cart))
            Indicator.close();
          })
        )
        .catch(e => {
          Indicator.close();
          alert(e);
        });
    }
  },
  computed: {
    calcIndex() {
      var arr = this.height_arr,
        scrollY = this.scrollY;

      for (var i = 0; i < arr.length; i++) {
        if (scrollY >= arr[i] && scrollY < arr[i + 1]) {
          return i;
        }
      }
    },
    cartFoods() {
      let foods = [];
      this.data.forEach(item => {
        item.products.forEach(food => {
          if (food.cartCount) {
            foods.push(food);
          }
        });
      });
      return foods;
    },
    store_msg() {
      return this.$store.state.store.store_;
    },
    ...mapState({
      store_: state => {
        return state.store.store_;
      }
    })
  },
  created() {
    this.ctrl = new CartCtrl();
    this.init_();
  },
  mounted() {
    /* this.data = json.products_class.map(item => {
      item.products.forEach(item2 => {
        item2.cartCount = 0;
      });
      return item;
    });
 */
  },
  components: {
    scroll,
    cart,
    operating
  }
};
</script>

<style scoped lang='less'>
@import "~@/style/base.less";
.store_list_page {
  height: 100%;
  flex: auto;
  position: relative;
  .scroll {
    height: 100%;
    overflow: hidden;
  }
  .store_list_wrap {
    position: absolute;
    top: 0;
    left: 0;
    width: 100%;
    bottom: 143 / @r;
    display: flex;
    .list_left {
      flex: none;
      height: 100%;
      background-color: #f8f8f8;
      width: 240 / @r;
      ul {
        li {
          height: 145 / @r;
          background-color: #f8f8f8;
          color: #666;
          font-size: 38 / @r;
          line-height: 140 / @r;
          text-align: center;
          &.active {
            background-color: #fff;
            color: #333;
          }
        }
      }
    }
    .list_right {
      height: 100%;
      background-color: #fff;
      flex: auto;
      padding: 0 37 / @r;
      .outer_list {
        width: 100%;
        > li {
          &:nth-last-of-type(1) {
            padding-bottom: 88 / @r;
          }
          .title {
            height: 140 / @r;
            font-size: 38 / @r;
            color: #666;
            line-height: 140 / @r;
            span {
              color: #999;
              font-size: 30 / @r;
            }
          }
          ul {
            > li {
              padding-bottom: 88 / @r;
              display: flex;
              &:nth-last-of-type(1) {
                padding-bottom: 0;
              }
              .img_wrap {
                flex: none;
                width: 283 / @r;
                height: 283 / @r;
                border: 1px solid #eee;
                margin-right: 36 / @r;
                position: relative;
                .new,
                .sign {
                  width: 72 / @r;
                  height: 40 / @r;
                  background: linear-gradient(to right, #78d654, #50cb48);
                  color: #fff;
                  line-height: 40 / @r;
                  font-size: 25 / @r;
                  position: absolute;
                  top: -1px;
                  left: -1px;
                  text-align: center;
                }
                .sign {
                  background: linear-gradient(to right, #ff9d6c, #ff6445);
                }
                img {
                  height: 100%;
                  width: 100%;
                }
              }
              .product_info {
                width: 590 / @r;
                h4 {
                  font-size: 44 / @r;
                  line-height: 54 / @r;
                  color: #333;
                  padding: 6 / @r 0 36 / @r;
                }
                .introduction {
                  overflow: hidden;
                  white-space: nowrap;
                  text-overflow: ellipsis;
                  font-size: 28 / @r;
                  max-width: 19em;
                  color: #999;
                  padding-bottom: 34 / @r;
                }
                .sales {
                  font-size: 28 / @r;
                  color: #999;
                  padding-bottom: 34 / @r;
                }
                .discount {
                  font-size: 28 / @r;
                  color: #eb6551;
                  padding-bottom: 26 / @r;
                  span {
                    padding: 6 / @r 10 / @r;
                    border: 1px solid #f9d0ca;
                    font-size: 28 / @r;
                    color: #eb6551;
                    border-radius: 4 / @r;
                    margin-right: 10 / @r;
                  }
                }
                .recommend {
                  padding: 6 / @r;
                  border: 1px solid #f9d8ba;
                  color: #ed7e1b;
                  font-size: 28 / @r;
                  margin-bottom: 24 / @r;
                  width: 5em;
                  text-align: center;
                  border-radius: 4 / @r;
                }
                .price_box {
                  display: flex;
                  justify-content: space-between;
                  align-items: center;
                  width: 100%;
                  .price {
                    font-size: 28 / @r;
                    color: #ff5339;
                    font-weight: bold;
                    strong {
                      font-size: 40 / @r;
                    }
                    span {
                      color: #999;
                      text-decoration: solid line-through #999;
                    }
                  }
                }
              }
            }
          }
        }
      }
    }
  }
  .shopCart_wrap {
    position: absolute;
    bottom: 0;
    left: 0;
    width: 100%;
    height: 143 / @r;
  }
}
</style>