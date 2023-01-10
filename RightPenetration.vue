<template>
  <div class="father-box">
    <div
      id="rightPenetrationpage" 
      :style="{ 'transition': 'transform .5s ease', '-ms-transition': 'transform .5s ease', '-moz-transition': 'transform .5s ease','-webkit-transition': 'transform .5s ease','-o-transition': 'transform .5s ease'}">
      <custom-nav-bar
        :title="title"
        left-arrow
        @on-clickleft="onClickLeft">
      </custom-nav-bar>
      <!-- <div
        class="full"
        @click.stop="showFullScreen">
        <div class="full-icon"></div>
        <span>{{isFull ? '退出全屏' :'全屏'}}</span>
      </div> -->
      <div
        id="penetrateChart"
        :style="{width:'100%',display:'block',margin:'auto'}"
      >
      </div>
    </div>
  </div>

</template>


<script lang="ts">
  import { defineComponent} from 'vue'
  import { useStore } from 'vuex'
  import  CustomNavBar from '@/components/common/CustomNavbar.vue'
  import { fetchCompanySearchDetail, fetchEquityUpperInfo, fetchEquityBelowInfo } from '@/api/companySearch'
  import { Notify, Toast } from 'vant'
  import { formatMoney, getBLen } from '@/utils/tool'
  import { sm2Decrypted } from '@/enrich/crypto-gm'
  import { GlobalMutation } from '@/store/types/mutation-types'
  import * as $d3 from 'd3'
  // 过渡时间
  const DURATION = 0
  // 加减符号半径
  const SYMBOLA_S_R = 9
  // 公司
  const COMPANY = '0'
  // 人
  const PERSON = '1'
  //x,y距离
  // let x0 = 0, y0 = 0, dx = 0, dy = 0
  export default defineComponent({
    props: {},
    components: {
      CustomNavBar
    },

    data () {
      return {
        layoutTree: {} as any,
        diamonds: {} as any,
        d3: $d3,
        // hasChildNodeArr: [],
        originDiamonds: {} as any,
        diagonalUp: '',
        diagonalDown: '',
        tree: {} as any,
        rootUp: {} as any,
        rootDown: {} as any,
        svg: {} as any,
        svgW: document.documentElement.clientWidth,
        svgH: document.documentElement.clientHeight - 44,
        title: '股权穿透图',
        isFull: false,
        name: '',
        id: '',
        token: '',
        regCapi: '',
        userid: '',
        parents: [] as any[], // 下游信息
        children: [] as any[], // 上游信息
      }
    },

    // beforeCreate () {
    //   document.body.style.overflow = 'hidden'
    // },

    // beforeDestroy () {
    //   document.body.style.overflow = 'auto'
    // },

    // created () {
      
    //   // window.addEventListener('orientationchange', this.changeOrient)
    // },

  mounted () {
      const store = useStore()
      const data = this.$route.query.data ? JSON.parse(sm2Decrypted(this.$route.query.data)) : {}
      const id = data.id
      const token = data.token ? data.token : store.state.global.token
      const userid = data.userid
      this.id = id
      this.token = token
      this.userid = userid
      store.commit(`global/${GlobalMutation.SET_TOKEN}`, token)

      Toast.loading({
        message: '加载中',
        forbidClick: true,
        duration: 0,
      });
      
      this.getInit()
    },
    beforeUnmount() {
      this.d3.select('#treesvg').remove()
      console.log('页面关闭')
    },

    methods: {
      // changeOrient () {
      //   const box = document.getElementById('penetrateChart').children[0]
      //   const g = document.getElementById('penetrateChart').children[0].children[0]
      //   let navbar = document.querySelector('.navbar')
      //   let flag = false
      //   flag = isOrient()
      //   setTimeout(()=>{
      //     if(flag){
      //       navbar?.classList.add('smallBar')
      //     }else{
      //       navbar?.classList.remove('smallBar')
      //     }
      //     console.log(document.documentElement.clientWidth, document.documentElement.clientHeight)
      //     box.setAttribute('width', document.documentElement.clientWidth)
      //     box.setAttribute('height', document.documentElement.clientHeight)
      //     g.setAttribute('transform', 'translate(' + (document.documentElement.clientWidth / 2) + ',' + (document.documentElement.clientHeight / 2) + ')')
      //   }, 100)
      // },
      async getDetailInfo(){
        await fetchCompanySearchDetail({
          token: this.token,
          instId: this.id,
          userId: this.userid
        }).then((response)=>{
          const {code =0, records = [] } = response
          if (code > 0 && records != null) {
            this.regCapi = records[0].reg_capi
            this.name = records[0].chn_full_nm
          }
        })
      },
      async getUpper(){
        await fetchEquityUpperInfo({
          token: this.token,
          instId: this.id,
          regCapi: this.regCapi,
          currentPage: 0,
          pageSize: 200,
        }).then((response) => {
          const {code =0, records = [] } = response
          if (code > 0 && records != null) {
            const dataSource = [] as any[];
            records.forEach(element =>{
              // let children = []
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
            this.parents = dataSource
          }
        })
      },
      async getBelow(){
        await fetchEquityBelowInfo({
          token: this.token,
          instId: this.id,
          currentPage: 0,
          pageSize: 200,
        }).then((response) => {
          const {code =0, records = []} = response
          if (code > 0 && records != null) {
            const dataSource = [] as any[];
            records.forEach(element =>{
            // let children = []
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
            this.children = dataSource
          }
        })
      },

      // 获取树状数据
      getTreeData(){
        console.log( this.children, this.parents, '111111111')
        let obj = {
          id: this.id,
          name: this.name,
          tap: '节点',
          children: this.children,
          parents: this.parents,
        }
        this.tree = {...obj}
        Toast.clear()
      },

      async getInit(){
        await this.getDetailInfo()
        // await this.getUpper()
        // await this.getBelow()
        Promise.all([this.getUpper(), this.getBelow()]).finally(()=>{
          this.getTreeData()
          this.init()
        })
      },

      init () {
        let d3 = this.d3
        let svgW = this.svgW
        let svgH = this.svgH
        // x0 = svgW / 2,
        // y0=  svgH / 2
        // 方块形状
        this.diamonds = {
          w: 162,
          h: 66,
          intervalW: 182,
          intervalH: 150
        }
        // 源头对象
        this.originDiamonds = {
          w: 208,
          h: 41
        }
        this.layoutTree = d3.tree().nodeSize([this.diamonds.intervalW, this.diamonds.intervalH]).separation(() => 1);
        // 主图
        this.svg = d3.select('#penetrateChart').append('svg').attr('width', svgW).attr('height', svgH).attr('id', 'treesvg')
          .call(d3.zoom().scaleExtent([0.3, 2]).on('zoom', () => {
            // 设置缩放位置以及平移初始位
            // if(isiOS && this.isFull){
            //   // 修改ios手机上才有的移动bug，安卓手机，pc端没有
            //   let x = d3.event.transform.x
            //   d3.event.transform.x = d3.event.transform.y
            //   d3.event.transform.y = -x
            //   console.log('222', '出现移动bug', d3.event.transform)
            //   console.log(isiOS, d3.event.transform.x, d3.event.transform.y)
            //   // dx = d3.event.transform.x - x0
            //   // dy = d3.event.transform.y - y0
            //   // x0 = d3.event.transform.x
            //   // y0 = d3.event.transform.y
            //   // d3.event.transform.x = d3.event.transform.x + dy
            //   // d3.event.transform.y = d3.event.transform.y + dx
            //   // this.svg.attr('transform', 'translate(' + (svgW / 2) + ',' + (svgH / 2) + ') rotate(90)')
            // }
            this.svg.attr('transform', d3.event.transform.translate(svgW / 2, svgH / 2));
          }))
          .on('dblclick.zoom', null)
          .attr('style', 'position: relative;z-index: 2') //background-image:url(${setWatermark({name: this.$store.state.global.user.userName, loginName: this.$store.state.global.user.userId}).toDataURL()})
          .append('g').attr('id', 'g').attr('transform', `translate(${svgW / 2},${svgH / 2})`)       

        let upTree = {} as any
        let downTree = {} as any
        // 拷贝树的数据
        Object.keys(this.tree).map(item => {
          if (item === 'parents') {
            upTree = JSON.parse(JSON.stringify(this.tree))
            upTree.children = this.tree[item]
            upTree.parents = null
          } else if (item === 'children') {
            downTree = JSON.parse(JSON.stringify(this.tree))
            downTree.children = this.tree[item]
            downTree.parents = null
          }
        })
                // hierarchy 返回新的结构 x0,y0初始化起点坐标
        this.rootUp = d3.hierarchy(upTree, d => d.children);
        this.rootUp.x0 = 0
        this.rootUp.y0 = 0

        this.rootDown = d3.hierarchy(downTree, d => d.children);
        this.rootDown.x0 = 0
        this.rootDown.y0 = 0;
        // 上 和 下 结构
        let treeArr = [
          {
            data: this.rootUp,
            type: 'up'
          },
          {
            data: this.rootDown,
            type: 'down'
          }
        ]
        if(!this.tree['children'].length && !this.tree['parents'].length){
          this.updataSelf()
        }else{
          treeArr.map(item => {
          if (item.data.children) {
            // item.data.children.forEach(this.collapse);
            this.update(item.data, item.type, item.data)
          }
          })
        }
      },

      updataSelf(){
        let nodes = this.rootUp.descendants()
        let node = this.svg.selectAll('g.node')
          .data(nodes, d => d.data.id || '');
        let nodeEnter = node.enter().append('g')
          .attr('class', d => 'node node_' + d.depth) //d => showtype     === 'up' && !d.depth ? 'hide-node' :
          // .attr('transform', 'translate(' + (svgW / 2) + ',' +     (svgH / 2) + ')')
          .attr('opacity', 1); // 拥有下部分则隐藏初始块  d => showtype     === 'up' && !d.depth ? (this.rootDown.data.children.   length ? 0 : 1) : 1
        // 创建矩形
        nodeEnter.append('rect')
          .attr('type', d => d.data.id + '_' + d.depth)
          .attr('width', d => d.depth ? this.diamonds.w : (getBLen(d.data.name)/2 * 20 + 20))
          .attr('height', d => d.depth ? (d.data.type === COMPANY ?  this.diamonds.h : this.diamonds.h - 10) :  this.originDiamonds.h)
          .attr('x', d => d.depth ? -this.diamonds.w / 2 : -(getBLen(d.data.name)/2 * 20 + 20) / 2)
          .attr('y', d => d.depth ?  0 : -15)
          .attr('stroke', '#DE4A3C')
          .attr('stroke-width', 1)
          .attr('rx', 10)
          .attr('ry', 10)
          .style('fill', d => {
            if (d.data.type === COMPANY || !d.depth) {
              return d.depth ? '#fff' : '#DE4A3C'
            } else if (d.data.type === PERSON) {
              return '#fff'
            }
          });
          // 文字
          nodeEnter.append('text')
            .attr('x', 0)
            .attr('y', 0)
            .attr('dy', `${this.originDiamonds.h/2 - 10}px`)
            .attr('text-anchor', 'middle')
            .attr('fill', d => d.depth ? '#DE4A3C' : '#fff')
            .text(d => d.data.name)
            .style('font-size', d => d.depth ? '16px' : '20px')
            .style('font-family', 'PingFangSC-Medium')
            .style('font-weight', '500')
      },

      /*
       *[update 函数描述], [click 函数描述]
       *  @param  {[Object]} source 第一次是初始源对象，后面是点击的对象
       *  @param  {[String]} showtype up表示向上 down表示向下
       *  @param  {[Object]} sourceTree 初始源对象
       */
      update (source, showtype, sourceTree) {
        // eslint-disable-next-line
        let _this = this
        if (source.parents === null) {
          source.isOpen = !source.isOpen
        }
        let nodes
        if (showtype === 'up') {
          nodes = this.layoutTree(this.rootUp).descendants()
        } else {
          nodes = this.layoutTree(this.rootDown).descendants()
        }
        let links = nodes.slice(1);
        nodes.forEach(d => {
          d.y = d.depth *(d.depth == 1 ? 120 : this.diamonds.intervalH);
        });

        let node = this.svg.selectAll('g.node' + showtype)
          .data(nodes, d => d.data.id || '');

        let nodeEnter = node.enter().append('g')
          .attr('class', d => showtype === 'up' && !d.depth ? 'hide-node' : 'node' + showtype)
          .attr('transform', d => showtype === 'up' ? 'translate(' + d.x + ',' + -(d.y) + ')' : 'translate(' + d.x + ',' + d.y + ')')
          .attr('opacity', d => showtype === 'up' && !d.depth ? (this.rootDown.data.children.length ? 0 : 1) : 1); // 拥有下部分则隐藏初始块
        // 创建矩形
        nodeEnter.append('rect')
          .attr('type', d => d.data.id)
          .attr('width', d => d.depth ? this.diamonds.w : (getBLen(d.data.name)/2 * 20 + 20))
          .attr('height', d => d.depth ? (d.data.type === COMPANY ? this.diamonds.h : this.diamonds.h - 10) : this.originDiamonds.h)
          .attr('x', d => d.depth ? -this.diamonds.w / 2 : -(getBLen(d.data.name)/2 * 20 + 20) / 2)
          .attr('y', d => d.depth ? showtype === 'up' ? -this.diamonds.h / 2 : 0 : -15)
          .attr('stroke', d => d.data.type === COMPANY || !d.depth ? '#DE4A3C' : '#7A9EFF')
          .attr('stroke-width', 1)
          .attr('rx', 10)
          .attr('ry', 10)
          .style('fill', d => {
            if (d.data.type === COMPANY || !d.depth) {
              return d.depth ? '#fff' : '#DE4A3C'
            } else if (d.data.type === PERSON) {
              return '#fff'
            }
          });

        // 创建圆 加减
        let circle = nodeEnter.append('g')
            .attr('class', 'circle')
            .on('click', function (d) {
            _this.click(d, showtype, sourceTree)
        });

        circle.append('circle')
          .attr('type', d => d.data.id || '')
          .attr('r', (d) => d.depth ? (d.data.isHaveChildren ? SYMBOLA_S_R : 0) : 0)
          .attr('cy', d => d.depth ? showtype === 'up' ? -(SYMBOLA_S_R + this.diamonds.h / 2) : (this.diamonds.h + SYMBOLA_S_R) : 0)
          .attr('cx', 0)
          .attr('fill', '#F9DDD9')
          .attr('stroke', '#FCEDEB')
          .style('stroke-width', 1)
          
        circle.append('text')
          .attr('x', 0)
          .attr('dy', d => d.depth ? (showtype === 'up' ? -(SYMBOLA_S_R / 2 + this.diamonds.h / 2) : this.diamonds.h + SYMBOLA_S_R + 4) : 0)
          .attr('text-anchor', 'middle')
          .attr('class', 'fa')
          .style('fill', '#DE4A3C')
          .text(function(d) {
            if(d.depth){
              if (d.children) {
                return '-';
              } else if (d._children || d.data.isHaveChildren) {
                return '+';
              } else {
                return '';
              }
            }else {
                return '';
            }
          })
          .style('font-size', '16px');
        
        node.select('.fa')
        .text(function (d) {
          if (d.children) {
            return '-';
          } else if (d._children || d.data.isHaveChildren) {
            return '+';
          } else {
            return '';
          }
        })

        // 持股比例
        nodeEnter.append('g')
          .attr('transform', () => 'translate(0,0)')
          .append('text')
          .attr('x', 35)
          .attr('y', showtype === 'up' ? this.diamonds.h -10 : -10)
          .attr('text-anchor', 'middle')
          .attr('fill', d => d.data.type === COMPANY ? '#DE4A3C' : '#7A9EFF')
          .attr('opacity', d => !d.depth ? 0 : 1)
          .text(d => d.data.scale)
          .style('font-size', '14px')
          .style('font-family', 'PingFangSC-Regular')
          .style('font-weight', '400');

        // 公司名称
        // y轴 否表源头的字体距离
        nodeEnter.append('text')
          .attr('x', 0)
          .attr('y', d => {
            // 如果是上半部分
            if (showtype === 'up') {
              // 如果是1层以上
              if (d.depth) {
                return -this.diamonds.h / 2
              } else {
                return 0
              }
            } else {
              if (d.depth) {
                return 0
              } else {
                // if (d.data.name.length > 10) {
                //   return -5
                // }
                return 0
              }
            }
          })
          .attr('dy', d => d.depth ? (d.data.name.length > 10 ? '1.3em' : '1.8em') :  `${this.originDiamonds.h/2 - 10}px`)
          .attr('text-anchor', 'middle')
          .attr('fill', d => d.depth ? '#DE4A3C' : '#fff')
          .text(d =>d.depth ? (d.data.name.length > 10) ? d.data.name.substr(0, 10) : d.data.name : d.data.name)
          .style('font-size', d => d.depth ? '14px' : '18px')
          .style('font-family', 'PingFangSC-Medium')
          .style('font-weight', '500')
          .on('click', (d) => {
            if(d.data.id  && d.depth){
              // 跳转操作之类的
            }
          });

        // 名称过长 第二段
        nodeEnter.append('text')
          .attr('x', 0)
          .attr('y', d => {
             // ? (d.depth ? -this.diamonds.h / 2 : 0) : 0
            if (showtype === 'up') {
              if (d.depth) {
                return -this.diamonds.h / 2
              }
              return 8
            } else {
              if (!d.depth) {
                return 8
              }
              return 0
            }
          })
          .attr('dy', d => d.depth ? '2.5em' : '.3em')
          .attr('text-anchor', 'middle')
          .attr('fill', d => d.depth ? '#DE4A3C' : '#fff')
          .text(d => {
            // 索引从第19个开始截取有表示超出
            if(d.depth){
              if (d.data.name.substr(19, 1)) {
                return d.data.name.substr(10, 9) + '...'
              }
              return d.data.name.substr(10, 9)
            }else{
              return null
            }
          })
          .style('font-size', '14px')
          .style('font-family', 'PingFangSC-Medium')
          .style('font-weight', '500');

        // 认缴金额
        nodeEnter.append('text')
          .attr('x', 0)
          .attr('y', showtype === 'up' ? -this.diamonds.h / 2 : 0)
          .attr('dy', d => d.data.name.substr(10, d.data.name.length).length ? '4.5em' : '4.1em')
          .attr('text-anchor', 'middle')
          .attr('fill', d => d.depth ? '#445166' : '#fff')
          .text(d => d.data.money ? d.data.money.length > 12 ? `认缴金额：${d.data.money.substr(0, 12)}…` : `认缴金额：${d.data.money}万元` : '')
          .style('font-size', '12px')
          .style('font-family', 'PingFangSC-Regular')
          .style('font-weight', '400')
          .style('color', '#666666');

        /*
        * 绘制箭头
        * @param  {string} markerUnits [设置为strokeWidth箭头会随着线的粗细发生变化]
        * @param {string} viewBox 坐标系的区域
        * @param {number} markerWidth,markerHeight 标识的大小
        * @param {string} orient 绘制方向，可设定为：auto（自动确认方向）和 角度值
        * @param {number} stroke-width 箭头宽度
        * @param {string} d 箭头的路径
        * @param {string} fill 箭头颜色
        * @param {string} id resolved0表示公司 resolved1表示个人
        * 直接用一个marker达不到两种颜色都展示的效果
        */
        nodeEnter.append('marker')
          .attr('id', showtype + 'resolved0')
          .attr('markerUnits', 'strokeWidth')
          .attr('markerUnits', 'userSpaceOnUse')
          .attr('viewBox', '0 -5 10 10')
          .attr('markerWidth', 12)
          .attr('markerHeight', 12)
          .attr('orient', '90')
          .attr('refX', () => showtype === 'up' ? '-50' : '10')
          .attr('stroke-width', 2)
          .attr('fill', '#DE4A3C')
          .append('path')
          .attr('d', 'M0,-5L10,0L0,5')
          .attr('fill', '#DE4A3C');

        nodeEnter.append('marker')
          .attr('id', showtype + 'resolved1')
          .attr('markerUnits', 'strokeWidth')
          .attr('markerUnits', 'userSpaceOnUse')
          .attr('viewBox', '0 -5 10 10')
          .attr('markerWidth', 12)
          .attr('markerHeight', 12)
          .attr('orient', '90')
          .attr('refX', () => showtype === 'up' ? '-50' : '10')
          .attr('stroke-width', 2)
          .attr('fill', '#DE4A3C')
          .append('path')
          .attr('d', 'M0,-5L10,0L0,5')
          .attr('fill', '#7A9EFF');

        // 将节点转换到它们的新位置。
        let nodeUpdate = node
          // .transition()
          // .duration(DURATION)
          .attr('transform', d => showtype === 'up' ? 'translate(' + d.x + ',' + -(d.y) + ')' : 'translate(' + d.x + ',' + (d.y) + ')');


        // 将退出节点转换到父节点的新位置.
        let nodeExit = node.exit()
          // .transition()
          // .duration(DURATION)
          .attr('transform', () => showtype === 'up' ? 'translate(' + source.x + ',' + -(source.y) + ')' : 'translate(' + source.x + ',' + (parseInt(source.y)) + ')')
          .remove();

        nodeExit.select('rect')
          .attr('width', this.diamonds.w)
          .attr('height', this.diamonds.h)
          .attr('stroke', 'black')
          .attr('stroke-width', 1);

        // 修改线条
        let link = this.svg.selectAll('path.link' + showtype)
          .data(links, d => d.data.id);

        // 在父级前的位置画线。
        let linkEnter = link.enter().insert('path', 'g')
          .attr('class', 'link' + showtype)
          .attr('marker-start', d => `url(#${showtype}resolved${d.data.type})`)// 根据箭头标记的id号标记箭头
          .attr('stroke', d => d.data.type === COMPANY ? '#DE4A3C' : '#7A9EFF')
          .style('fill-opacity', 1)
          .attr('fill', 'none')
          .attr('stroke-width', '1px')
          .attr('d', () => {
            let o = {x: source.x0, y: source.y0};
            return _this.diagonal(o, o, showtype)
          });

        let linkUpdate = linkEnter.merge(link);
        // 过渡更新位置.
        linkUpdate
          // .transition()
          // .duration(DURATION)
          .attr('d', d => _this.diagonal(d, d.parent, showtype));

        // 将退出节点转换到父节点的新位置
        link.exit()
          // .transition()
          // .duration(DURATION)
          .attr('d', () => {
            let o = {
              x: source.x,
              y: source.y
            };
            return _this.diagonal(o, o, showtype)
          }).remove();

        // 隐藏旧位置方面过渡.
        nodes.forEach(d => {
          d.x0 = d.x;
          d.y0 = d.y
        });
      },

      // 拷贝到_children 隐藏1排以后的树
      // collapse (source) {
      //   if (source.children) {
      //     source._children = source.children;
      //     source._children.forEach(this.collapse);
      //     source.children = null;
      //     this.hasChildNodeArr.push(source);
      //   }
      // },

      // 获取点击上游的上游
      async fetchUpper (id, regCapi){
        Toast.loading({
          message: '加载中',
          forbidClick: true,
          duration: 0,
        });
        const dataSource = [];
        try{
          const response = await fetchEquityUpperInfo({
            token: this.token,
            instId: id,
            currentPage: 0,
            pageSize: 200,
            regCapi: regCapi,
          })
          const {code =0, records = []} = response
          if (code > 0 && records != null) {
            const dataSource = [] as any[];
            records.forEach(element =>{
              dataSource.push({
                isHaveChildren: null,
                money: element.amount ? formatMoney((element.amount / 10000).toFixed(2)) :'--',
                scale: element.hold_rati || '--%',
                name: element.chn_full_nm || '--',
                id: element.inst_cust_id || '--',
                type: '0'
              })
            })
            Toast.clear()
            return dataSource
          }else{
            Toast.clear()
            return dataSource
          }
        }catch(error){
          Toast.clear()
          return dataSource
        }
      },

      // 获取点击下游的下游
      async fetchBelow (id){
        Toast.loading({
          message: '加载中',
          forbidClick: true,
          duration: 0,
        });
        const dataSource = [];
        try{
          const response = await fetchEquityBelowInfo({
            token: this.token,
            instId: id,
            currentPage: 0,
            pageSize: 200,
          })
          const {code =0, records = []} = response
          if (code > 0 && records != null) {
            const dataSource = [] as any[];
            records.forEach(element =>{
              dataSource.push({
                isHaveChildren: null,
                money: element.amount ? formatMoney((element.amount / 10000).toFixed(2)) :'--',
                scale: element.hold_rati || '--%',
                name: element.chn_full_nm || '--',
                id: element.inst_cust_id || '--',
                type: '0'
              })
            })
            Toast.clear()
            return dataSource
          }else{
            Toast.clear()
            return dataSource
          }
        }catch(error){
          Toast.clear()
          return dataSource
        }
      },

      async click  (source, showType, sourceTree) {
        // 不是起点才能点
        // if (source.depth) {
        //   if (source.children) {
        //     source._children = source.children;
        //     source.children = null;
        //   } else {
        //     source.children = source._children;
        //     source._children = null;
        //   }
        // }
        if(source.children){
          // 点击减号
          source._children = source.children;
          source.children = null;
        }else {
          // 点击加号
          if(!source._children){
            let res = [] as any[]
            if(showType === 'up'){
              res = await this.fetchUpper(source.data.id, source.data.regCapi)
            }else {
              res = await this.fetchBelow(source.data.id)
            }
            if(!res.length){
              Notify({
                message: '上游或下游企业信息为空！',
                type: 'warning',
                duration: 1500
              })
              return
            }
            res.forEach(item =>{
              let newNode = this.d3.hierarchy(item)
              newNode.depth = source.depth + 1; 
              newNode.height = source.height - 1;
              newNode.parent = source; 
              if(!source.children){
                source.children = [];
                source.data.children = [];
              }
              source.children.push(newNode);
              source.data.children.push(newNode.data);
            })
          }else{
            source.children = source._children;
            source._children = null;
          }
        }
        this.update(source, showType, sourceTree)
      },

      diagonal (s, d, showtype) {
          // 折线
          let endMoveNum = 0;
          let moveDistance = 0;
          if (d) {
            if (showtype == 'down') {
              let downMoveNum =  d.depth ? this.diamonds.h/2 : this.originDiamonds.h/2 -10 ;
              // var downMoveNum =  30;
              let tmpNum = s.y + (d.y - s.y) / 2;
              endMoveNum = downMoveNum;
              moveDistance = tmpNum + endMoveNum;
            } else {
              let upMoveNum = d.depth ? 0 : -this.originDiamonds.h/2 + 5 ;
              let tmpNum = d.y + (s.y - d.y) / 2;
              endMoveNum = upMoveNum;
              moveDistance = tmpNum + endMoveNum;
            }
          }
          if (showtype === 'up') {
            return (
              'M' +
              s.x +
              ',' +
              -s.y +
              'L' +
              s.x +
              ',' +
              -moveDistance +
              'L' +
              d.x +
              ',' +
              -moveDistance +
              'L' +
              d.x +
              ',' +
              -d.y
            );
          }else {
            return (
              'M' +
              s.x +
              ',' +
              s.y +
              'L' +
              s.x +
              ',' +
              moveDistance +
              'L' +
              d.x +
              ',' +
              moveDistance +
              'L' +
              d.x +
              ',' +
              d.y
            );
          }
      },

      resetSvg () {
        this.d3.select('#treesvg').remove()
        this.init()
      },

      // 点击全屏
      showFullScreen(){
        let width = document.documentElement.clientWidth,
        height = document.documentElement.clientHeight,
        wrapper = document.getElementById('rightPenetrationpage') as HTMLElement,
        style = '';
        const navbar = document.querySelector('.navbar') as HTMLElement
        const fullScreen = document.querySelector('.full') as HTMLElement
        // const box = document.getElementById('penetrateChart').children[0]
        // const g = document.getElementById('penetrateChart').children[0].children[0]
        // setTimeout(()=>{
        //   // box.setAttribute('width', width)
        //   // box.setAttribute('height', height - 44)
          // g.setAttribute('transform', 'translate(' + (width / 2) + ',' + (height / 2) + ')')
        // }, 200)

        if (this.isFull) { // 竖屏
          console.log('竖过来')
          this.isFull = false
          this.svgH = height - 44
          this.svgW = width
          // 设置按钮和顶部样式
          fullScreen.classList.remove('fullRight')
          navbar.classList.remove('smallBar')
          style += 'width:100%';
          style += 'height:100%;';
          style += '-webkit-transform: translateX(0) translateZ(0px) rotate(0); -ms-transform: translateX(0) translateZ(0px) rotate(0); -moz-transform:translateX(0) translateZ(0px)  rotate(0); -o-transform: translateX(0) translateZ(0px) rotateY(0); transform: translateX(0) translateZ(0px) rotate(0);';
          style += '-webkit-transform-origin: 0 0;';
          style += '-ms-transform-origin: 0 0;';
          style += '-moz-transform-origin: 0 0;';
          style += '-o-transform-origin: 0 0;';
          style += 'transform-origin: 0 0;';
        } else { // 横屏
          console.log('横过来')
          this.isFull = true
          this.svgH = width - 44
          this.svgW = height
          // 设置按钮和顶部样式
          fullScreen.classList.add('fullRight')
          navbar.classList.add('smallBar')
          style += 'width:' + height + 'px;';// 注意旋转后的宽高切换
          style += 'height:' + width + 'px;';
          style += '-webkit-transform: translateX(0) translateZ(0px) rotate(90deg); -ms-transform: translateX(0) translateZ(0px) rotate(90deg); -moz-transform: translateX(0) translateZ(0px) rotate(90deg); -o-transform: translateX(0) translateZ(0px) rotate(90deg); transform:translateX(0) translateZ(0px) rotate(90deg);';
          // 注意旋转中点的处理
          style += 'transform-origin: ' + width / 2 + 'px ' + width / 2 + 'px;';
          style += '-webkit-transform-origin: ' + width / 2 + 'px ' + width / 2 + 'px;';
          style += '-ms-transform-origin: ' + width / 2 + 'px ' + width / 2 + 'px;';
          style += '-moz-transform-origin: ' + width / 2 + 'px ' + width / 2 + 'px;';
          style += '-o-transform-origin: ' + width / 2 + 'px ' + width / 2 + 'px;';
        }
        wrapper.style.cssText = style;
        // 重新渲染图片
        this.resetSvg()

      },

      // 点击返回
      onClickLeft(){
        // jsBridge.callHandler('navigationToSkip', {type: '0'}, ()=> {
        //   console.log('11111111111')
        // })
        history.back()
      }
    }

  })
</script>

<style lang="scss" scoped>
.father-box{
  transform: perspective(1000px);
  -ms-transform: perspective(1000px);
  -moz-transform: perspective(1000px);
  -webkit-transform: perspective(1000px);
  -o-transform: perspective(1000px);
}

.info-icon {
  width: 16px;
  height: 16px;
  background-image: url('../../assets/icon/icon_info.png');
  background-repeat: no-repeat;
  background-size: cover;
}
.full {
  position: absolute;
  top: 12px;
  right:20px;
  font-size: 14px;
  color: #DE4A3C;
  display: flex;
  z-index: 9999;
  line-height: 24px;
  .full-icon {
    width: 24px;
    height: 24px;
    background-image: url('../../assets/icon/icon_fullscreen.png');
    background-repeat: no-repeat;
    background-size: cover;
    margin-right: 7px;
  }
}

.fullRight{
  top: 12px !important;
  right:35px;
}
.smallBar {
  :deep(.van-nav-bar__left){
    display: none;
  }
  :deep(.van-nav-bar__content){
    background-color: transparent;
  }
  :deep(.van-nav-bar__title){
    // font-size: 4.8vh;
    margin-left:30px;
  }
}

</style>

