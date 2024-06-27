---
attachments: [Clipboard_2024-06-27-10-44-42.png]
title: echarts柱状图数据label遮挡问题
created: '2024-06-27T01:27:02.287Z'
modified: '2024-06-27T05:57:34.462Z'
---

# 6. echarts柱状图数据label遮挡问题
![](@attachment/tapd_46573674_base64_1717670732_945.png)
echarts版本为4.9.0，需求是为了解决上述柱状图出现label遮挡的问题，由于在业务中需要导出图表为pdf，故不能够隐藏label。在经过大量收集资料后，了解到一些方案：
通过echarts实例上的_chartView可以获取到图表渲染信息（包括图表中每一个柱子的X轴Y轴和Z轴）但是实际下来发现改渲染数据可能存在顺序不一致的情况。
![](@attachment/Clipboard_2024-06-27-10-44-42.png)

1.如何判断label重叠：
判断方法在两个柱子高度相近的情况下（根据渲染后的数据判断 高度差小于label高度），比较倆label长度的一半和两柱子x轴距离。
```
// 计算label是否会发生重叠
    isLabelOverLap(beforeLabelWidth, curLabelWidth,  beforeX, curX) {
      // x轴间隔
      let xGap = curX - beforeX;
      // 两者label的占用x轴间隔总宽度 （留余1）
      // label比例测试得出只针对于数字
      let labelAllLen = (beforeLabelWidth + curLabelWidth)/2 * 0.55  + 1
      // console.log(xGap, labelAllLen)
      return labelAllLen > xGap;
    },
```
网上的方案大多采用多层循环进行反复判断即（ABCD四个柱子，A对B，A对C，A对D。。。。。），对于组数据量较大的列表来说性能影响是非常大的。由于业务中存在大量这种的表格，都进行循环判断对整体页面加载速度影响巨大。

由于实际上业务中大多是相邻坐标轴会发送重叠现象。因此为了减少性能消耗，只对相邻坐标轴重叠情况进行了调整。同时为了避免多个相邻柱子过于密集导致的ABCD折叠，采用了叠加的偏移方案
```
// 获取整个图表的像素信息
    getEchartPX(chart) {
      let barWidth = -1; // 柱子宽度
      let barHeights = []; // 有三个系列，包括了每个柱子的像素信息
      let maxHeight = 0; // 计算柱子最大高度
      chart._chartsViews.forEach(bars => {
        // 获取每一个小series的bar像素信息
        let barH = []; // 单个系列的柱子像素信息
        bars.group._children.forEach(bar => {
          // 每个x对应bar的像素信息
          
          let shape = bar.shape;
          
          if (shape) {
            barH.push([shape.x, shape.y, 0 - shape.height]); // 像素信息： [横坐标x, 纵坐标y, 高度height]
            barWidth = shape.width; // 柱子宽度
            maxHeight =
              0 - shape.height > maxHeight ? 0 - shape.height : maxHeight;
          }
        });
        // 不知什么原因chartview返回数据可能出现乱序，现在根据bar的横坐标x进行排序
        barH.sort((a, b) => {
          return a[0] - b[0];
        });
        barHeights.push(barH);
      });
      // console.log(barHeights)
      return [barHeights, maxHeight];
    },
```

2.偏移叠加方案
在连续检测到相邻柱子label重叠时，累计记录offset偏移量，和当前偏移的柱子高度； 在两种情况下重置偏移量：1.当出现相邻不重叠的柱子时(当前这种情况在ABC柱子比较密集情况下，B与AC标签不重叠，而A C 重叠的情况下无法进行调整，可以根据情况调整为3个或以上柱子不重叠时进行重置,需要循环判断) 2.当当前偏移量超过一定范围（目前设置的是柱子长度+label+偏移量 < 容器高度，可以根据实际进行调整）

