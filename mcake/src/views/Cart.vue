<template>
  <div class="my-cart">
    <div class="cart-title">
      <img v-bind:src="require('..//assets/imgs/mycart.png')">
    </div>
    <div class="cart-top">
      <span class="first">商品信息</span>
      <span>规格</span>
      <span>数量</span>
      <span>单价</span>
      <span class="last">操作</span>
    </div>
    <div class="cart-list">
      <div class="cart-item" v-for="goods in cartList" :key="goods._id">
        <check-box @check="checkGoods($event,goods)" :isCheck="goods.isCheck"/>
        <div class="cart-img">
          <img v-lazy="goods.img" @click="toDetail(goods.goodsId)">
        </div>
        <div class="cart-name">
          <p class="name" @click="toDetail(goods.goodsId)">{{goods.goodsName}}</p>
        </div>
        <div class="cart-sku-list">
          <el-select v-model="goods.sku.weight" @change="chooseSku($event,goods)">
            <el-option
              v-for="(item,index) in goods.skuList"
              :key="index"
              :value="item.weight"
              :value-key='item.id'>
            </el-option>
          </el-select>
        </div>
        <div class="cart-num">
          <span class="jian" @click="num(false,goods)">-</span>
          <span class="number">{{goods.cartNum}}</span>
          <span class="jia" @click="num(true,goods)">+</span>
        </div>
        <div class="cart-money">
          <span>￥{{goods.sku.price}}</span>
        </div>
        <div class="cart-operate">
          <span class="del" @click="delGoods(goods)"></span>
        </div>
      </div>
    </div>
    <div class="cart-pay">
      <div class="pay-info">
        共计{{totalNum}}件商品
      </div>
      <div class="pay-money">
        合计：<span>￥{{totalPrice}}</span>
      </div>
    </div>
    <div class="cart-btn">
      <div class="pay-btn" @click="checkList.length==0?this.$message.warning('请选择要结算的商品'):addressBoxVisible = true">结算
      </div>
    </div>


    <!--  隐藏的弹窗 -->
    <div class="address-box" @click.stop="">
      <el-dialog :visible.sync="addressBoxVisible" width="400px">
        <div class="box-content" slot="">
          <div class="box-item" v-for="item in allAddress" :key="item._id">
            <z-radio @change="chooseAddress(item)" v-model="address._id" :label="item._id">
              <div class="address-info">
                收货人：{{item.name}}
                联系方式：{{item.phone}}
                <p>
                  收货地址：{{item.address}}
                </p>
              </div>
            </z-radio>
          </div>
          <div class="box-item">
            <el-input
              type="textarea"
              :rows="2"
              placeholder="备注"
              v-model="mark">
            </el-input>
          </div>
        </div>
        <div slot="footer" class="box-footer">
          <div @click="addressBoxVisible = false">取消</div>
          <div @click="order">确认</div>
        </div>
      </el-dialog>
    </div>
  </div>
</template>

