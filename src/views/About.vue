<!--
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
-->
<template>
  <div class="zrender">
    <div id="zrender-canvas"></div>
  </div>
</template>

<script>
  import zrender from 'zrender'

  export default {
    name: 'zrender',
    data() {
      return {
        zr: null,
        group: null,
        type: 0, // 用于辅助循环计数
        countData: {},
        styleObj: {
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
        },
        dataObj: {
          r: 5,
          x: 200,
          y: 300,
          width: 200, // 矩形宽
          height: 60, // 矩形高
          typeRectWidth: 150, // 公司类型矩形宽
          typeRectHeight: 20, // 公司类型矩形高
          lineWidth: 60, // 线宽度
          lineHeight: 100, // 线长
        },
        rectArray: [],
        leval: 0
      }
    },
    props: {
      mapLength: {
        type: [String, Number],
        default() {
          return 3
        }
      },
      dataList: {
        type: Object,
        default() {
          return {}
        }
      },
      draggable: { // 是否可以拖拽
        type: Boolean,
        default() {
          return true
        }
      },
      initData: { // 图形尺寸相关
        type: Object,
        default() {
          return {}
        }
      },
      initStyle: { // 样式相关设置
        type: Object,
        default() {
          return {};
        }
      }
    },
    mounted() {
      this.init();
    },
    methods: {
      init() {
        this.setInitData();
        this.computedData();
      },
      // 设置props相关值
      setInitData() {
        let tmpSytle = this.styleObj,
                tmpData = this.dataObj;
        for (let s in this.initStyle) {
          if (tmpSytle.hasOwnProperty(s)) {
            tmpSytle[s] = this.initStyle[s];
          }
        }
        for (let d in this.initData) {
          if (tmpData.hasOwnProperty(d)) {
            tmpData[d] = this.initData[d];
          }
        }
      },

      // 处理数据
      computedData() {
        if (this.dataList.hasOwnProperty('child')) {
          let computedChildData = [].concat(this.dataList.child);
          this.addChildCounts(computedChildData);
          this.addLeval(computedChildData);

          this.dataList['childCount'] = computedChildData.length;
          this.dataList['count'] = computedChildData.length;
          this.dataList['leval'] = 0;
          this.drawChart();
        }
      },
      addChildCounts(childData) {
        let data = childData;
        if (data && data.length > 0) {
          for (let i = 0; i < data.length; i++) {
            if (data[i].hasOwnProperty('child')) {
              data[i]['childCount'] = data[i]['child'].length;
              data[i]['count'] = data[i]['child'].length;
              this.addChildCounts(data[i]['child'])
            }
          }
        }
      },
      addLeval(data) {
        let levalData = data || [];
        if (levalData.length > 0) {
          for (let i = 0; i < levalData.length; i++) {
            this.leval = 1;
            levalData[i]['leval'] = this.leval;
            if (levalData[i].hasOwnProperty('child')) {
              this.setLeval(levalData[i].child)
            }
          }
        }
      },
      setLeval(data) {
        let levalData = data || [];
        this.leval++;
        if (levalData.length > 0) {
          for (let i = 0; i < levalData.length; i++) {
            levalData[i]['leval'] = this.leval;
            if (levalData[i].hasOwnProperty('child')) {
              this.setLeval(levalData[i].child)
            }
          }
        }
      },
      computedMostCount() {

      },

      drawChart() {
        let that = this;
        this.zr = zrender.init(document.getElementById('zrender-canvas'));
        this.group = new zrender.Group();
        this.group.draggable = this.draggable; // 是否开启拖拽
        this.group.progressive = 2; // 是否渐进加载（用于加载元素较多时）
        this.group.silent = false; // 是否相应鼠标事件 文档是反的  false响应  true不响应

        this.zr.add(this.group);
        window.onresize = function () {
          if (that.zr !== 'null') {
            that.zr.resize({
              width: null,
              height: null
            });
          }
        };
        let dataObj = this.dataObj;
        let tmpData = Object.assign({},this.dataList);

        this.drawRect(dataObj.r, dataObj.x, dataObj.y, dataObj.width, dataObj.height, false, tmpData);

        this.updateChart()
      },
      updateChart() {
        let that = this;
        this.group.eachChild(function(data) {
          if (data.name && data.name.indexOf('rect') > -1) {
            that.rectArray.push(data)
          }
        });

        let levalObj = {},
                len = that.rectArray;
        if (len.length > 0) {
          for (let i = 0; i < len.length; i++) {
            levalObj[len[i].name] = [];
          }
          // 将同一等级的图的位置放到一个数组中
          for (let key in levalObj) {
            for (let ii = 0; ii < len.length; ii++) {
              if (len[ii].name === key) {
                levalObj[key].push({
                  x: len[ii].shape.x,
                  y: len[ii].shape.y,
                  id: len[ii].id
                })
              }
            }
          }
          // 拿到了同一等级图 升序排序
          for (let iii in levalObj) {
            if (levalObj[iii].length > 0) {
              levalObj[iii].sort(function(a, b) {
                return a.y - b.y
              })
            }
          }
          // 拿到分组切排序好的数据后进行计算 n - n+1 如果间隔大于固定值证明没重叠 计算差值 小于固定值 代表图片叠加了 固定值其实就是线高
          let updateId = '';
          for (let n in levalObj) {
            if (levalObj[n].length > 1) {
              for (let nn = 0; nn < levalObj[n].length -1; nn++) {
                let intervals = levalObj[n][nn + 1].y - levalObj[n][nn].y - this.dataObj.lineHeight;
                if (intervals < 0) {
                  updateId = levalObj[n][nn+1].id;
                  break;
                }
              }
            }
          }
          this.getDataId(this.dataList.child, updateId);
        }
      },
      getDataId(dataList, updateId) {
        let data = dataList || [];
        if (data && data.length > 0 && updateId) {
          for (let i = 0; i < data.length; i++) {
            if (data[i].hasOwnProperty('child')) {
              for (let ii = 0; ii < data[i].child.length; ii++) {
                if(data[i]['child'][ii]['id'] === updateId) {
                  console.log(data[i]);
                  // 子集包含了updateId 给这个数据往上加childCount
                  if (data[i].childCount) {
                    data[i].childCount += 2;
                  }else {
                    data[i].childCount = 2;
                  }
                  this.zr.clear();
                  // this.drawChart();
                  break;
                }
              }
              this.getDataId(data[i]['child'], updateId)
            }
          }
        }
      },
      /**
       * @desc: 创建矩形
       * @params: r <number> 圆角
       * @params: x ,y <number> 左上角起点横纵坐标
       * @params: width,height <number> 宽度与高度
       * @params: data <object> 循环数据
       * @params: drawTypeRect <boolean> 是否绘制公司类型矩形框
       * @return: null
       */
      drawRect(r, x, y, width, height, drawTypeRect, data ) {
        let that = this;
        let fillColor = this.styleObj.rectFillColor,
                strokeColor = this.styleObj.rectStrokeColor,
                textColor = this.styleObj.rectTextColor;
        drawTypeRect ? fillColor = this.styleObj.whiteColor : fillColor;
        !drawTypeRect ? textColor = this.styleObj.whiteColor : textColor;
        let rect = new zrender.Rect({
          shape: {
            r: r,
            x: x,
            y: y,
            width: width,
            height: height
          },
          style: {
            fill: fillColor, // 填充颜色
            stroke: strokeColor, // 描边颜色，默认null
            lineWidth: this.styleObj.rectLineWidth, // 线宽
            text: data.orgName || '--', // 内部显示文字
            fontSize: this.styleObj.textFont, // 字体大小
            textFill: textColor, // 颜色
            truncate: { // 文字溢出相关设置
              outerWidth: width,
              ellipsis: '...'
            },
          },
          name: 'rect' + data.leval,
          onclick: function (value) {
            that.$emit('rectClick', data)
          },
        });
        this.group.add(rect);
        // 给数据赋值 唯一id用于计算是否重叠
        data.id = rect.id || '';

        // 绘制公司类型文字矩形框
        if (drawTypeRect) {
          this.drawCompanyTypeRect(x, y, data);
        }
        // 有child绘制
        if (data.hasOwnProperty('child')) {
          // 绘制总引导横线 只有一个child的话不绘制引导线
          let noGuideLine = false;
          if (data.child.length === 1) {
            noGuideLine = true
          }
          this.drawGuideLine(x + width, y + height / 2, data, noGuideLine);
        }
      },
      /**
       * @desc: 公司类型矩形
       * @params: x ,y <number> 左上角起点横纵坐标
       * @params: data <object> 循环数据
       * @return: null
       */
      drawCompanyTypeRect(x, y, data) {
        let rect = new zrender.Rect({
          shape: {
            r: 0,
            x: x + this.dataObj.width / 2 - this.dataObj.typeRectWidth / 2,
            y: y - this.dataObj.typeRectHeight / 2,
            width: this.dataObj.typeRectWidth,
            height: this.dataObj.typeRectHeight
          },
          style: {
            fill: this.styleObj.typeRectFillColor, // 填充颜色，默认#000
            stroke: this.styleObj.typeRectStrokeColor, // 描边颜色，默认null
            lineWidth: 1, // 线宽， 默认1
            text: '非挂牌上市公司' || '--', // 内部显示文字
            fontSize: this.styleObj.textFont, // 字体大小
            textFill: this.styleObj.typeRectTextColor, // 颜色
            truncate: { // 文字溢出相关设置
              outerWidth: this.dataObj.typeRectWidth,
              ellipsis: '...'
            }
          }
        });
        this.group.add(rect);
      },
      /**
       * @desc: 绘制总引导横线 y轴相同  x轴不同
       * @params: x ,y <number> 起点横纵坐标
       * @params: data <object> 循环数据
       * @params: noGuideLine <boolean> 当只有一个子集  不绘制引导线
       * @return: null
       */
      drawGuideLine(x, y, data, noGuideLine) {
        let GuideLineWidth = noGuideLine ? 20 : this.dataObj.lineWidth;
        let line = new zrender.Line({
          shape: {
            x1: x, // 起点x
            y1: y, // 起点y
            x2: x + GuideLineWidth, // 终点x
            y2: y // 终点y
          },
          style: {
            stroke: this.styleObj.baseColor, // 填充颜色，默认#000
          }
        });

        this.group.add(line);

        if (data.hasOwnProperty('child')) {
          let child = data.child || [],
                  totalCount = 0;

          // 计算有子集需要撑开的总高度 child长度为1、最后一个元素child有值不计算
          for (let x  = 0; x < child.length; x++) {
            if (child[x].hasOwnProperty('child') && child[x].child.length !== 1 && x !== child.length -1 ) {
              totalCount += child[x].child.length;
            }
          }
          totalCount +=  data.child.length;

          // 计算起点 为总高度的一半  因为最后一个不画所以往上移动半个距离
          let yStart= y + this.dataObj.lineHeight * ((totalCount / 2) + 0.5);
          this.drawColumnLIne(x + GuideLineWidth, yStart, noGuideLine, data);
        }
      },
      // 绘制竖线 y轴不同 x轴不同
      drawColumnLIne(x, y, noGuideLine, data) {
        let x1 = '',
                y1 = '',
                x2 = '',
                y2 = '',
                lineColor = '';

        // 绘制竖线 起点终点的x坐标都是一个 只需确认起点Y与终点Y
        x1 = x2 = x;
        y1 = y;
        y2 = y1 - this.dataObj.lineHeight;

        let child = data.child || [];

        let lastTab = true;
        if (child[data.count - 1] && child[data.count - 1].hasOwnProperty('child') && data.childCount !== 1) {
          y2 = y2 - ((child[data.count - 1].child.length - 1) * this.dataObj.lineHeight);
          // 数据是最后一个时 不在增加全部高度
          if (data.count === data.childCount) {
            lastTab = false;
            y2 = y2 + (((child[data.count - 1].child.length)/2) * this.dataObj.lineHeight);
          }
        }
        // 重新定义结束y2 作为下一次的起点y1
        data.yStart = y2;

        data.count === data.childCount && lastTab? lineColor = 'transparent' : lineColor = this.styleObj.baseColor;
        let line = new zrender.Line({
          shape: {
            x1: x1, // 起点x
            y1: y1, // 起点y
            x2: x2, // 终点x
            y2: y2  // 终点y
          },
          style: {
            stroke: lineColor, // 填充颜色
          }
        });
        this.group.add(line);
        // 递减当前数据递归
        data.count--;

        // 如果高度是计算了child的高度 将横线位移到中间
        if (child[data.count] && child[data.count].hasOwnProperty('child') && data.count !== 0 && data.count !== data.childCount -1) {
          let computedY =  data.yStart + ((child[data.count].child.length - 1) / 2) *this.dataObj.lineHeight;
          this.drawRowLine(x1, computedY, data.child[data.count]);
        }else if (!lastTab) {
          this.drawRowLine(x1, y1, data.child[data.count]);
        }else {
          this.drawRowLine(x1, data.yStart, data.child[data.count]);
        }
        if (data.count > 0) {
          this.drawColumnLIne(x, data.yStart, false, data)
        }

      },
      // 绘制横线 y轴相同  x轴不同
      drawRowLine(x, y, data) {
        let x1 = '',
                y1 = '',
                x2 = '',
                y2 = '';
        x1 = x;
        y1 = y;
        x2 = x + this.dataObj.lineWidth;
        y2 = y1;
        let line = new zrender.Line({
          shape: {
            x1: x1, // 起点x
            y1: y1, // 起点y
            x2: x2, // 终点x
            y2: y2  // 终点y
          },
          style: {
            stroke: this.styleObj.baseColor, // 填充颜色，默认#000
          }
        });
        this.group.add(line);
        this.drawCircleHasText(x1 + this.dataObj.lineWidth / 2, y);
        this.drawRect(this.dataObj.r, x1 + this.dataObj.lineWidth, y - this.dataObj.height / 2, this.dataObj.width, this.dataObj.height, true, data, )
      },
      // 绘制有文字的圆
      drawCircleHasText(x, y) {
        let Circle = new zrender.Circle({
          shape: {
            cx: x, // 圆心起点x
            cy: y, // 圆心起点y
            r: this.styleObj.circleR
          },
          style: {
            fill: this.styleObj.circleFillColor,
            stroke: this.styleObj.circleStrokeColor, // 填充颜色，默认#000
            text: this.styleObj.circleText,
            textFill: this.styleObj.circleTextColor, // 颜色
            fontSize: this.styleObj.textFont, // 字体大小
          },
        });
        this.group.add(Circle);
      },
      drawLoadMore(x, y) {
        let that = this;
        let Circle = new zrender.Circle({
          shape: {
            cx: x, // 圆心起点x
            cy: y + 2, // 圆心起点y
            r: this.styleObj.circleR
          },
          style: {
            fill: this.styleObj.whiteColor,
            stroke: this.styleObj.circleStrokeColor, // 描边颜色，默认#000
            text: "+",
            textFill: this.styleObj.circleStrokeColor, // 颜色
            fontSize: 20, // 字体大小
            fontWeight: '800',
            lineWidth: 2
          },
          onclick: function() {
            // that.group.eachChild(function (item) {
            //     that.zr.remove(item)
            // })
            // that.maxLength = that.maxLength + that.mapLength;
            // console.log(that.maxLength)
            // that.drawChart();
          }
        });
        this.group.add(Circle);
      }
    }
  }
</script>

<style>
  #zrender-canvas {
    height: 700px;
  }
</style>
