<!--
    props:  dataList <Object> 绘图数据
            draggable <Boolean> 图谱是否可拖拽
            initData <Object> 位置相关参数
                r : 5, // 矩形圆角
                x : 200, // 起始x坐标
                y : 330, // 起始y坐标
                width : 200, // 矩形宽
                height : 60, // 矩形高
                typeRectWidth: 150, // 公司类型矩形宽
                typeRectHeight: 20, // 公司类型矩形高
                lineWidth: 60, // 线宽度
                lineHeight: 100, // 线长

            styleObj <Object> 样式相关参数
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
                countData: {
                    // count: 0, // 当前绘制竖线数量 用于计算循环绘制图形 并非固定值
                    // tmpCount: 0, // 当前绘制竖线数量 固定值
                },
                maxLength: 0,
                childCount: [],
                counts: 0
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
            styleObj: { // 样式相关设置
                type: Object,
                default() {
                    return {};
                }
            }
        },
        mounted() {
            this.computedData();
            this.setInitData();
            this.init();
        },
        methods: {
            computedData() {
                if (this.dataList.hasOwnProperty('child')) {
                    let firstChild = [].concat(this.dataList.child)
                    this.computedChild(firstChild);
                }
            },
            // 计算自身下一级child个数
            computedChild(childData) {
                let data = childData;
                if (data && data.length > 0) {
                    for (let i = 0; i < data.length; i++) {
                        if (data[i].hasOwnProperty('child') && data[i]['child'].length !== 1) {
                            data[i]['childCount'] = data[i]['child'].length;
                            this.computedChild(data[i]['child'])
                        }
                    }
                }
            },
            // 计算累加的child个数
            computedMostCount(childData) {

            },
            // 设置默认初始值
            setInitData() {
                let styleObj = this.styleObj,
                    initData = this.initData,
                    tempStyleObj = {
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
                    tempInitData = {
                        r: 5,
                        x: 200,
                        y: 400,
                        width: 200, // 矩形宽
                        height: 60, // 矩形高
                        typeRectWidth: 150, // 公司类型矩形宽
                        typeRectHeight: 20, // 公司类型矩形高
                        lineWidth: 60, // 线宽度
                        lineHeight: 100, // 线长
                    };
                for (let style in tempStyleObj) {
                    if (!styleObj.hasOwnProperty(style)) {
                        styleObj[style] = tempStyleObj[style]
                    }
                }
                for (let initItem in tempInitData) {
                    if (!initData.hasOwnProperty(initItem)) {
                        initData[initItem] = tempInitData[initItem]
                    }
                }
            },
            init() {
                let that = this;
                this.maxLength = this.mapLength;
                this.zr = zrender.init(document.getElementById('zrender-canvas'));
                this.group = new zrender.Group();
                this.drawChart();
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
                }
            },
            drawChart() {
                let initData = this.initData;
                let tmpData = Object.assign({},this.dataList);
                // if (tmpData && tmpData.child) {
                //     if (tmpData.child.length > this.maxLength) {
                //         tmpData.child = tmpData.child.splice(0, this.maxLength);
                //         tmpData.child[tmpData.child.length -1]['loadMore'] = true;
                //     }
                // }
                this.drawRect(initData.r, initData.x, initData.y, initData.width, initData.height, tmpData);
            },
            /**
             * @desc: 创建矩形
             * @params: r <number> 圆角
             * @params: x ,y <number> 左上角起点横纵坐标
             * @params: width,height <number> 宽度与高度
             * @params: data <object> 循环数据
             * @params: hasTypeText <boolean> 是否绘制公司类型矩形框
             * @return: null
             */
            drawRect(r, x, y, width, height, data, hasTypeText) {
                let that = this;
                let fillColor = this.styleObj.rectFillColor,
                    strokeColor = this.styleObj.rectStrokeColor,
                    textColor = this.styleObj.rectTextColor;
                hasTypeText ? fillColor = this.styleObj.whiteColor : fillColor;
                !hasTypeText ? textColor = this.styleObj.whiteColor : textColor;
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
                    onclick: function (value) {
                        that.$emit('rectClick', data)
                    }
                });
                this.group.add(rect);

                // 绘制公司类型文字矩形框
                if (hasTypeText) {
                    this.drawCompanyTypeRect(x, y, data);
                }
                // 假如有子集
                if (data.hasOwnProperty('child')) {
                    this.type++;
                    // 每个子集独有的countData属性
                    this.countData['count' + this.type] = {
                        count: 0,
                        yStart: 0
                    };
                    this.countData['count' + this.type]['count'] = data.child.length;
                    this.countData['count' + this.type]['yStart'] = 0;
                    this.countData['tmpCount' + this.type] = data.child.length;
                    // 绘制总引导横线
                    let noGuideLine = false;
                    if (data.child.length === 1) {
                        noGuideLine = true
                    }
                    this.drawGuideLine(x + width, y + height / 2, data, this.type, noGuideLine);
                }
                if (data.hasOwnProperty('loadMore')) {
                    this.drawLoadMore(x + width/2, y + height + this.styleObj.circleR)
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
                        x: x + this.initData.width / 2 - this.initData.typeRectWidth / 2,
                        y: y - this.initData.typeRectHeight / 2,
                        width: this.initData.typeRectWidth,
                        height: this.initData.typeRectHeight
                    },
                    style: {
                        fill: this.styleObj.typeRectFillColor, // 填充颜色，默认#000
                        stroke: this.styleObj.typeRectStrokeColor, // 描边颜色，默认null
                        lineWidth: 1, // 线宽， 默认1
                        text: '非挂牌上市公司' || '--', // 内部显示文字
                        fontSize: this.styleObj.textFont, // 字体大小
                        textFill: this.styleObj.typeRectTextColor, // 颜色
                        truncate: { // 文字溢出相关设置
                            outerWidth: this.initData.typeRectWidth,
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
             * @params: type 辅助计数
             * @params: noGuideLine <boolean> 当只有一个子集  不绘制引导线
             * @params: dataAll <object> 全部数据包含childCount
             * @return: null
             */
            drawGuideLine(x, y, data, type, noGuideLine) {
                let line = new zrender.Line({
                    shape: {
                        x1: noGuideLine ? 1 : x, // 起点x
                        y1: y, // 起点y
                        x2: noGuideLine ? 1 : x + this.initData.lineWidth, // 终点x
                        y2: y // 终点y
                    },
                    style: {
                        stroke: this.styleObj.baseColor, // 填充颜色，默认#000
                    }
                });

                this.group.add(line);

                let child = data.child || [];
                if (this.countData['tmpCount' + type] > 0) {
                    let computedCount = 0;
                    // 各子集下总含有多少child
                    for (let x  = 0; x < child.length; x++) {
                        if (child[x].hasOwnProperty('childCount')) {
                            computedCount += child[x].childCount;
                        }
                    }
                    // 计算起点 为总高度的一半  因为最后一个不用画所以往上移动半个距离
                    this.countData['count' + type]['yStart'] = y + this.initData.lineHeight * (((this.countData['tmpCount' + type] + computedCount) / 2) + 0.5);
                    let yStart = this.countData['count' + type]['yStart'];
                    // 画完总引导线后绘制竖线
                    let tab = false;
                    child[child.length - 1].hasOwnProperty('childCount')?tab = true:tab;
                    this.drawColumnLIne(noGuideLine ? x + 1 : x + this.initData.lineWidth, yStart, child, type, data, tab);
                }
            },
            // 绘制竖线 y轴不同 x轴不同
            drawColumnLIne(x, y, data, type, dataAll, isAdd) {
                let x1 = '',
                    y1 = '',
                    x2 = '',
                    y2 = '',
                    lineColor = '',
                    hasChild = dataAll && dataAll.childCount;
                // 绘制竖线 起点终点的x坐标都是一个 只需确认起点Y与终点Y
                x1 = x2 = x;
                y1 = y;
                y2 = y1 - this.initData.lineHeight;
                if (hasChild && isAdd) {
                    console.log(dataAll)
                    // 假如包含子集 向上减去初始值*子集个数
                    y2 = y2 - (dataAll.childCount - 1)*this.initData.lineHeight;
                }
                this.countData['count' + type]['yStart'] = y2;
                let yStart = this.countData['count' + type]['yStart'];

                this.countData['count' + type]['count'] === this.countData['tmpCount' + type]? lineColor = '#000' : lineColor = this.styleObj.baseColor;
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
                this.countData['count' + type]['count']--;
                let curData = data[this.countData['count' + type]['count']];
                if (hasChild && isAdd && this.countData['count' + type]['count'] !== 0
                    && this.countData['count' + type]['count'] !== this.countData['tmpCount' + type] - 1) {
                    // 需要加高度计算的向下位移到中间
                    this.drawRowLine(x1, yStart + ((dataAll.childCount - 1)/2) *this.initData.lineHeight, curData);
                }else {
                    this.drawRowLine(x1, yStart, curData);
                }

                if (this.countData['count' + type]['count'] > 0) {
                    let tmpData = data[this.countData['count' + type]['count'] - 1];
                    this.drawColumnLIne(x, yStart, data, type, tmpData, true)
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
                x2 = x + this.initData.lineWidth;
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
                // this.drawCircleHasText(x1 + this.initData.lineWidth / 2, y);
                this.drawRect(this.initData.r, x1 + this.initData.lineWidth, y - this.initData.height / 2, this.initData.width, this.initData.height, data, true)
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
