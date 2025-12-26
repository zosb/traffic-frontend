<template>
  <div class="role-container">

    <el-row :gutter="20" class="mb-20">
      <el-col :span="24">
        <div class="dashboard-header">
          <div class="title">
            <i class="el-icon-monitor" style="color: #409EFF; margin-right: 8px;"></i>
            数据仓库质量管控台
          </div>
        </div>
      </el-col>
    </el-row>

    <el-row :gutter="20">
      <el-col :span="6">
        <div class="kpi-card bg-gradient-blue">
          <div class="card-title">ODS 层数据缺失率</div>
          <div class="card-value">{{ odsMetric.value }}%</div>
          <div class="card-footer">
            <span :class="odsMetric.status === '正常' ? 'text-white' : 'text-danger'">
              <i :class="odsMetric.status === '正常' ? 'el-icon-circle-check' : 'el-icon-warning'"></i>
              {{ odsMetric.status }} (阈值 &lt; 1%)
            </span>
          </div>
        </div>
      </el-col>

      <el-col :span="6">
        <div class="kpi-card bg-gradient-purple">
          <div class="card-title">Spark GPS 匹配准确率</div>
          <div class="card-value">{{ gpsMetric.value }}%</div>
          <div class="card-footer">
            <span :class="gpsMetric.status === '正常' ? 'text-white' : 'text-warning'">
              <i :class="gpsMetric.status === '正常' ? 'el-icon-s-operation' : 'el-icon-warning-outline'"></i>
              {{ gpsMetric.status }} (阈值 &gt; 95%)
            </span>
          </div>
        </div>
      </el-col>

      <el-col :span="6">
        <div class="kpi-card bg-gradient-orange">
          <div class="card-title">有效清洗记录数</div>
          <div class="card-value">{{ recordMetric.value }} <span style="font-size: 14px">万条</span></div>
          <div class="card-footer">
            <i class="el-icon-data-line"></i> 环比昨日 {{ recordMetric.trend }}
          </div>
        </div>
      </el-col>

      <el-col :span="6">
        <div class="kpi-card bg-gradient-red" style="cursor: pointer" @click="scrollToFaults">
          <div class="card-title">异常采集设备监控</div>
          <div class="card-value">{{ faultyDevices.length }} <span style="font-size: 14px">个</span></div>
          <div class="card-footer">
            <i class="el-icon-message-solid"></i> 点击查看故障清单
          </div>
        </div>
      </el-col>
    </el-row>

    <el-row :gutter="20" style="margin-top: 20px; display: flex; align-items: stretch;">

      <el-col :span="14" style="display: flex; flex-direction: column;">
        <el-card shadow="hover" class="custom-card" style="flex: 1;">
          <div slot="header" class="card-header">
            <span><i class="el-icon-trend-charts"></i> 数据处理质量趋势监控 (近7天)</span>
            <el-tooltip content="用于观察 Spark 算法参数调整后的数据质量波动" placement="top">
              <i class="el-icon-info text-gray"></i>
            </el-tooltip>
          </div>
          <div ref="qualityTrendChart" style="height: 320px; width: 100%;"></div>
        </el-card>
      </el-col>

      <el-col :span="10" style="display: flex; flex-direction: column;">
        <el-card shadow="hover" class="custom-card" body-style="padding: 0;" style="flex: 1;">
          <div slot="header" class="card-header border-bottom-red">
            <span style="color: #F56C6C"><i class="el-icon-warning"></i> 采集设备故障排查清单</span>
            <el-tag type="danger" size="mini" effect="plain">需运维介入</el-tag>
          </div>

          <div style="height: 320px; overflow-y: auto;">
            <el-table :data="faultyDevices" stripe style="width: 100%">
              <el-table-column prop="roadName" label="部署路段" width="120" show-overflow-tooltip></el-table-column>
              <el-table-column prop="captureCount" label="日采集量" width="100" align="center">
                <template slot-scope="scope">
                  <span class="text-danger font-bold">{{ scope.row.captureCount }}</span>
                </template>
              </el-table-column>
              <el-table-column label="诊断建议">
                <template>
                  <el-tag type="info" size="mini"><i class="el-icon-switch-button"></i> 设备离线</el-tag>
                </template>
              </el-table-column>
            </el-table>

            <div v-if="!faultyDevices || faultyDevices.length === 0" style="padding-top: 100px; text-align: center; color: #909399;">
              <i class="el-icon-circle-check" style="font-size: 40px; color: #67C23A;"></i>
              <p>今日设备运行状况良好</p>
            </div>
          </div>
        </el-card>
      </el-col>
    </el-row>

    <el-row :gutter="20" style="margin-top: 20px; display: flex; align-items: stretch;">

      <el-col :span="14" style="display: flex; flex-direction: column;">
        <el-card shadow="hover" class="custom-card" style="flex: 1;">
          <div slot="header" class="card-header">
            <span>📉 城市流量时段特征挖掘</span>
            <div style="float: right;">
              <el-date-picker
                  v-model="currDate"
                  type="date"
                  value-format="yyyy-MM-dd"
                  size="mini"
                  placeholder="加载中..."
                  style="width: 140px; margin-right: 10px;"
                  @change="refreshData">
              </el-date-picker>
              <el-select v-model="selectedRoad" size="mini" style="width: 120px;" @change="loadTrend" filterable>
                <el-option v-for="road in mainRoads" :key="road" :label="road" :value="road"></el-option>
              </el-select>
            </div>
          </div>
          <div ref="trendChart" style="width: 100%; height: 350px;"></div>
        </el-card>
      </el-col>

      <el-col :span="10" style="display: flex; flex-direction: column;">
        <el-card shadow="hover" class="custom-card" style="flex: 1; display: flex; flex-direction: column;">
          <div slot="header" class="card-header">
            <span style="color: #303133; font-weight: bold;"><i class="el-icon-cpu"></i> AI 智能归因报告</span>
            <el-button type="text" icon="el-icon-download" size="mini">导出报告</el-button>
          </div>
          <div class="ai-report-container" style="flex: 1; height: auto; min-height: 300px;">
            <div v-if="analysisText" v-html="analysisText" class="ai-content"></div>
            <div v-else class="loading-state">
              <i class="el-icon-loading"></i>
              <p>AI 正在分析 Hive 数仓与流量特征...</p>
            </div>
          </div>
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
      currDate: '', // 默认日期
      selectedRoad: '长安街',
      mainRoads: ['长安街', '建国门外大街', '中关村大街', '学院路', '朝阳北路', '西直门北大街'],

      odsMetric: { value: 0, status: '计算中' },
      gpsMetric: { value: 0, status: '计算中' },
      recordMetric: { value: 0, trend: '+0%' },
      faultyDevices: [],

      analysisText: '',

      trendChartInstance: null,
      qualityChartInstance: null,
    }
  },
  mounted() {
    this.loadQualityReport();
    window.addEventListener('resize', this.handleResize);
  },
  beforeDestroy() {
    if (this.trendChartInstance) this.trendChartInstance.dispose();
    if (this.qualityChartInstance) this.qualityChartInstance.dispose();
    window.removeEventListener('resize', this.handleResize);
  },
  methods: {
    handleResize() {
      if (this.trendChartInstance) this.trendChartInstance.resize();
      if (this.qualityChartInstance) this.qualityChartInstance.resize();
    },

    scrollToFaults() {
      // 简单的滚动定位效果
      this.$el.querySelector('.el-table').scrollIntoView({ behavior: 'smooth' });
    },

    refreshData() {
      this.loadTrend();
    },

    // 1. 加载数据质量报告 (C.2 核心)
    loadQualityReport() {
      // 复用之前的 quality_report 接口
      axios.get('http://localhost:8080/api/analysis/quality_report').then(res => {
        const data = res.data;
        if (data.date) {
          this.currDate = data.date;
          this.loadTrend();
        }
        // 映射 ODS 缺失率
        this.odsMetric = {
          value: data.odsValue,
          status: data.odsValue < 1 ? '正常' : '异常'
        };

        // 映射 GPS 准确率
        this.gpsMetric = {
          value: data.gpsValue,
          status: data.gpsValue > 95 ? '正常' : '需优化'
        };

        // 模拟一些故障设备数据 (如果后端没返回这么多)
        this.faultyDevices = data.faultyDevices || [];

        // 模拟有效记录数 (可以从后端 quality 接口获取)
        // 这里为了演示效果，给一个随机波动值
        this.recordMetric = { value: (145 + Math.random() * 5).toFixed(2), trend: '+2.5%' };

        // 渲染质量趋势图 (Spark 优化效果)
        this.initQualityTrendChart(data.trend);
      });
    },

    // 2. 渲染质量趋势图
    initQualityTrendChart(trendData) {
      if (!this.$refs.qualityTrendChart) return;
      if (this.qualityChartInstance) this.qualityChartInstance.dispose();

      this.qualityChartInstance = echarts.init(this.$refs.qualityTrendChart);

      // Mock 数据兜底，防止后端没传 trend 字段
      const dates = trendData ? trendData.dates : ['11-01', '11-02', '11-03', '11-04', '11-05', '11-06', '11-07'];
      const odsData = trendData ? trendData.ods : [2.1, 1.8, 1.5, 0.9, 0.8, 0.6, 0.5];
      const gpsData = trendData ? trendData.gps : [92, 93, 94.5, 96.2, 97.1, 97.5, 98.2];

      const option = {
        tooltip: { trigger: 'axis' },
        legend: { bottom: 0, data: ['ODS缺失率(%)', 'GPS准确率(%)'] },
        grid: { top: '15%', bottom: '15%', left: '5%', right: '5%', containLabel: true },
        xAxis: { type: 'category', data: dates, axisLine: { lineStyle: { color: '#909399' } } },
        yAxis: [
          { type: 'value', name: '缺失率', max: 5, splitLine: { lineStyle: { type: 'dashed' } } },
          { type: 'value', name: '准确率', min: 90, max: 100, splitLine: { show: false } }
        ],
        series: [
          {
            name: 'ODS缺失率(%)', type: 'line', smooth: true, data: odsData,
            itemStyle: { color: '#F56C6C' },
            markLine: { data: [{ yAxis: 1, name: '阈值' }] } // 1% 阈值线
          },
          {
            name: 'GPS准确率(%)', type: 'line', yAxisIndex: 1, smooth: true, data: gpsData,
            itemStyle: { color: '#67C23A' },
            areaStyle: { opacity: 0.1, color: '#67C23A' }
          }
        ]
      };
      this.qualityChartInstance.setOption(option);
    },

    // 3. 加载流量趋势 (C.1 核心)
    loadTrend() {
      if (!this.currDate) return;
      axios.get(`http://localhost:8080/api/analysis/trend_compare?date=${this.currDate}&roadName=${this.selectedRoad}`)
          .then(res => {
            if (!this.$refs.trendChart) return;
            if (this.trendChartInstance) this.trendChartInstance.dispose();

            this.trendChartInstance = echarts.init(this.$refs.trendChart);
            const data = res.data;
            this.analysisText = data.insight; // 保留 AI 报告功能

            const option = {
              tooltip: { trigger: 'axis', axisPointer: { type: 'cross' } },
              legend: { data: ['今日流量', '历史同期(工作日)'], top: 0 },
              grid: { left: '3%', right: '4%', bottom: '3%', containLabel: true },
              xAxis: { type: 'category', boundaryGap: false, data: data.hours },
              yAxis: { type: 'value', name: '流量 (pcu/h)' },
              series: [
                {
                  name: '今日流量', type: 'line', smooth: true,
                  data: data.todayFlow,
                  itemStyle: { color: '#409EFF' },
                  areaStyle: {
                    color: new echarts.graphic.LinearGradient(0, 0, 0, 1, [{ offset: 0, color: '#409EFF' }, { offset: 1, color: '#fff' }])
                  }
                },
                {
                  name: '历史同期(工作日)', type: 'line', smooth: true,
                  lineStyle: { type: 'dashed', width: 2 },
                  data: data.historyFlow,
                  itemStyle: { color: '#909399' }
                }
              ]
            };
            this.trendChartInstance.setOption(option);
          })
          .catch(() => {
            this.analysisText = "<span style='color:orange'>⚠️ 暂无该日期数据，请选择其他日期。</span>";
          });
    }
  }
}
</script>

