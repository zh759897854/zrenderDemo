#### 添加原生D3.js 生成树图
```
    开发环境：Vue
    props:  dataList <Object> 绘图数据
            draggable <Boolean> 图谱是否可拖拽
            initData <Object> 位置相关参数  默认设置
                r : 5, // 矩形圆角
                x : 200, // 起始x坐标
                y : 330, // 起始y坐标
                width : 200, // 矩形宽
                height : 60, // 矩形高
                typeRectWidth: 150, // 公司类型矩形宽
                typeRectHeight: 20, // 公司类型矩形高
                lineWidth: 60, // 线宽度
                lineHeight: 100, // 线长

            initStyle <Object> 样式相关参数 默认设置
                rectFillColor: '#14b8d4', // 矩形填充色
                rectStrokeColor: '#14b8d4', // 矩形描边色
                rectTextColor: '#333', // 矩形文字颜色
                typeRectFillColor: '#fff', // 类型矩形填充色
                typeRectStrokeColor: '#fff', // 类型矩形描边色
                typeRectTextColor: '#ffaa3d', // 类型矩形文字颜色
                circleFillColor: '#14b8d4', // 圆形填充色
                circleStrokeColor: '#14b8d4', // 圆形描边色
                circleTextColor: '#fff', // 圆形文字颜色
                circleR: 12, // 圆半径
                circleText: '投', // 圆形文字
                rectLineWidth: 2, // 描边线宽
                whiteColor: '#fff',
                baseColor: '#14b8d4',
                textFont: 14,
   缺点 计算时间长
```

