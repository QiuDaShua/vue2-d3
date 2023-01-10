<template>
  <div id="structruepage">
    <custom-nav-bar
      :title="title"
      left-arrow
      @on-clickleft="onClickLeft">
    </custom-nav-bar>
    <van-tabs
      shrink
      line-height="0.8vw"
      line-width="19.2vw"
      :lazy-render="false"
      @change="changeTab">
      <van-tab
        title="股东持股"
        name="up"
        title-class="title-class">
        <RightStructureIn
          :tree="treeUp"
          :token="token"
          key="up"></RightStructureIn>
      </van-tab>
      <van-tab
        title="对外投资"
        name="down"
        title-class="title-class">
        <RightStructureOut
          :tree="treeDown"
          :token="token"
          key="up"></RightStructureOut>
      </van-tab>
    </van-tabs>
  </div>
</template>

<script lang="ts">
import { defineComponent, onMounted, ref } from 'vue'
import  CustomNavBar from '@/components/common/CustomNavbar.vue'
import RightStructureIn from '@/components/companySearch/RightStructureIn.vue'
import RightStructureOut from '@/components/companySearch/RightStructureOut.vue'
import { useRoute } from 'vue-router'
import { useStore } from 'vuex'
import { Notify, Toast } from 'vant'
import { fetchEquityUpperInfo, fetchEquityBelowInfo, fetchCompanySearchDetail } from '@/api/companySearch'
import { formatMoney } from '@/utils/tool'
import { sm2Decrypted } from '@/enrich/crypto-gm'
import { GlobalMutation } from '@/store/types/mutation-types'

export default defineComponent({
  name: 'RightStructure',

  components: {
    CustomNavBar,
    RightStructureIn,
    RightStructureOut
  },
  setup() {
    const store = useStore()
    const route = useRoute()
    const title = ref('股权结构图')

    // 点击回退
    const onClickLeft =()=>{
        // jsBridge.callHandler('navigationToSkip', {type: '0'}, ()=> {
        //   console.log('11111111111')
        // })
        history.back()
    }

    const data = route.query.data ? JSON.parse(sm2Decrypted(route.query.data)) : {}
    const id = data.id ? data.id : '' // 企业id
    const token = data.token ? data.token : store.state.global.token // 请求token
    store.commit(`global/${GlobalMutation.SET_TOKEN}`, token)
    const userid = data.userid ? data.userid :''
    const regCapi = ref('') // 企业持缴金额
    const name = ref('') // 企业名称
    const parents = ref<any[]>([])
    const children = ref<any[]>([])
    const treeUp = ref({})
    const treeDown = ref({})

    onMounted(()=>{
      Toast.loading({
        message: '加载中',
        forbidClick: true,
        duration: 0,
      });
      getInit()
    })

    const getInit = async () =>{
        await getDetailInfo()
        // await getUpper()
        // await getBelow()
        // getTreeData()
        Promise.all([getUpper(), getBelow()]).finally(()=>{
          getTreeData()
        })
    }

    const getDetailInfo = async () =>{
      await fetchCompanySearchDetail({
          token: token,
          instId: id,
          userId: userid
      }).then((response)=>{
          const {code =0, records = [] } = response
          if (code > 0 && records != null) {
            regCapi.value = records[0].reg_capi
            name.value = records[0].chn_full_nm
          }
        })
    }

    const getUpper = async () =>{
        await fetchEquityUpperInfo({
          token: token,
          instId: id,
          regCapi: regCapi.value,
          currentPage: 0,
          pageSize: 200,
        }).then((response) => {
          const {code =0, records = [] } = response
          if (code > 0 && records != null) {
            const dataSource = [] as any[]
            records.forEach(element =>{
              // let children = [] as any[]
              // // 设置children节点
              // if(element.list){
              //   element.list.forEach(child =>{
              //     children.push({
              //       money: child.amount ? formatMoney((child.amount / 10000).toFixed(2)) :'--',
              //       scale: child.hold_rati || '--%',
              //       name: child.chn_full_nm || '--',
              //       id: child.inst_cust_id || '--',
              //       type: '0'
              //     })
              //   })
              // }
            
              dataSource.push({
                // children: children,
                isHaveChildren: element.dataType === '1' ? true : false,
                money: element.amount ? formatMoney((element.amount / 10000).toFixed(2)) : '--',
                scale: element.hold_rati || '--%',
                name: element.chn_full_nm || '--',
                id: element.inst_cust_id || '--',
                type: '0',
                regCapi: element.reg_capi
              })
            })
            parents.value = dataSource
          }
        })
    }

    const getBelow = async () =>{
        await fetchEquityBelowInfo({
          token: token,
          instId: id,
          currentPage: 0,
          pageSize: 200,
        }).then((response) => {
          const {code =0, records = []} = response
          if (code > 0 && records != null) {
            const dataSource = [] as any[];
            records.forEach(element =>{
            // let children = [] as any[]
            // // 设置children节点
            // if(element.list){
            //   element.list.forEach(child =>{
            //     children.push({
            //       money: child.amount ? formatMoney((child.amount / 10000).toFixed(2)) :'--',
            //       scale: child.hold_rati || '--%',
            //       name: child.chn_full_nm || '--',
            //       id: child.inst_cust_id || '--',
            //       type: '0'
            //     })
            //   })
            // }
          
            dataSource.push({
              // children: children,
              isHaveChildren: element.dataType === '1' ? true : false,
              money: element.amount ? formatMoney((element.amount / 10000).toFixed(2)) :'--',
              scale: element.hold_rati || '--%',
              name: element.chn_full_nm || '--',
              id: element.inst_cust_id || '--',
              type: '0',
            })
            })
            children.value = dataSource
          }
        })
    }

    
    // 获取树状数据
    const getTreeData = () =>{
      let upobj = {
        id: id,
        name: name.value,
        tap: '节点',
        children: parents.value,
      }
      treeUp.value = {...upobj}
      let downobj = {
        id: id,
        name: name.value,
        tap: '节点',
        children: children.value,
      }
      treeDown.value = {...downobj}
      Toast.clear()
    }

    // 点击tab
    const changeTab = (value) =>{
      console.log(value)
      // if(value === 'up'){
      //   let obj = {
      //     id: id,
      //     name: name,
      //     tap: '节点',
      //     children: parents.value,
      //   }
      //   treeUp.value = {...obj}
      // }else {
      //   let obj = {
      //     id: id,
      //     name: name,
      //     tap: '节点',
      //     children: children.value,
      //   }
      //   treeDown.value = {...obj}
      // }
    }

    return {
      token,
      title,
      treeUp,
      treeDown,
      onClickLeft,
      changeTab 
    }
  }
})

</script>

<style lang="scss" scoped>
::v-deep .van-tabs__wrap{
  height: 45px !important;
  text-align: left;
  background: #fff;
  border-bottom: 1px solid rgb(241, 238, 238);
  .van-tab--line {
    padding: 0;
  }
}
::v-deep .title-class {
  font-size: 16px;
  color:#999999;
  padding:0 16px 0 16px
}
::v-deep .van-tab {
  width:96px
}
::v-deep .van-tabs__nav{
  margin-left:6px;
  padding: 0;
  height: 100%;
}
::v-deep .van-tabs__line{
  background-color: #DE4A3C;
  border-radius: 0px;
  bottom: 4px;
}
::v-deep .van-tab--active {
  color:#DE4A3C;
  font-size:18px;
}
</style>