<style scoped>
.role-container { padding: 15px; background-color: #f5f7fa; min-height: 100vh; }
.mb-20 { margin-bottom: 20px; }

/* 头部样式 */
.dashboard-header {
  display: flex;
  justify-content: space-between;
  align-items: center;
  background: #fff;
  padding: 15px 20px;
  border-radius: 8px;
  box-shadow: 0 2px 12px 0 rgba(0,0,0,0.05);
}
.dashboard-header .title { font-size: 18px; font-weight: bold; color: #303133; }

/* KPI 卡片 */
.kpi-card {
  border-radius: 8px;
  padding: 20px;
  color: #fff;
  height: 120px;
  display: flex;
  flex-direction: column;
  justify-content: space-between;
  box-shadow: 0 4px 12px rgba(0,0,0,0.1);
  transition: transform 0.2s;
}
.kpi-card:hover { transform: translateY(-3px); }

.bg-gradient-blue { background: linear-gradient(135deg, #36D1DC, #5B86E5); }
.bg-gradient-purple { background: linear-gradient(135deg, #BD3F32, #CB356B); } /* 这里的紫色调成了偏红紫，突出GPS校验 */
.bg-gradient-orange { background: linear-gradient(135deg, #FF8008, #FFC837); }
.bg-gradient-red { background: linear-gradient(135deg, #EB3349, #F45C43); }

.card-title { font-size: 14px; opacity: 0.9; }
.card-value { font-size: 28px; font-weight: bold; }
.card-footer { font-size: 12px; opacity: 0.8; }

.text-white { color: #fff; }
.text-danger { color: #ffebee; font-weight: bold; }
.text-warning { color: #fff3e0; font-weight: bold; }

/* 自定义卡片 */
.custom-card { border-radius: 8px; border: none; }
.card-header { display: flex; justify-content: space-between; align-items: center; font-weight: bold; font-size: 15px; }
.text-gray { color: #909399; margin-left: 5px; cursor: pointer; }
.text-danger { color: #F56C6C; }
.font-bold { font-weight: bold; }
.border-bottom-red { border-bottom: 2px solid #F56C6C; padding-bottom: 10px; margin-bottom: -10px; }

/* AI 报告区域 */
.ai-card { display: flex; flex-direction: column; }
.ai-report-container {
  height: auto;
  flex: 1;
  overflow-y: auto;
  background-color: #fdfdfd;
  border: 1px dashed #dcdfe6;
  border-radius: 4px;
  padding: 15px;
}
.ai-content { line-height: 2; font-size: 14px; color: #606266; }
.loading-state { text-align: center; margin-top: 100px; color: #909399; }
</style>