<script>
  import checkBox from '../components/checkBox';
  import ZRadio from "../components/radio";
  import {mapActions, mapGetters} from 'vuex';

  export default {
    name: "Cart",
    components: {
      ZRadio,
      checkBox,
    },
    data: () => {
      return {
        cartList: [],
        checkList: [],//勾选的商品
        totalNum: 0,//共计商品
        totalPrice: 0,//总价
        addressBoxVisible: false,//
        address: {
          _id: 0,//避免报错
          name: "",
          phone: "",
          address: ""
        },//选择的收获地址
        mark: "",//备注
      }
    },
    computed: {
      ...mapGetters({
        allAddress: 'getAddress'
      }),
    },
    mounted() {
      this.getUser();
    },
    methods: {
      ...mapActions(['setOrder']),
      ///获取用户信息
      getUser() {
        this.$api.user.getUser()
          .then(res => {
            this.cartList = res.cartList;
            res.cartList.forEach((_) => {
              if (_.isCheck) {
                this.checkList.push(_);
                this.totalNum += _.cartNum;
                this.totalPrice += _.cartNum * Number.parseFloat(_.sku.price);
              }
            });
          }).catch(err => {
          console.log(err);
        })
      },
      //进入详情页
      toDetail(id) {
        this.$router.push({name: 'goodsDetail', params: {goodsId: id}})
      },
      //保存购物车数据
      saveCart(cartList, showSuccess = true) {
        this.$api.user.saveCart(cartList)
          .then(res => {
            if (res.status == 0) {
              if (showSuccess) {
                this.$message.success(res.msg);
              }
            } else {
              this.$message.error(res.msg);
            }
          }).catch(err => {
          this.$message.error(err.msg);
        });
      },

      // +:true -:false
      num(type, goods) {
        let isCommit = true;
        if (type) {
          goods.cartNum++;
          if (goods.isCheck) {//在购物车中被勾选
            this.totalNum++;
            this.totalPrice += Number.parseInt(goods.sku.price);
          }
        } else {
          if (goods.cartNum == 1) {
            isCommit = false;
          } else {
            goods.cartNum--;
            if (goods.isCheck) {//在购物车中被勾选
              this.totalNum--;
              this.totalPrice -= Number.parseInt(goods.sku.price);
            }
          }
        }

        if (isCommit) {
          this.saveCart(this.cartList, false);
        }

      },

      //修改sku
      chooseSku(weight, goods) {
        // console.log(`${weight} goods is ${JSON.stringify(goods)}`);
        let oldPrice = Number.parseFloat(goods.sku.price);
        //先修改本商品
        for (let item of goods.skuList) {
          if (weight == item.weight) {
            goods.sku = Object.assign({}, item);
            goods.sku.id = item._id;
            break;
          }
        }

        //修改总价 共计
        if (goods.isCheck) {
          let agio = goods.cartNum * (Number.parseFloat(goods.sku.price) - oldPrice);//差价
          // console.log(`勾选了 agio is ${agio}`);
          this.totalPrice += agio;
        }

        //修改购物车列表 重复的合并
        if (goods.isSame) {
          for (let item of this.cartList) {
            if (!item.isSame || item._id == goods._id) {
              // console.log('跳过不同商品和自己');
              continue;
            } else if (item.goodsId == goods.goodsId && goods.sku.id == item.sku.id) {
              item.cartNum += goods.cartNum;
              this.cartList.remove(goods);
              this.checkList.remove(goods);
              item.isSame = false;
              // break;
            } else if (item.goodsId == goods.goodsId) {
              // console.log('同商品');
              item.isSame = true;
            }
          }
        }
        //保存到数据库
        this.saveCart(this.cartList);

      },

      //删除商品
      delGoods(goods) {
        this.$confirm('是否删除此商品?', '提示', {
          confirmButtonText: '确定',
          cancelButtonText: '取消',
          type: 'warning'
        }).then(() => {
          this.cartList.remove(goods);
          this.saveCart(this.cartList);
          if (goods.isCheck) {//在购物车中被勾选
            this.checkList.remove(goods);
            this.totalNum -= goods.cartNum;
            this.totalPrice -= goods.cartNum * Number.parseFloat(goods.sku.price);
          }
        }).catch(() => {
          this.$message({
            type: 'info',
            message: '已取消删除'
          });
        });
      },

      //勾选商品
      checkGoods(state, goods) {
        if (state) {
          goods.isCheck = true;
          this.checkList.push(goods);
          this.totalNum += goods.cartNum;
          this.totalPrice += goods.cartNum * goods.sku.price;
        } else {
          goods.isCheck = false;
          this.checkList.remove(goods);
          this.totalNum -= goods.cartNum;
          this.totalPrice -= goods.cartNum * goods.sku.price;
        }
        this.saveCart(this.cartList, false);
      },

      chooseAddress(address) {
        // console.log(`address is ${address._id} ${address}`);
        this.address = Object.assign({}, address);
      },
      //结算
      order() {
        let order = {
          "list": this.checkList,//订单内容
          "totalPrice": this.totalPrice,//订单总金额
          "totalNum": this.totalNum,//订单总件数
          "address": this.address,//收货地址
          "mark": this.mark,//收货地址
        };
        // console.log(`order is ${JSON.stringify(order)}`);
        this.$api.user.order(order)
          .then(res => {
            if(res.status==0){
              this.checkList.forEach((_)=>{
                this.cartList.remove(_);
              });
              this.saveCart(this.cartList,false);///备注：缺少对isSame的判断（可能不再使用isSame属性）
              this.$message.success(res.msg);
              this.setOrder(res.data);
              this.$router.push('/pay');
            }
          }).catch(err => {
          this.$message.error(err.msg);
        })
      }
    }
  }