```
// 动态调整label位置尽量减少label重叠的情况出现
    changeLabelOffset(
      series,
      formatter,
      total,
      // barWidth,
      barHeights,
      maxHeight,
      fontSize
    ) {
      let seriesLen = series.length || 1;
      let dataLen = series[0].data.length || 1;
      if (seriesLen * dataLen < 8) {
        // 当数据列小于8时不进行操作 减少性能消耗
        return series;
      }

      if (seriesLen > 1) {
        let index = 0;
        let offsetSum = fontSize + 1;
        let beforeHeight = barHeights[0][0][2];
        
        while (index < dataLen) {
          // 遍历单组数据
          offsetSum = fontSize + 1;
          beforeHeight = barHeights[0][index][2];
          for (let i = 1; i < series.length; i++) {
            // 遍历数组
            // 1.当当前数据与上一列数据相同或相近
            // 根据formatter计算实际label
            let label = formatter(series[i].data[index], total);
            // 计算label宽度
            let labelWidth = label.length * fontSize;
            let beforeLabelWidth = formatter(series[i - 1].data[index], total).length * fontSize;
            // 当前柱子高度
            let curHeight = barHeights[i][index][2];
            if (
              Math.abs(curHeight - beforeHeight) > fontSize ||
              beforeHeight + offsetSum + fontSize >= maxHeight
            ) {
              // 当前柱子高度与累计偏移量的原始柱子高度不是一个级别时重置偏移量 默认高于12px 重置累计高度
              // 当柱子累计高度超过容器范围也进行重置
              offsetSum = fontSize + 1;
              // console.log(beforeHeight + offsetSum, maxHeight)
              beforeHeight = curHeight;
            }

            if (
              (Math.abs(barHeights[i - 1][index][2] - curHeight) < fontSize || series[i - 1].data[index].value === series[i].data[index].value)&&
              this.isLabelOverLap(beforeLabelWidth, labelWidth, barHeights[i - 1][index][0], barHeights[i][index][0]) &&
              // 当数据为 0 时不展示label不需要考虑
              curHeight > 0
            ) {
              
              // 当相邻两个柱子的高度相近<10px 且  label宽度大于柱子宽度时进行偏移
              // 累计修改偏移量
              series[i].data[index].label.offset = [0, -offsetSum];
              offsetSum += fontSize + 1;
              beforeHeight = curHeight;
              
            }
          }
          index++;
        }
      } else {
        // 对于非复合表格
        let offsetSum = fontSize + 1;
        let index = 1;
        console.log(series)
        let beforeHeight = barHeights[0][index][2];
        while (index < dataLen && index < barHeights[0].length) {
          // 非复合表格数据为0无法获取像素高度 故增加筛选条件
          
          let labelWidth = formatter(series[0].data[index], total).length * fontSize;
          let beforeLabelWidth = formatter(series[0].data[index-1], total).length * fontSize;
          let curHeight = barHeights[0][index][2];
          // console.log(barHeights[0][index][2], barHeights[0][index - 1][2]);
          if (
            Math.abs(curHeight - beforeHeight) > fontSize ||
            beforeHeight + offsetSum + fontSize >= maxHeight
          ) {
            // 当前柱子高度与累计偏移量的原始柱子高度不是一个级别时重置偏移量 默认高于12px 重置累计高度
            // 当柱子累计高度超过容器范围也进行重置
            // console.log(1111111)
            offsetSum = fontSize + 1;
            // console.log(beforeHeight + offsetSum, maxHeight)
            beforeHeight = curHeight;
          }
          if (
            Math.abs(curHeight - barHeights[0][index - 1][2]) < fontSize &&
            this.isLabelOverLap(beforeLabelWidth, labelWidth, barHeights[0][index - 1][0], barHeights[0][index][0]) &&
            curHeight > 0
          ) {
            // 当相邻两个柱子的高度相近<12px（label长度） 且  label宽度大于柱子宽度时
            // 修改label的偏移量
            series[0].data[index].label.offset = [0, -offsetSum];
            // series[0].data[index].label.isChange = true;
            offsetSum += fontSize + 1;
            beforeHeight = curHeight;
          }
          index++;
        }
      }
    },
```
3. 触发时机
在echarts加载完之后才能获取到渲染数据，因此可以绑定echarts的finished事件，在finished回调函数里面利用setOption操作更改每个label的对应offset值。注意setOption会触发Finished事件的再次回调，设置flag使得finished只执行一次。
```
// 只执行一次判断
        let hasSetOffset = false;
        // 仅对柱状图生效
        this.chart.on("finished", function() {
          if (hasSetOffset) {
            return;
          }
          let formatter = that.chart.getOption().series[0].label.formatter;
          let series = that.chart.getOption().series;
          that.changeLabelOffset(
            series,
            formatter,
            total,
            ...that.getEchartPX(that.chart),
            series[0].label.fontSize || 12
          );
          hasSetOffset = true;
          that.chart.setOption({
            series: series
          });
        });
```