```
<style lang="less">
.invest-map {
  background-color: #fff;
  position: relative;
  min-height: 200px;
  overflow-x: hidden;

  svg {
    position: relative;
    width: 100%;

    .link-g {
      cursor: pointer;
    }
  }

  .map-container {
    min-height: 600px;
    background-color: #fff;

    .active-text {
      cursor: pointer;
    }
  }

  .float-container {
    position: fixed;
    left: 0;
    top: 0;
    width: 100%;
    height: 100%;

    > div {
      width: 100%;
      height: 100%;
      background-color: #fff;
      opacity: 0.9;
    }

    .close {
      position: absolute;
      right: 30px;
      top: 30px;
      width: 30px;
      height: 30px;
      border-radius: 100%;
      border: 1px solid #999;
      color: #333;
      text-align: center;
      line-height: 30px;
      background-color: #fff;
      z-index: 100;
    }
  }
}

#float-svg {
  position: fixed !important;
  left: 0;
  top: 0;
  width: 100%;
  height: 100%;
}
</style>

<template>
  <div class="invest-map">
    <div class="map-container" ref="svg">
      <svg id="main-svg" xmlns="http://www.w3.org/2000/svg" version="1.1">
        <!-- defs是svg中可复用标签 -->
        <defs>
          <!-- 添加带箭头的标线 -->
          <marker
            id="markerArrow"
            markerWidth="8"
            markerHeight="8"
            refx="2"
            refy="5"
            orient="auto"
          ></marker>
          <!-- 高斯模糊 -->
          <filter id="Gaussian_Blur">
            <feGaussianBlur in="SourceGraphic" stdDeviation="5" />
          </filter>
        </defs>
      </svg>

      <div class="float-container" v-show="showFloat">
        <div class="mask"></div>
        <div class="close" role="link" @click="removeMask">X</div>
        <svg id="float-svg" xmlns="http://www.w3.org/2000/svg" version="1.1">
          <filter id="Gaussian_Blur__float">
            <feGaussianBlur in="SourceGraphic" stdDeviation="5" />
          </filter>
        </svg>
      </div>
    </div>
    <loading :loading="!init"></loading>
    <no-data
      v-show="!mapData.orgName && init"
      dataInfo="暂无投资族谱相关数据"
    ></no-data>
  </div>
</template>

<script>
import DataService from '../../../core/service/dataService.service';
import NoData from '../../../components/base/no-data.component';
import Loading from '../../../components/loading/map-loading.component';
import * as d3 from 'd3';

export default {
  data() {
    return {
      mapData: {},
      lastData: {},
      init: false,
      requestMark: false,
      floatData: [],
      showFloat: false,
      hierarchyDatalist: null
    };
  },
  props: {
    companyName: {
      type: String,
      default() {
        return '';
      }
    }
  },
  mounted() {
    this.getInvestMapData();
  },
  components: { Loading, NoData },
  methods: {
    // 获取投资族谱数据
    getInvestMapData() {
      let that = this;
      if (this.requestMark) return false;
      this.requestMark = true;
      that.init = false;

      new DataService({
        proxy: true,
        url: that.$VConfig.inters.dataInters.org.getInvestStructureData,
        data: {
          orgName: encodeURIComponent(that.companyName),
          isReduce: that.$VConfig.globalData.isReduce
        },
        success: function(d) {
          let data = d.data || {};
          data = data.data_result || {};
          that.speedShowData(data);
          that.mapData = Object.assign({}, data);
          that.computedData(that.mapData);
          that.upDataMap('#main-svg', 'Gaussian_Blur', that.lastData);
        },
        always: function() {
          that.requestMark = false;
          that.init = true;
        }
      }).request();
    },
    // 快速展示数据处理
    speedShowData(obj) {
      let data = Object.assign({}, obj);
      let tempData = {};
      tempData.orgName = this.deepClone(data.orgName);
      tempData.child = [];
      for (let i = 0; i < data.child.length; i++) {
        tempData.child.push(this.deepClone(data.child[i]));
      }

      // 隐藏二级之后的数据
      for (let k = 0; k < tempData.child.length; k++) {
        if (tempData.child[k].hasOwnProperty('child')) {
          tempData.child[k].child = [];
        }
      }
      this.computedData(tempData);
      this.upDataMap('#main-svg', 'Gaussian_Blur', this.lastData);
    },
    // 深拷贝数据
    deepClone(data) {
      let result = '';
      try {
        result = JSON.parse(JSON.stringify(data));
      } catch (e) {
        result = data;
      }
      return result;
    },
    // 处理投资族谱数据
    computedData(data) {
      if (data.hasOwnProperty('child')) {
        let tmpData = Object.assign(data);
        tmpData.children = data.child || [];
        tmpData.name = data.orgName;
        this.computedChild(tmpData);
        this.lastData = Object.assign({}, tmpData);
      }
    },
    // 计算自身下一级child个数
    computedChild(childData) {
      let data = childData.child;
      if (data && data.length > 0) {
        for (let i = 0; i < data.length; i++) {
          data[i]['name'] = data[i]['orgName'];
          data[i]['nodeType'] = this.getOrgType(data[i]['orgType'])[0].type;
          data[i]['type'] = this.getOrgType(data[i]['orgType']);
          if (data[i].hasOwnProperty('child')) {
            data[i]['children'] = data[i]['child'];
            this.computedChild(data[i]);
          }
        }
      }
    },
    // 计算公司属性
    getOrgType(list) {
      let data = [].concat(list);
      let tempOrgType = {};
      for (let i = 0, l = data.length; i < l; i++) {
        if (!Object.hasOwnProperty.call(tempOrgType, data[i])) {
          tempOrgType[data[i]] = 'a';
        } else {
          data.splice(i, 1);
          i--;
          l--;
        }
      }
      let typeData = [];
      for (let n = 0; n < data.length; n++) {
        let curWeight = this.$Funclnner.getCategoryWeight(data[n]);
        let curTypeText = this.$VApi.transferCategoryToText(data[n]);
        if (curTypeText.length > 0) {
          typeData.push({
            name: curTypeText,
            value: curWeight.value,
            type: curWeight.type
          });
        }
      }
      return typeData;
    },
    /**
     * @desc: 更新树图
     * @params: container 视图容器
     * @params: filter 滤镜类型
     * @params: curData 需处理的数据
     * @return: null
     */
    upDataMap(container, filter, curData) {
      // hierarchy 函数可以获取层级数据
      this.hierarchyDatalist = d3.hierarchy(curData);
      // 收起二级之后的数据
      this.hierarchyDatalist.each(function(d) {
        if (d.depth === 1 && d.data.children) {
          d._children = d.children || [];
          delete d.children;
          if (!d.collapse) {
            d.collapse = 'close';
          }
        }
        return d;
      });
      // 开始绘制
      this.drawMap(container, filter, this.hierarchyDatalist);
    },
    // 绘制树图
    drawMap(container, filter, hierarchyData) {
      let svg = d3.select(container),
        that_ = this;

      let nameData = [],
        maxName = '';

      // 计算最长的name
      for (let i = 0; i < hierarchyData.descendants().length; i++) {
        let item = hierarchyData.descendants()[i].data.name,
          reg = new RegExp('[A-Za-z]+$');
        let len = reg.test(item.slice(0, 4)) ? item.length / 2 : item.length;
        nameData.push(len || 0);
      }
      // 获取最大宽度
      maxName = d3.max(nameData);
      maxName = maxName < 20 ? 20 : maxName;
      // 声明一个tree工具
      let tree = d3
        .tree()
        .nodeSize([95, maxName * 28])
        .separation(function(a, b) {
          return a.parent == b.parent ? 1 : 2; // a.depth会影响坐标计算
        });
      // 获取tree需要的节点和边界线
      let treeData = tree(hierarchyData),
        nodes = treeData.descendants(),
        links = treeData.links(nodes);
      // 生成竖向的导线
      let linkHorizontal = d3
        .line()
        .y(function(d) {
          return d[0];
        })
        .x(function(d) {
          return d[1];
        })
        .curve(d3.curveStep); // curveStep属性是生成对角线，并以阶梯形式绘制图形
      // 外层画布
      svg.select('g').remove();
      let g = svg.append('g').attr('class', 'tree-container');

      // 添加拖拽事件
      d3.select(container).call(
        d3.drag().on('start', function() {
          let dragBox = d3.select(container + ' g').classed('dragging', true);
          d3.event.on('drag', dragged).on('end', ended);

          let translateData = d3.select(container + ' g').attr('transform');
          let computedData = translateData
            .substring(10, translateData.length - 1)
            .split(',');

          let xMove = parseFloat(computedData[0]),
            yMove = parseFloat(computedData[1]);

          // dx, dy 相对于上一次偏移量
          function dragged() {
            xMove += d3.event.dx;
            yMove += d3.event.dy;
            dragBox
              .raise()
              .attr('transform', 'translate(' + xMove + ',' + yMove + ')');
          }

          function ended() {
            dragBox.classed('dragging', false);
          }
        })
      );

      // 开始绘制图形
      g.append('g')
        .selectAll('path')
        .data(links)
        .enter()
        .append('path')
        .attr('d', function(d) {
          let sourceArray = [];
          sourceArray.push([d.source.x, d.source.y], [d.target.x, d.target.y]);
          return linkHorizontal(sourceArray);
        })
        .attr('fill', 'none')
        .attr('stroke', '#ffbc00')
        .attr('stroke-width', 1);

      let gs = g
        .append('g')
        .selectAll('g')
        .data(nodes)
        .enter()
        .append('g')
        .attr('transform', function(d) {
          let cx = d.x;
          let cy = d.y;
          // 绘制投资关系
          let circleObj = {
            circle4: [
              [-190, -17],
              [-152, -17],
              [-190, 17],
              [-152, 17]
            ],
            circle3: [
              [-190, -17],
              [-152, -17],
              [-190, 17]
            ],
            circle2: [
              [-190, 0],
              [-152, 0]
            ],
            circle1: [[-172, 0]],
            circle0: []
          };
          let that = d3.select(this);
          if (d.depth + '' !== '0' && d.data) {
            let relation = d.data.relation || [];
            for (let i = 0; i < relation.length; i++) {
              drawInvistCircle(
                that,
                circleObj['circle' + relation.length][i],
                relation[i]
              );
            }
          }
          // 绘制展开图标
          if (d.depth && d.collapse && d.collapse === 'close') {
            drawCollapse(that, d);
          }
          // 绘制收起图标
          if (d.depth && d.collapse && d.collapse === 'open') {
            drawCollapseOpen(that, d);
          }
          return 'translate(' + cy + ',' + cx + ')';
        })
        .on('click', function(d) {
          // 已经是弹出层值计入return
          if (that_.showFloat) {
            return false;
          }
          that_.drawFloatMap(d);
          event.stopPropagation();
          return false;
        });

      // 绘制节点 虚线框
      gs.insert('rect', '.filter_circle')
        .attr('x', -220)
        .attr('y', -40)
        .attr('width', function(d) {
          // depth为0 是祖先级节点不需要
          let reg = new RegExp('[A-Za-z]+$'),
            name = d.data.name;
          let len = reg.test(name.slice(0, 4)) ? name.length / 2 : name.length;
          return d.depth + '' === '0' ? 0 : (len + 5) * 15;
        })
        .attr('height', 80)
        .attr('rx', '40')
        .attr('ry', '40')
        .attr('fill', '#fff')
        .attr('stroke', function(d) {
          return d.data.curBorder ? '#14b8d4' : '#ffbc00';
        })
        .attr('stroke-dasharray', '5,5');
      //绘制节点 模糊背景
      gs.append('rect')
        .attr('x', -80)
        .attr('y', -40)
        .attr('width', function(d) {
          let reg = new RegExp('[A-Za-z]+$'),
            name = d.data.name;
          let len = reg.test(name.slice(0, 4)) ? name.length / 2 : name.length;
          return (len + 2) * 15;
        })
        .attr('height', 80)
        .attr('rx', '40')
        .attr('ry', '40')
        .attr('fill', '#bddbf7')
        .attr('filter', 'url(#' + filter + ')');
      //绘制节点 第一个矩形
      gs.append('rect')
        .attr('x', -80)
        .attr('y', -40)
        .attr('width', function(d) {
          let reg = new RegExp('[A-Za-z]+$'),
            name = d.data.name;
          let len = reg.test(name.slice(0, 4)) ? name.length / 2 : name.length;
          return (len + 2) * 15;
        })
        .attr('height', 80)
        .attr('rx', '40')
        .attr('ry', '40')
        .attr('fill', '#fff')
        .attr('stroke', '#fff')
        .attr('stroke-width', 1);
      //绘制节点 第二个矩形
      gs.append('rect')
        .attr('x', -76)
        .attr('y', -35)
        .attr('width', function(d) {
          let reg = new RegExp('[A-Za-z]+$'),
            name = d.data.name;
          let len = reg.test(name.slice(0, 4)) ? name.length / 2 : name.length;
          return (len + 1.5) * 15;
        })
        .attr('height', 70)
        .attr('rx', '35')
        .attr('ry', '35')
        .attr('fill', function(d) {
          if (d.depth + '' !== '0') {
            return d.data.nodeType === 'company' ? '#e8f3fc' : '#e5fafd';
          } else {
            return '#14b8d4';
          }
        })
        .attr('stroke', 'transparent')
        .attr('stroke-width', 1);

      //文字公司
      gs.append('text')
        .attr('x', -60)
        .attr('y', function(d) {
          if (d.depth + '' !== '0') {
            return -15;
          } else {
            return -5;
          }
        })
        .attr('dy', 10)
        .attr('fill', function(d) {
          if (d.depth + '' !== '0') {
            return '#333';
          } else {
            return '#fff';
          }
        })
        .text(function(d) {
          return d.data.name;
        })
        .on('click', function(d) {
          let name = d.data.name || '';
          if (name && d.depth + '' !== '0') {
            that_.$VRoute(
              'routeCompanyDetail',
              {
                companyName: name
              },
              '_blank'
            );
          }
        })
        .on('mouseover', function(d) {
          if (d.depth + '' === '0') {
            return;
          }
          d3.select(this)
            .attr('class', 'active-text')
            .attr('fill', '#14B8D4');
        })
        .on('mouseout', function(d) {
          if (d.depth + '' === '0') {
            return;
          }
          d3.select(this)
            .attr('class', '')
            .attr('fill', '#333');
        });
      // 文字类型
      gs.append('text')
        .attr('class', 'break-word')
        .attr('textLength', function(d) {
          if (d.depth + '' !== '0') {
            let result = [],
              maxLength = d.data.name.length;
            for (let i = 0; i < d.data.type.length; i++) {
              let name = d.data.type || [];
              result.push(name[i].name);
            }
            result = result.join(' ');
            result = result.length < maxLength ? result.length : maxLength;
            return result * 14;
          }
        })
        .attr('x', -60)
        .attr('y', 5)
        .attr('dy', 10)
        .attr('font-size', '12')
        .attr('fill', '#666')
        .text(function(d) {
          if (d.depth + '' !== '0') {
            let result = [],
              maxLength = d.data.name.length;
            for (let i = 0; i < d.data.type.length; i++) {
              let name = d.data.type || [];
              result.push(name[i].name);
            }
            result = result.join(' ');
            result =
              result.length - 4 > maxLength
                ? result.substr(0, maxLength) + '...'
                : result;
            return result;
          } else {
            return '';
          }
        });
      gs.selectAll('.break-word')
        .append('title')
        .text(function(d) {
          if (d.depth + '' !== '0') {
            let result = [];
            for (let i = 0; i < d.data.type.length; i++) {
              let name = d.data.type || [];
              result.push(name[i].name);
            }
            return result.join(' ');
          } else {
            return '';
          }
        });
      // 箭头
      gs.append('path')
        .attr('transform', 'translate(-120, -12)')
        .attr('d', function(d) {
          if (d.depth + '' !== '0') {
            return 'M2,6L10,6L10,0L23,12L10,24L10,18L2,18Z';
          } else {
            return null;
          }
        })
        .attr('fill', '#ffac2f')
        .attr('stroke', 'transparent')
        .attr('stroke-width', 0);

      // 展开图标
      function drawCollapse(that, data) {
        that
          .append('circle')
          .attr('class', 'link-g')
          .attr('cx', (data.data.name.length - 1) * 14)
          .attr('cy', 0)
          .attr('r', 10)
          .attr('fill', '#ffac2f')
          .attr('stroke', '#ffac2f')
          .on('click', function(d) {
            d.collapse = 'open';
            d.children = d._children || [];
            delete d._children;
            d3.select(container + ' g').remove();
            that_.drawMap(container, filter, that_.hierarchyDatalist);
            event.stopPropagation();
            return false;
          });
        that
          .append('line')
          .attr('x1', 0)
          .attr('x2', 15)
          .attr('y1', 0)
          .attr('y2', 0)
          .attr('stroke', '#fff')
          .attr('stroke-width', 2)
          .on('click', function() {
            event.stopPropagation();
            return false;
          })
          .attr(
            'transform',
            'translate(' + ((data.data.name.length - 1) * 14 - 7) + ',0)'
          );
        that
          .append('line')
          .attr('x1', 0)
          .attr('x2', 0)
          .attr('y1', 0)
          .attr('y2', 15)
          .attr('stroke', '#fff')
          .attr('stroke-width', 2)
          .on('click', function() {
            event.stopPropagation();
            return false;
          })
          .attr(
            'transform',
            'translate(' + (data.data.name.length - 1) * 14 + ', -7)'
          );
      }

      // 收起图标
      function drawCollapseOpen(that, data) {
        that
          .append('circle')
          .attr('class', 'link-g')
          .attr('cx', (data.data.name.length - 1) * 14)
          .attr('cy', 0)
          .attr('r', 10)
          .attr('fill', '#ffac2f')
          .attr('stroke', '#ffac2f')
          .on('click', function(d) {
            d.collapse = 'close';
            d._children = d.children;
            delete d.children;
            d3.select(container + ' g').remove();
            that_.drawMap(container, filter, that_.hierarchyDatalist);
            event.stopPropagation();
            return false;
          });
        that
          .append('line')
          .attr('x1', 0)
          .attr('x2', 15)
          .attr('y1', 0)
          .attr('y2', 0)
          .attr('stroke', '#fff')
          .attr('stroke-width', 2)
          .attr(
            'transform',
            'translate(' + ((data.data.name.length - 1) * 14 - 7) + ',0)'
          )
          .on('click', function() {
            event.stopPropagation();
            return false;
          });
      }

      // 投资类型圆
      function drawInvistCircle(that, data, text) {
        that
          .append('circle') // 绘制模糊背景色
          .attr('class', 'filter_circle')
          .attr('cx', data[0] - 1)
          .attr('cy', data[1])
          .attr('r', 14)
          .attr('fill', '#bddbf7')
          .attr('stroke', '#bddbf7')
          .attr('filter', 'url(#' + filter + ')');
        that
          .append('circle') // 绘制底色
          .attr('cx', data[0] - 1)
          .attr('cy', data[1])
          .attr('r', 14)
          .attr('fill', '#fff')
          .attr('stroke', '#fff');
        that
          .append('circle') // 绘制背景色
          .attr('cx', data[0])
          .attr('cy', data[1])
          .attr('r', 14)
          .attr('fill', '#18b4de')
          .attr('stroke', '#fff');
        that
          .append('text') // 绘制文字
          .attr('x', data[0])
          .attr('y', data[1])
          .attr('dy', 4)
          .attr('dx', -7)
          .attr('fill', '#fff')
          .text(that_.translateRelationText(text));
      }

      // 重置高度与初始位置
      if (container !== '#main-svg') {
        return;
      }
      let svgContainer = d3.select('#main-svg .tree-container'),
        height = svgContainer.node().getBBox().height;
      height = height > 1500 ? 1500 : height;
      svg.style('height', height);
      svgContainer.attr(
        'transform',
        'translate(' + 150 + ',' + height / 2 + ')'
      );
    },
    // 转换投资类型文字
    translateRelationText(text) {
      let result = '';
      switch (text) {
        case 'INVEST':
          result = '投';
          break;
        case 'SUBSCRIBE':
          result = '定';
          break;
        case 'TOP10':
          result = '十';
          break;
        case 'MANAGE':
          result = '管';
          break;
        case 'RATION':
          result = '配';
          break;
        default:
          break;
      }
      return result;
    },
    // 展示悬浮的map
    drawFloatMap(data) {
      // 给当前点击节点添加属性
      this.floatData = [];
      this.getParentData(Object.assign({}, data));
      let lastData = [].concat(this.floatData),
        l = lastData.length;
      // 给当前点击的添加特殊border
      lastData[l - 1].curBorder = true;
      if (l > 0) {
        for (let i = 0; i < l - 1; i++) {
          let item = lastData[i];
          item.children = [lastData[i + 1]];
        }
        this.showFloat = true;
        this.drawMap(
          '#float-svg',
          'Gaussian_Blur__float',
          d3.hierarchy(lastData[0])
        );
        // 设置居中位置
        let height = document.documentElement.clientHeight;
        d3.select('#float-svg .tree-container').attr(
          'transform',
          'translate(150, ' + height / 2 + ')'
        );
      }
    },
    // 处理悬浮数据
    getParentData(data) {
      if (data.hasOwnProperty('parent')) {
        // 每次处理时重新赋值children 需要深拷贝否则会改变原数据
        this.floatData.unshift(JSON.parse(JSON.stringify(data.data)));
        if (data.parent !== null) {
          this.getParentData(data.parent);
        }
      }
    },
    // 移除悬浮层
    removeMask() {
      d3.select('#float-svg .tree-container').remove();
      this.showFloat = false;
    }
  }
};
</script>

```