</script>

<style lang="less">
  .my-cart {
    .cart-top {
      height: 44px;
      line-height: 44px;
      margin-top: 58px;
      margin-bottom: 12px;
      background-color: white;

      span {
        color: #000;
        font-size: 15px;
        margin-right: 128px;
      }

      span:nth-child(2) {
        margin-right: 160px;
      }

      span:nth-child(4) {
        margin-right: 118px;
      }

      .first {
        margin-left: 135px;
        margin-right: 340px;
      }

      .last {
        margin-right: 0px;
      }
    }

    .cart-list {
      margin-bottom: 26px;

      .cart-item {
        display: flex;
        align-items: center;
        padding-top: 36px;
        padding-bottom: 18px;
        border-bottom: 1px solid #e8e8e8;

        .cart-img {
          margin: 0 12px;

          img {
            width: 135px;
            height: 135px;
            cursor: pointer;
          }
        }

        .cart-name {
          margin-right: 30px;
          width: 220px;

          .name {
            font-weight: bold;
            font-size: 16px;
            color: #000;
            margin-bottom: 2px;
            cursor: pointer;
          }
        }

        .cart-sku-list {
          padding-left: 16px;
          padding-right: 24px;
          color: #a2a2a2;
          font-size: 16px;
          height: 50px;
          line-height: 50px;

          /deep/ input.el-input__inner {
            height: 30px;
            font-size: 16px;
            border: none;
            color: #888;
            text-align: center;
            outline: none;
            appearance: none;
          }
        }

        .cart-num {
          display: flex;
          align-items: center;
          width: 150px;

          .number {
            width: 30px;
            text-align: center;
            border: none;
            outline: none;
            /*margin: 0px 10px 0px;*/
          }

          span:first-child, span:last-child {
            width: 25px;
            height: 25px;
            font-size: 20px;
            display: inline-block;
            text-align: center;
            line-height: 25px;
            background: #f0f0f0;
            border-radius: 50%;
            cursor: pointer;
            user-select: none;
          }

          span:first-child:hover, span:last-child:hover {
            background: #ffe32a;
          }
        }

        .cart-money {
          min-width: 130px;

          span {
            font-size: 16px;
            font-weight: bold;
            color: #000;
          }

        }

        .cart-operate {
          text-align: right;
          margin-left: 42px;

          .del {
            display: inline-block;
            width: 24px;
            height: 24px;
            background: url(../assets/imgs/icon41.png) no-repeat center;
            cursor: pointer;
          }

          .del:hover {
            background: url(../assets/imgs/icon42.png) no-repeat center;
          }
        }
      }
    }

    .cart-pay {
      padding-left: 520px;
      font-size: 14px;
      display: flex;
      align-items: center;
      justify-content: space-between;

      .pay-info {
        font-weight: bold;
        color: #000;
      }

      .pay-money {
        > span {
          font-size: 26px;
          font-weight: bold;
          color: #000;
        }
      }
    }

    .cart-btn {
      display: flex;
      justify-content: flex-end;
      margin-top: 58px;

      .pay-btn {
        cursor: pointer;
        border: none;
        outline: none;
        font-size: 15px;
        color: #000;
        font-weight: bold;
        width: 150px;
        height: 44px;
        text-align: center;
        line-height: 44px;
        background: url(../assets/imgs/icon12.png) no-repeat center;
      }
    }


    .address-box {
      /deep/ .el-dialog__body {
        padding: 0;
      }

      /deep/ .el-dialog__footer {
        padding: 0;
      }

      .box-content {
        padding: 24px;

        .box-item {
          display: flex;
          align-items: center;
          margin-top: 8px;
          margin-bottom: 18px;

          .address-info {
            margin-left: 26px;
          }
        }

        .box-item:last-child {
          margin-bottom: 0px;
        }
      }

      .box-footer {
        display: flex;

        div {
          width: 50%;
          height: 50px;
          font-size: 20px;
          text-align: center;
          background-color: #ffe32a;
          line-height: 50px;
          cursor: pointer;
        }

        div:nth-child(1) {
          background-color: #888;
          color: #fff;
        }
      }
    }
  }
</style>
