<template>

  <div
    id="structureChartIn"
    :style="{width:'100%',display:'block',margin:'auto'}"
  >
  </div>

</template>


<script lang="ts">
  // import  { setWatermark } from '@/utils/tool.js'
  import { defineComponent} from 'vue'
  import { formatMoney, getBLen } from '@/utils/tool'
  import { fetchEquityUpperInfo } from '@/api/companySearch'
  import { Notify, Toast } from 'vant'
  import * as $d3 from 'd3'
  // 过渡时间
  const DURATION = 400
  // 加减符号半径
  const SYMBOLA_S_R = 9
  // // 公司
  // const COMPANY = '0'
  // // 人
  // const PERSON = '1'

  export default defineComponent({
    props: {
      tree: {
        type: Object,
        default: () => {
          return {}
        }
      },
      token: {
        type: String,
        default: ''
      }
    },
    components: {
    },

    data () {
      return {
        diamonds: {} as any,
        originDiamonds: {} as any,
        d3: $d3,
        // hasChildNodeArr: [],
        root: {} as any,
        svg: {} as any,
        svgW: document.documentElement.clientWidth,
        svgH: document.documentElement.clientHeight - 88,
        title: '股权结构图',
        lastClickD: null,
      }
    },

    watch: {
      tree(newVal){
        if(newVal.name){
          this.init()
        }
      }
    },

    mounted () {
      // window.addEventListener('orientationchange', this.changeOrient)
    },

    // beforeUnmount(){
    //   window.removeEventListener('orientationchange', this.changeOrient)
    // },

    methods: {
      // changeOrient () {
      //   const box = document.getElementById('structureChartIn').children[0]
      //   const g = document.getElementById('structureChartIn').children[0].children[0]
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

      init () {
        let d3 = this.d3
        let svgW = this.svgW
        let svgH = this.svgH
        let margin = {top: 20, right: 20, bottom: 30, left: 10}
        // 方块形状
        this.diamonds = {
          w: 320,
          h: 60,
        }
        // 源头对象
        this.originDiamonds = {
          w: 208,
          h: 36
        }

        // 主图
        this.svg = d3.select('#structureChartIn').append('svg').attr('width', svgW).attr('height', svgH).attr('id', 'treesvgIn')
          .call(d3.zoom().scaleExtent([0.3, 2]).on('zoom', () => {
            const transform = d3.event.transform
            this.svg.attr('transform', transform.translate(margin.left, margin.top));
          }))
          .on('dblclick.zoom', null)
          .attr('style', 'position: relative;z-index: 2') // background-image:url(${setWatermark({name: this.$store.state.global.user.userName, loginName: this.$store.state.global.user.userId}).toDataURL()})
          .append('g').attr('id', 'gIn')
          .attr('transform', `translate(${margin.left},${margin.top})`)       


        // 拷贝树的数据
        let downTree = {} as any
        Object.keys(this.tree).map(item => {
          if (item === 'children') {
            downTree = JSON.parse(JSON.stringify(this.tree))
            downTree.children = this.tree[item]
          }
        })
        // hierarchy 返回新的结构 x0,y0初始化起点坐标
        this.root = d3.hierarchy(downTree);
        this.root.x0 = 0
        this.root.y0 = 0
        if(!this.root.children){
          this.update(this.root)
        }else {
          // this.root.children.forEach(this.collapse)
          this.update(this.root)
        }
      },

      /*
       *[update 函数描述], [click 函数描述]
       *  @param  {[Object]} source 第一次是初始源对象，后面是点击的对象
       */
      update (source) {
        // eslint-disable-next-line
        let _this = this

        let nodes= this.root.descendants()

        let index = -1, count = 0;
        this.root.eachBefore(function(n) {
          count+=20;
          n.style = 'node_' + n.depth;
          n.x = ++index * _this.diamonds.h + count;
          n.y = n.depth * 25; // 设置下一层水平位置向后移25px
        });


        let node = this.svg.selectAll('g.node')
          .data(nodes, d => {
            return d.data.id || ''
          } );

        let nodeEnter = node.enter().append('g')
          .attr('class', d => 'node node_' + d.depth)
          .attr('transform', 'translate(' + source.y0 + ',' + source.x0 + ')')
          .attr('opacity', 0);
        // 创建矩形
        nodeEnter.append('rect')
          .attr('type', d => d.data.id)
          .attr('width', d => d.depth ? this.diamonds.w : (d.data.children.length ? (getBLen(d.data.name)/2 * 18 + 62) : (getBLen(d.data.name)/2 * 18 + 20)))
          .attr('height', d => d.depth ? this.diamonds.h : this.originDiamonds.h)
          .attr('y', -this.diamonds.h / 2)
          .style('stroke', '#DE4A3C')
          .attr('stroke-width', 1)
          .attr('rx', 6)
          .attr('ry', 6)
          .style('fill', d => {
            return d.data.tap ? '#DE4A3C' : '#fff'
          });

        nodeEnter.append('rect')
          .attr('y', -this.diamonds.h / 2)
          .attr('height', d => d.depth ? this.diamonds.h : this.originDiamonds.h)
          .attr('width', 6)
          .attr('rx', 6)
          .attr('ry', 6)
          .style('fill', '#DE4A3C')

          // 文字
          nodeEnter.append('text')
			.attr('dy', d=> d.depth ? -7 : -5)
			.attr('dx', d=> d.depth ? 36 : (d.data.children.length ? 36 : 10))
			.style('font-size', d=> d.depth ? '14px' : '18px')
      .style('font-weight', '500')
      .attr('fill', d =>  d.depth ? '#333333' : '#fff')
			.text(function(d) {
        // 名字长度超过进行截取
        if(d.depth){
          if(d.data.name.length>20){
            return	d.data.name.substring(0, 19) + '...'; 
          } 
        }
				return d.data.name; 
			})
      .on('click', (d) => {
        if(d.data.id && d.depth){
            // 跳转操作之类的
        }
      });
 
          // 持股比例
		nodeEnter.append('text')
			.attr('dy', 17)
			.attr('dx', 36)
      .style('font-size', '12px')
      .style('fill', '#666666')
			.text(function(d) {
				if(!d.data.tap){
					return ('持股比例 ' +'：')
				} 
			});

      nodeEnter.append('text')
			.attr('dy', 17)
			.attr('dx', 95)
      .style('font-size', '12px')
      .style('fill', '#DE4A3C')
			.text(function(d) {
				if(!d.data.tap){
					return (d.data.scale)
				} 
			});

      // 认缴金额
      nodeEnter.append('text')
			.attr('dy', 17)
			.attr('dx', 150)
      .style('font-size', '12px')
      .style('fill', '#666666')
			.text(function(d) {
				if(!d.data.tap){
					return ('认缴金额 ' +'：')
				} 
			});

      nodeEnter.append('text')
			.attr('dy', 17)
			.attr('dx', 210)
      .style('font-size', '12px')
      .style('fill', '#DE4A3C')
			.text(function(d) {
        if(!d.data.tap){
          if(d.data.money.length > 14){
            return  d.data.money.substr(0, 14) + '...'
          }else{
            return (d.data.money + '万元')
          }
        } 
			});

      // 箭头
		// nodeEnter.append('text')
		// 	.attr('dy', 5.5)
		// 	.attr('dx', 200 )
		// 	.style('font-size', '20px')
		// 	.style('fill', '#000')
		// 	.text(function(d) {
		// 		if(!d.data.tap){
		// 			return '>'
		// 		}
      
		// 	});

      // 创造圆 加减
      let circle = nodeEnter.append('g')
            .attr('class', 'circle')
            .on('click', _this.click);
			
		circle.append('circle')
            .style('fill', '#F9DDD9')
            .style('stroke', '#FCEDEB')
            .style('stroke-width', 1)
            .attr('r', function (d) {
              if (d.children || d.data.isHaveChildren) {
                return 9;
              } else {
                return 0;
              }
            })
            .attr('cy', d => d.depth ? 0 : (- SYMBOLA_S_R -3))
            .attr('cx', 20)
		
		circle.append('text')
      .attr('dy', d => d.depth ? 4.5 : -7)
      .attr('dx', 20)
			.attr('text-anchor', 'middle')
			.attr('class', 'fa')
			.style('fill', '#DE4A3C')
			.text(function(d) {
				if (d.children) {
          return '-';
        } else if (d._children || d.data.isHaveChildren) {
          return '+';
        } else {
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

    /*
        * 绘制箭头
        * @param  {string} markerUnits [设置为strokeWidth箭头会随着线的粗细发生变化]
        * @param {string} viewBox 坐标系的区域
        * @param {number} markerWidth,markerHeight 标识的大小
        * @param {string} orient 绘制方向，可设定为：auto（自动确认方向）和 角度值
        * @param {number} stroke-width 箭头宽度
        * @param {string} d 箭头的路径
        * @param {string} fill 箭头颜色
        */
        // nodeEnter.append('marker')
        //   .attr('id', 'resolvedIn')
        //   .attr('markerUnits', 'strokeWidth')
        //   .attr('markerUnits', 'userSpaceOnUse')
        //   .attr('viewBox', '0 -5 10 10')
        //   .attr('markerWidth', 8)
        //   .attr('markerHeight', 8)
        //   .attr('orient', '0')
        //   .attr('refX', '10')
        //   // .attr('refY', '10')
        //   .attr('stroke-width', 2)
        //   .attr('fill', '#DE4A3C')
        //   .append('path')
        //   .attr('d', 'M0,-5L10,0L0,5')
        //   .attr('fill', '#DE4A3C');


        // 将节点转换到它们的新位置。
        nodeEnter
          // .transition()
          // .duration(DURATION)
          .attr('transform', function(d) { return 'translate(' + d.y + ',' + d.x + ')'; })
          .style('opacity', 1);

        node
        // .transition()
        // .duration(DURATION)
        .attr('transform', function(d) { return 'translate(' + d.y + ',' + d.x + ')'; })
        .style('opacity', 1)
        .select('rect');

        // 将退出节点转换到父节点的新位置.
        let nodeExit = node.exit()
          // .transition()
          // .duration(DURATION)
          .attr('transform', () => 'translate(' + source.y + ',' + (parseInt(source.x)) + ')')
          .style('opacity', 0)
          .remove();

          
        // nodeExit.select('rect')
        //   .attr('width', this.diamonds.w)
        //   .attr('height', this.diamonds.h)
        //   .attr('stroke', 'black')
        //   .attr('stroke-width', 1);

        // 修改线条
        let link = this.svg.selectAll('path.link')
          .data(this.root.links(), d => d.target.id);

        // 在父级前的位置画线。
        let linkEnter = link.enter().insert('path', 'g')
          .attr('class', d =>{
            return 'link link_' + d.target.depth
          } )
          // .attr('marker-end', 'url(#resolvedIn)')// 根据箭头标记的id号标记箭头
          .attr('stroke', '#DE4A3C')
          .style('fill-opacity', 1)
          .attr('fill', 'none')
          .attr('stroke-width', '1px')
          .attr('d', () => {
            let o = {x: source.x0, y: source.y0};
            return _this.diagonal({source: o, target: o})
          })
          // .transition()
          // .duration(DURATION)
          .attr('d', _this.diagonal);

        // 过渡更新位置.
        link
          // .transition()
          // .duration(DURATION)
          .attr('d', _this.diagonal);

        // 将退出节点转换到父节点的新位置
        link.exit()
          // .transition()
          // .duration(DURATION)
          .attr('d', () => {
            let o = {
              x: source.x,
              y: source.y
            };
            return _this.diagonal({source: o, target: o})
          }).remove();

        // 隐藏旧位置方面过渡.
        this.root.each(d => {
          d.x0 = d.x;
          d.y0 = d.y
        });
      },


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
            console.log(records)
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

      async click  (source) {
        // if (d.children) {
        //   d._children = d.children;
        //   d.children = null;
        // } else {
        //   d.children = d._children;
        //   d._children = null;
        // }
        // if (this.lastClickD){
        //   this.lastClickD._isSelected = false;
        // }
        // d._isSelected = true;
        // this.lastClickD = d;
        if(source.children){
          // 点击减号
          source._children = source.children;
          source.children = null;
        }else {
          // 点击加号
          if(!source._children){
            let res = [] as any[]
            res = await this.fetchUpper(source.data.id, source.data.regCapi)
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
        this.update(source);
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

      diagonal (d) {
        return `M ${d.source.y} ${d.source.x}
        H ${(d.source.y + (d.target.y-d.source.y)/2)}
        V ${d.target.x}
        H ${d.target.y}`;
      },

    }

  })
</script>


