html
复制代码
<template>
  <div class="stream-graph" @wheel="handleWheel">
    <input type="file" @change="handleFileChange" />
    <button @click="reset">重置画面</button>
    <svg :width="width" :height="height" ref="svg" @mousedown="handleMouseDown" >
      <path @mouseover="handleMouseover" @mouseout="handleMouseout"  
        v-for="(path, index) in paths"
        :key="index"
        :class="{ 'mask-effect': isHover(path.name) }"
        :d="path.d"
        :fill="path.color"
        :title="path.name"
        stroke="none"
      /> 
       <!-- 用于显示选区框 -->
      <rect
        v-if="isSelecting"
        :x="selectionBox.x"
        :y="selectionBox.y"
        :width="selectionBox.width"
        :height="selectionBox.height"
        class="selection-box"
      />
    </svg>
    <!-- 用于展示当前选中的名字 -->
    <div class="selected-name" v-if="selectedName">{{ selectedName }}</div>
  </div>
</template>

<script>
import Papa from 'papaparse';

export default {
  name: 'StreamGraph', 
  data() {
    return {
      width: 800, // SVG 宽度
      height: 400, // SVG 高度
      colors: ["#1E90FF", "#0000FF", "#0000CD", "#00008B", "#4169E1", "#6495ED", "#4682B4", "#2F4F4F", "#2A52BE", "#3A5FCD"],
      rawData: [], // 原始数据
      paths: [], // 路径数据
      lines: {}, // 转换后并补全后的数据 {name:[{year:n},{year:n}...]}
      yearSum: {}, // 年份总数 { year: sum }
      selectedName: '', // 当前选中的名字
      minYear: 0, // 开始的年份
      maxYear: 0, // 结束的年份
      minValue: 100000, // 最小值
      maxValue: 0, // 最大值
      xScaleFactor: 1, // x轴缩放因子
      yScaleFactor: 1, // y轴缩放因子
      mouseXOffset: 0, // x坐标偏移量
      mouseYOffset: 0, // y坐标偏移量
      isSelecting: false, // 是否正在选择
      selectionBox: { x: 0, y: 0, width: 0, height: 0 }, // 选区框数据
      startX: 0, // 鼠标按下时的x坐标
      startY: 0 // 鼠标按下时的y坐标
    };
  },
  methods: {
    handleFileChange(event) { // 处理文件选择事件
      const file = event.target.files[0];
      if (file) {
        const reader = new FileReader();
        reader.onload = (e) => {
          const csvData = e.target.result;
          this.readCsv(csvData).then(data => {
            this.rawData = data;
            this.paths = this.processedData(data);
          }).catch(err => {
            console.error('Error parsing CSV:', err);
          });
        };
        reader.readAsText(file);
      }
    },
    readCsv(csvData) { // 读取 CSV 文件
      return new Promise((resolve, reject) => {
        Papa.parse(csvData, {
          header: true,
          complete: (results) => {
            resolve(results.data);
          },
          error: (err) => {
            reject(err);
          }
        });
      });
    },
    getYearSum(data) { // 获取年份总数 
      let yearSum = [];
      data.forEach(d => {
        let nValue = parseInt(d.n); // 将字符串转换为数字
        let yearIndex = yearSum.findIndex(entry => entry.year === d.year);
        if (yearIndex === -1) {
          yearSum.push({ year: d.year, n: nValue });
        } else {
          yearSum[yearIndex].n += nValue;
        }
      });
      // 计算最小值和最大值
      this.minValue = Math.min(...yearSum.map(entry => entry.n));
      this.maxValue = Math.max(...yearSum.map(entry => entry.n));
      this.minYear = parseInt(yearSum[0].year);
      this.maxYear = parseInt(yearSum[yearSum.length - 1].year);
      return yearSum;
    },
    transformData(originalData) { // 转换数据=> {name:[{year:n},{year:n}...]}
      // 创建一个空对象来存储转换后的数据
      const transformedData = {};

      // 遍历原始数据
      originalData.forEach(item => {
        // 检查转换后的对象中是否已经存在该名字的数组
        if (!transformedData[item.name]) {
          transformedData[item.name] = [];
        }

        // 将原始数据项添加到对应的数组中
        transformedData[item.name].push({
          year: parseInt(item.year), // 将年份转换为整数
          n: parseInt(item.n) // 将数量转换为整数
        });
      });

      return transformedData;
    },
    completeLines(lines) { // 补全缺失的年份数据
      let newLines = {};

      Object.keys(lines).forEach(name => {
        let line = lines[name];

        // 创建一个从minYear到maxYear的年份范围数组
        let years = Array.from({ length: this.maxYear - this.minYear + 1 }, (_, i) => this.minYear + i);  
        // 使用reduce方法构建一个新对象，包含所有年份和对应的n值（如果不存在则为0）
        let completedLine = years.reduce((acc, year) => {
          // 查找当前年份是否存在于原始数据中
          let existingData = line.find(data => data.year === year);
          // 如果存在，使用原始的n值；如果不存在，设置n为0
          acc[year] = existingData ? existingData.n : 0, this.minValue = 0;
          return acc;
        }, {});

        let result = Object.entries(completedLine).map(([key, value]) => ({ year: parseInt(key), n: value }));
        newLines[name] = result;
      });

      return newLines;
    }, 
    getPathTop(data) { // 获取路径的最上边计算公式：（this.height-this.yearSum / (this.maxValue - this.minValue)) * this.height）/ 2 这个yScaleFactor要乘在外面
      let pathTop = [];
      data.forEach(d => { pathTop.push({ year: d.year, n: (this.height - d.n / (this.maxValue - this.minValue) * this.height ) / 2 * this.yScaleFactor + this.mouseYOffset}); });
      return pathTop;
    },   
    computedLines() { // 计算路径数据
      //绘制开始的最上边  计算公式：（this.height-yearSum）/2
      let pathTop = this.getPathTop(this.yearSum);
      // 创建一个空数组来存储路径数据
      let paths = [];
      let index = 0;
      // 遍历姓名数组
      Object.keys(this.lines).forEach(name => {
        // 获取对应姓名的路径点数组
        let points = this.lines[name];
        // 创建一个空数组来存储路径数据=> {name:name,d:path,color:color}
        let path = {};  
        let bottomLine = [];//图形的底部
        let topLine = [];//图形的顶部
        // 遍历路径点数组
        for (let i = 0; i < points.length; i++) {
          //缩放操作
          let xScale = this.maxYear - this.minYear;
          let x = (points[i].year - this.minYear) / xScale * this.width * this.xScaleFactor;
          let y = (points[i].n - this.minValue) / (this.maxValue - this.minValue) * this.height * this.yScaleFactor;
          // 计算操作
          y = y + pathTop[i].n;
          x = x + this.mouseXOffset;

          // 如果是第一个路径点，则移动到第一个路径点的位置
          if (i === 0) {
            //先初始化为空
            bottomLine = [];//图形的底部
            topLine = [];//图形的顶部
            topLine.push(`M${x},${y}`);
          } else {
            topLine.push(`L${x},${pathTop[i].n}`);
            bottomLine.unshift(`L${x},${y}`);//底部边是回来的
          }
          pathTop[i].n = y;//更新pathTop数组
        }
        // 为路径添加名字和颜色属性
        path.name = name;
        path.d = topLine.join('') + bottomLine.join('');//生成路径字符串
        path.color = this.colors[index % this.colors.length];
        index++;
        // 将路径数组添加到路径数据数组中
        paths.push(path);
      });
      //打印路径数据
      console.log(JSON.parse(JSON.stringify(paths)));
      return paths;
    },
    processedData(rawData) { // 处理数据
      //将原始数据按年份排序
      let sortedData = rawData.slice().sort((a, b) => a.year - b.year);
      //将表头去除
      sortedData.shift();
      // 转换数据格式=> {name:[{year:n},{year:n}...]}
      this.lines = this.transformData(sortedData); 
      //拿到每年的总和和开始年份和结束年份=> [{year:sum}]
      this.yearSum = this.getYearSum(sortedData);
      // 将lines中每个数据项中缺失的年份补全,统一设置为0
      this.lines = this.completeLines(this.lines);
      // 计算路径数据对原始数据进行缩放以及重新定位
      return this.computedLines();
    },
    handleMouseover(event) { // 处理悬浮事件  显示到页面上当前选中的名字 并且蒙版
      this.selectedName = event.target.getAttribute('title');
    },
    handleMouseout() { // 处理鼠标移出事件  隐藏蒙版
      this.selectedName = '';
    },
    isHover(name) { // 判断是否在蒙版上 
      if (this.selectedName === '' || this.isSelecting) return false;//如果没有选中任何人或者当前正在框选，则不显示蒙版
      return this.selectedName !== name;
    },
    handleWheel(event) { // 处理滚轮事件  缩放 同时得到鼠标的x坐标，作为缩放中心
      if (event.ctrlKey) {
        event.preventDefault();
        //鼠标x的坐标是相对于svg的，需要减去svg的初始坐标
        let mouseX = event.clientX -  this.$refs.svg.getBoundingClientRect().left;
        let s = this.xScaleFactor;
        // console.log('mouseX', mouseX);
        let zoomIntensity = 0.1;
        if (event.deltaY < 0) {// 放大
          this.xScaleFactor += zoomIntensity;
        } else {// 缩小
          this.xScaleFactor = Math.max(this.xScaleFactor - zoomIntensity, 0.1);
        }
        // 计算公式
        //绘制x（mouseX） = pointX * xScale + mouseXOffset
        //绘制x'(mouseX') = pointX * xScale' + mouseXOffset'
        //要保持x' = x，则有：
        //mouseXOffset' = (xScle - xScale') * mouseX + mouseXOffset
        //mouseXOffset' = mouseX - xScale' * mouseX
        //pointX = (mouseX - this.mouseXOffset) / s
        this.mouseXOffset = mouseX - (mouseX - this.mouseXOffset) / s * this.xScaleFactor;
        //计算错误公式，也能实现缩放，但是转换鼠标位置时会有奇怪的问题
        // this.mouseXOffset = -mouseX * (this.xScaleFactor - 1);
        // console.log('mouseXOffset', this.mouseXOffset);
        this.paths = this.computedLines();
      }
    },
    handleMouseDown(event) { // 处理鼠标按下事件  开始框选 并添加两个监听事件 mousemove 和 mouseup
      if (event.ctrlKey) {
        event.preventDefault();
        this.startX = event.clientX - this.$refs.svg.getBoundingClientRect().left;
        this.startY = event.clientY - this.$refs.svg.getBoundingClientRect().top;
        this.isSelecting = true;
        window.addEventListener('mousemove', this.handleMouseMove);
        window.addEventListener('mouseup', this.handleMouseUp);
      }
    },
    handleMouseMove(event) { // 处理鼠标移动事件  显示框选框
      let currentX = event.clientX - this.$refs.svg.getBoundingClientRect().left;
      let currentY = event.clientY - this.$refs.svg.getBoundingClientRect().top;
      let x = Math.min(this.startX, currentX);
      let y = Math.min(this.startY, currentY);
      let width = Math.abs(this.startX - currentX);
      let height = Math.abs(this.startY - currentY);
      this.selectionBox = { x, y, width, height };
    },
    handleMouseUp() { // 处理鼠标松开事件  结束框选 并移除两个监听事件 mousemove 和 mouseup
      this.isSelecting = false;
      // this.selectionBox = { x: 0, y: 0, width: 0, height: 0 };
      window.removeEventListener('mousemove', this.handleMouseMove);
      window.removeEventListener('mouseup', this.handleMouseUp);
      this.updateSelectedArea();
    },
    updateSelectedArea() { // 更新选中的区域, 将展示区域转换为这个
  // 打印 selectionBox
  console.log('selectionBox', JSON.parse(JSON.stringify(this.selectionBox)));
  
  // x轴计算公式
  // 绘制 x（mouseX） = pointX * xScale + mouseXOffset
  // 绘制 x'(mouseX') = pointX * xScale' + mouseXOffset'
  // 要使 x' = 0，则有：
  // x'(mouseX') = pointX * xScale' + mouseXOffset' = 0
  // mouseXOffset' = -pointX * xScale'
  // pointX = (mouseX - this.mouseXOffset) / s
  let s = this.xScaleFactor;
  this.xScaleFactor = this.xScaleFactor * (this.width / this.selectionBox.width);
  let pointX = (this.selectionBox.x - this.mouseXOffset) / s;
  this.mouseXOffset = -pointX * this.xScaleFactor;

  // y 轴计算公式
  //但是y轴的mouseYOffset是给pathTop加的，否则负数相加pathTop更新会有错误
  let yScle = this.yScaleFactor;
  this.yScaleFactor = this.yScaleFactor * (this.height / this.selectionBox.height);
  let pointY = (this.selectionBox.y - this.mouseYOffset) / yScle;
  this.mouseYOffset = -pointY * this.yScaleFactor;

  this.paths = this.computedLines();
},
    reset(){//按下重置画面
      if(!this.paths.length) return;
      this.xScaleFactor= 1, // x轴缩放因子
      this.yScaleFactor= 1, // y轴缩放因子
      this.mouseXOffset= 0, // x坐标偏移量
      this.mouseYOffset= 0, // y坐标偏移量
      this.isSelecting= false, // 是否正在选择
      this.selectionBox= { x: 0, y: 0, width: 0, height: 0 }, // 选区框数据
      this.startX= 0, // 鼠标按下时的x坐标
      this.startY= 0 // 鼠标按下时的y坐标
      this.paths = this.computedLines();
    },
  }
}
</script>

<style scoped>

.selected-name{
  position: absolute;
  top: 50px;
  left: 50px;
  font-size: 16px;
}

.stream-graph {
  position: relative;
  width: 1000px;
  height: 600px;
}

.mask-effect {
  opacity: 0.5;
  transition: opacity 0.3s ease;
}

.selection-box {
  fill: rgba(128, 128, 128, 0.3); 
  stroke: #808080; 
  stroke-width: 1;
}


</style>