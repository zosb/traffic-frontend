<template>
  <div class="role-container">
    <el-row :gutter="20">
      <el-col :span="24">
        <el-card shadow="hover" body-style="padding: 10px 20px 30px 20px;">
          <div slot="header">
            <span>✅ 数据质量监控 </span>
          </div>
          <el-row type="flex" justify="space-around">
            <el-col :span="6" v-for="(item, index) in qualityList" :key="index" style="text-align: center;">
              <div ref="gaugeChart" style="width: 100%; height: 220px;"></div>

              <div style="margin-top: -20px; font-size: 16px; font-weight: bold; color: #303133;">{{ item.metricName }}</div>

              <el-tag size="small" :type="item.status === '优秀' || item.status === '正常' || item.status === '充足' ? 'success' : 'warning'" style="margin-top: 10px;">
                状态: {{ item.status }}
              </el-tag>
            </el-col>
          </el-row>
        </el-card>
      </el-col>
    </el-row>

    <el-row :gutter="20" style="margin-top: 20px;">
      <el-col :span="15">
        <el-card shadow="hover">
          <div slot="header">
            <span>📉 节假日 vs 工作日 流量对比分析 </span>
            <el-select v-model="selectedRoad" size="small" style="float: right; width: 180px;" @change="loadTrend" filterable placeholder="请选择路段">
              <el-option-group label="核心主干道">
                <el-option v-for="road in mainRoads" :key="road" :label="road" :value="road"></el-option>
              </el-option-group>
              <el-option-group label="环路快速路">
                <el-option v-for="road in ringRoads" :key="road" :label="road" :value="road"></el-option>
              </el-option-group>
            </el-select>
          </div>
          <div ref="trendChart" style="width: 100%; height: 380px;"></div>
        </el-card>
      </el-col>

      <el-col :span="9">
        <el-card shadow="hover" style="height: 483px; background-color: #fcfcfc; border-left: 4px solid #409EFF;">
          <div slot="header"><span style="color: #303133; font-weight: bold;"><i class="el-icon-cpu"></i> AI 智能归因报告 </span></div>
          <div v-if="analysisText" v-html="analysisText" style="line-height: 2.2; color: #606266; padding: 10px; font-size: 14px;"></div>
          <div v-else style="padding: 20px; text-align: center; color: #999;"><i class="el-icon-loading"></i> 正在生成分析...</div>
        </el-card>
      </el-col>
    </el-row>
  </div>
</template>

<script>
import * as echarts from 'echarts';
import axios from 'axios';

export default {
  data() {
    return {
      selectedRoad: '长安街',
      mainRoads: ['长安街', '建国门外大街', '中关村大街', '学院路', '朝阳北路', '西直门北大街'],
      ringRoads: ['二环', '三环', '四环', '五环'],
      qualityList: [],
      analysisText: '',
      trendChartInstance: null,
      gaugeInstances: []
    }
  },
  mounted() {
    this.loadQuality();
    this.loadTrend();
  },
  beforeDestroy() {
    if (this.trendChartInstance) this.trendChartInstance.dispose();
    this.gaugeInstances.forEach(inst => inst.dispose());
  },
  methods: {
    loadQuality() {
      axios.get('http://localhost:8080/api/analysis/quality').then(res => {
        if (this._isDestroyed) return;
        this.qualityList = res.data;
        this.$nextTick(() => {
          this.gaugeInstances.forEach(i => i.dispose());
          this.gaugeInstances = [];
          const gaugeDoms = this.$refs.gaugeChart;

          if (gaugeDoms) {
            this.qualityList.forEach((item, index) => {
              if (gaugeDoms[index]) {
                const chart = echarts.init(gaugeDoms[index]);
                this.gaugeInstances.push(chart);

                chart.setOption({
                  series: [{
                    type: 'gauge',
                    startAngle: 180,
                    endAngle: 0,
                    center: ['50%', '75%'],
                    radius: '90%',
                    min: 0,
                    max: item.metricName.includes('万') ? 100 : 100,
                    splitNumber: 5,
                    axisLine: {
                      lineStyle: {
                        width: 10,
                        color: [[0.3, '#F56C6C'], [0.7, '#E6A23C'], [1, '#67C23A']]
                      }
                    },
                    pointer: {
                      length: '60%',
                      width: 6,
                      itemStyle: { color: 'auto' }
                    },
                    axisTick: { distance: -10, length: 6, lineStyle: { color: '#fff', width: 1 } },
                    splitLine: { distance: -10, length: 10, lineStyle: { color: '#fff', width: 2 } },
                    axisLabel: { color: 'auto', distance: 15, fontSize: 12 },
                    detail: {
                      valueAnimation: true,
                      formatter: '{value}' + (item.metricName.includes('万') ? '' : '%'),
                      color: 'auto',
                      fontSize: 24,
                      offsetCenter: [0, '-20%']
                    },
                    data: [{ value: item.value }]
                  }]
                });
              }
            });
          }
        });
      });
    },

    loadTrend() {
      axios.get(`http://localhost:8080/api/analysis/trend_compare?date=2025-11-04&roadName=${this.selectedRoad}`)
          .then(res => {
            if (!this.$refs.trendChart) return;
            if (this.trendChartInstance) this.trendChartInstance.dispose();

            this.trendChartInstance = echarts.init(this.$refs.trendChart);
            const data = res.data;
            this.analysisText = data.insight;

            const option = {
              tooltip: { trigger: 'axis', axisPointer: { type: 'cross' } },
              legend: { data: ['今日流量', '历史同期(工作日)'], bottom: 0 },
              grid: { left: '3%', right: '4%', bottom: '10%', containLabel: true },
              xAxis: { type: 'category', boundaryGap: false, data: data.hours },
              yAxis: { type: 'value', name: '流量 (pcu/h)' },
              series: [
                {
                  name: '今日流量',
                  type: 'line',
                  smooth: true,
                  data: data.todayFlow,
                  itemStyle: { color: '#F56C6C' },
                  areaStyle: { opacity: 0.1, color: '#F56C6C' }
                },
                {
                  name: '历史同期(工作日)',
                  type: 'line',
                  smooth: true,
                  lineStyle: { type: 'dashed', width: 2 },
                  data: data.historyFlow,
                  itemStyle: { color: '#409EFF' }
                }
              ]
            };
            this.trendChartInstance.setOption(option);
          });
    }
  }
}
</script>