<template>
  <div class="stream-graph">
    <input type="file" @change="handleFileChange" />
    <svg :width="width" :height="height">
      <path @mouseover="handleMouseover" @mouseout="handleMouseout"  
        v-for="(path, index) in paths"
        :key="index"
        :class="{ 'mask-effect': isHover(path.name) }"
        :d="path.d"
        :fill="path.color"
        :title="path.name"
        stroke="none"
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
      yearSum: {}, // 年份总数 { year: sum }
      selectedName: '', // 当前选中的名字
      minYear: 0, // 开始的年份
      maxYear: 0, // 结束的年份
      yScale: 300000, // 初始缩放因子（0-200，000）
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
    scaleData(data) {//缩放数据
    let xScale = this.maxYear - this.minYear;

    data.forEach(d => {
      d.n = d.n / this.yScale * this.height;
      d.year = (parseFloat(d.year)  - this.minYear) / xScale * this.width;// 将字符串转换为数字
    });
    return data;
    },
    getAllNames(data) { // 获取所有婴儿的姓名
      const names = [];
      data.forEach(d => {
      if (!names.includes(d.name)) {
        names.push(d.name);
      }
    });
    return names;
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

      this.minYear = parseInt(yearSum[0].year);
      this.maxYear = parseInt(yearSum[yearSum.length - 1].year);

      return yearSum;
    },
    getPathTop(data) { // 获取路径的最上边计算公式：（this.height-this.yearSum）/2
    let pathTop = [];
      data.forEach(d => { pathTop.push({ year: d.year, n: (this.height - d.n) / 2 }); });
    return pathTop;
    },
    transformData(originalData) { // 转换数据
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

        // 输出转换后的数据
        // console.log(transformedData);

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
      acc[year] = existingData ? existingData.n : 0;
      return acc;
    }, {});

    let result = Object.entries(completedLine).map(([key, value]) => ({ year: parseInt(key), n: value }));
    newLines[name] = result;})

    // console.log('补全后的数据',JSON.parse(JSON.stringify(newLines))); // 打印补全后的数据

    return newLines;
    },
    computedLines(lines, pathTop) { // 计算路径数据
      // 创建一个空数组来存储路径数据
      let paths = [];
      let index = 0;
      // 遍历姓名数组
      Object.keys(lines).forEach(name => {
          // 获取对应姓名的路径点数组
          let points = lines[name];
          // 创建一个空数组来存储路径数据=> {name:name,d:path,color:color}
          let path = {};  
          let bottomLine = [];//图形的底部
          let topLine = [];//图形的顶部
          // 遍历路径点数组
          for (let i = 0; i < points.length; i++) {
              // 计算当前路径点的位置
              let x = points[i].year;
              let y = points[i].n + pathTop[i].n;
              // 如果是第一个路径点，则移动到第一个路径点的位置
              if (i === 0) {
                //先初始化为空
                  bottomLine = [];//图形的底部
                  topLine = [];//图形的顶部
                  topLine.push(`M${x},${y}`);
              } else {
                  topLine.push(`L${x},${pathTop[i].n}`) ;
                  bottomLine.unshift(`L${x},${y}`);//底部边是回来的
              }
              pathTop[i].n = y;//更新pathTop数组
          }
          // 为路径添加名字和颜色属性
          path.name = name;
          path.d = topLine.join('') + bottomLine.join('');//生成路径字符串
          path.color = this.colors[index % this.colors.length];
          index ++;
          // 将路径数组添加到路径数据数组中
          paths.push(path);
    });
      return paths;
    },
    processedData(rawData) { // 处理数据
    //将原始数据按年份排序
    let sortedData = rawData.slice().sort((a, b) => a.year - b.year);
    //将表头去除
    sortedData.shift();
    // 转换数据格式=> {name:[{year:n},{year:n}...]}
    let lines = this.transformData(sortedData); 
    //拿到每年的总和=> [{year:sum}]
    this.yearSum = this.getYearSum(sortedData);
    // 将lines中每个数据项中缺失的年份补全,统一设置为0
    lines = this.completeLines(lines);
    //对数据进行缩放成合适的大小
    let scaledLines = {};
    Object.keys(lines).forEach(name => {
      scaledLines[name] = this.scaleData(lines[name]);
    });
    //绘制开始的最上边  计算公式：（this.height-yearSum）/2
    let pathTop = this.getPathTop(this.scaleData(this.yearSum));
    //遍历排序后的数据，为每个姓名生成路径点 转换为SVG路径字符串
    let paths = this.computedLines(scaledLines, pathTop);

      // 返回SVG路径字符串数组
      return paths;

    },
    handleMouseover(event) { // 处理悬浮事件  显示到页面上当前选中的名字 并且蒙版
    this.selectedName = event.target.getAttribute('title');
    },
    handleMouseout() { // 处理鼠标移出事件  隐藏蒙版
    this.selectedName = '';
    },
    isHover(name) { // 判断是否在蒙版上 
      if (this.selectedName === '') return false;//如果没有选中任何人，则不显示蒙版
      return this.selectedName !== name;
    }
}}
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

</style>