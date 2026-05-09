<template>
  <div class="planner-container">
    <div class="header-bar">
      <div class="title">
        <i class="el-icon-s-cooperation" style="color: #409EFF; margin-right: 8px;"></i>
        城市路网规划决策支持
      </div>
      <div class="controls">
        <span class="label">规划基准日：</span>
        <el-date-picker
            v-model="currDate"
            type="date"
            value-format="yyyy-MM-dd"
            size="small"
            placeholder="正在获取..."
            :clearable="false"
            @change="refreshData">
        </el-date-picker>
      </div>
    </div>

    <el-row :gutter="20" class="flex-row">
      <el-col :span="16" class="flex-col">
        <el-card shadow="hover" class="custom-card">
          <div slot="header" class="card-header border-left-blue">
            <span><i class="el-icon-s-data"></i> 部分路段流量负荷压力测试 </span>
            <el-tag size="mini" type="danger" effect="dark"> 超饱和警戒线: 100% </el-tag>
          </div>
          <div ref="loadChart" style="height: 340px; width: 100%;"></div>
        </el-card>
      </el-col>

      <el-col :span="8" class="flex-col">
        <el-card shadow="hover" class="custom-card">
          <div slot="header" class="card-header border-left-purple">
            <span><i class="el-icon-pie-chart"></i> 区域流量承载占比 & 未来增长预测 </span>
          </div>
          <div ref="regionChart" style="height: 340px; width: 100%;"></div>
        </el-card>
      </el-col>
    </el-row>

    <el-row :gutter="20" class="flex-row mt-20">
      <el-col :span="16" class="flex-col">
        <el-card shadow="hover" class="custom-card" body-style="padding: 0;" v-loading="isAiLoading" element-loading-text="AI 大模型正在深度推理信控策略..." element-loading-spinner="el-icon-loading">
          <div slot="header" class="card-header border-left-orange">
            <span><i class="el-icon-cpu"></i> 信号灯配时与流量匹配度分析 </span>
            <el-tag size="mini" :type="isAiLoading ? 'warning' : 'success'" effect="plain">
              <i :class="isAiLoading ? 'el-icon-loading' : 'el-icon-check'"></i> {{ isAiLoading ? 'AI 测算中...' : 'AI 测算完成' }}
            </el-tag>
          </div>

          <div style="height: 400px; overflow-y: auto;">
            <el-table :data="signalAdviceList" stripe style="width: 100%" size="medium">
              <el-table-column prop="roadName" label="路段/路口" width="140" show-overflow-tooltip>
                <template slot-scope="scope"><span style="font-weight: bold; color: #303133;">{{ scope.row.roadName }}</span></template>
              </el-table-column>

              <el-table-column label="饱和度 (V/C)" width="120" align="center">
                <template slot-scope="scope">
                  <el-progress :percentage="Math.min(scope.row.saturation * 50, 100)" :format="() => scope.row.saturation" :color="getSaturationColor(scope.row.saturation)" :stroke-width="8"></el-progress>
                </template>
              </el-table-column>

              <el-table-column label="优化策略" width="120" align="center">
                <template slot-scope="scope">
                  <el-tag size="mini" :type="scope.row.tagType" effect="light" style="font-weight: bold; border-width: 1px;">
                    {{ scope.row.strategyAction }}
                  </el-tag>
                </template>
              </el-table-column>

              <el-table-column label="深度分析与措施诊断" show-overflow-tooltip>
                <template slot-scope="scope"><span style="color: #606266; font-size: 13px;">{{ scope.row.strategyDetail }}</span></template>
              </el-table-column>

              <el-table-column label="预期增益" width="130" align="center">
                <template slot-scope="scope">
                  <div class="impact-badge" :class="{'impact-up': scope.row.impact.includes('+'), 'impact-down': scope.row.impact.includes('-')}">
                    <i :class="scope.row.impact.includes('+') ? 'el-icon-top' : (scope.row.impact.includes('-') ? 'el-icon-bottom' : 'el-icon-check')"></i>
                    {{ scope.row.impact }}
                  </div>
                </template>
              </el-table-column>
            </el-table>
          </div>
        </el-card>
      </el-col>

      <el-col :span="8" class="flex-col">
        <el-card shadow="hover" class="custom-card" v-loading="isAiLoading" element-loading-text="生成基建规划中..." element-loading-spinner="el-icon-loading">
          <div slot="header" class="card-header border-left-green">
            <span><i class="el-icon-truck"></i> 道路与硬件设施改造规划 </span>
          </div>

          <div style="height: 400px; overflow-y: auto; padding-right: 5px;">
            <el-timeline style="padding-left: 5px; padding-top: 15px;" v-if="constructionList.length > 0">
              <el-timeline-item v-for="(item, index) in constructionList" :key="index" placement="top" :color="getSaturationColor(item.saturation)" hide-timestamp>
                <el-card shadow="never" class="advice-item-card">
                  <div class="advice-header">
                    <span class="road-title">{{ item.roadName }}</span>
                    <el-tag size="mini" :type="item.saturation > 1 ? 'danger' : 'warning'" effect="plain">负荷 {{ (item.saturation * 100).toFixed(0) }}%</el-tag>
                  </div>

                  <div class="advice-content">
                    <div class="problem"><i class="el-icon-warning-outline" style="color:#F56C6C"></i><span style="color:#606266">{{ item.diagnose }}</span></div>
                    <div class="solution"><i class="el-icon-s-cooperation" style="color:#409EFF"></i><span style="color:#303133; font-weight: 500;">{{ item.advice }}</span></div>
                  </div>

                  <div class="project-meta">
                    <span><i class="el-icon-money"></i> 预估预算: {{ item.cost }}</span>
                    <span class="divider">|</span>
                    <span><i class="el-icon-date"></i> 工期: {{ item.duration }}</span>
                  </div>
                </el-card>
              </el-timeline-item>
            </el-timeline>

            <div v-if="!isAiLoading && constructionList.length === 0" style="text-align: center; color: #909399; margin-top: 80px;">
              <i class="el-icon-circle-check" style="font-size: 40px; color: #67C23A; margin-bottom: 10px;"></i><br>当前路网运行良好<br>无需硬件改造
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
      currDate: '',
      loadChartInst: null,
      regionChartInst: null,
      fullData: [],
      signalAdviceList: [],
      constructionList: [],
      isAiLoading: false
    }
  },
  mounted() {
    this.initPage();
    window.addEventListener('resize', this.handleResize);
  },
  beforeDestroy() {
    if (this.loadChartInst) this.loadChartInst.dispose();
    if (this.regionChartInst) this.regionChartInst.dispose();
    window.removeEventListener('resize', this.handleResize);
  },
  methods: {
    handleResize() {
      if (this.loadChartInst) this.loadChartInst.resize();
      if (this.regionChartInst) this.regionChartInst.resize();
    },

    fixSat(val) {
      if (!val) return '0.00';
      let num = parseFloat(val);
      if (num > 3.0) num = num / 10;
      return num.toFixed(2);
    },

    initPage() {
      axios.get('http://localhost:8080/api/analysis/quality_report')
          .then(res => {
            this.currDate = (res.data && res.data.date) ? res.data.date : this.getYesterday();
            this.refreshData();
          }).catch(() => {
        this.currDate = this.getYesterday();
        this.refreshData();
      });
    },

    getYesterday() {
      const date = new Date();
      date.setDate(date.getDate() - 1);
      return `${date.getFullYear()}-${String(date.getMonth() + 1).padStart(2, '0')}-${String(date.getDate()).padStart(2, '0')}`;
    },

    refreshData() {
      if (!this.currDate) return;

      this.fullData = [];
      this.signalAdviceList = [];
      this.constructionList = [];
      this.isAiLoading = true;

      Promise.all([
        axios.get(`http://localhost:8080/api/planning/advice?date=${this.currDate}`),
        axios.get(`http://localhost:8080/api/planning/region?date=${this.currDate}`)
      ]).then(([resAdvice, resRegion]) => {
        const adviceData = resAdvice.data || [];
        const regionData = resRegion.data || [];

        this.$nextTick(() => {
          this.initLoadChart(adviceData);
          this.initRegionChart(regionData, []);
        });

        const payload = {
          regions: regionData.map(i => ({ name: i.regionName, flow: i.totalFlow })),
          roads: adviceData.map(i => ({ name: i.roadName, saturation: this.fixSat(i.maxSaturation) }))
        };

        return axios.post('http://localhost:8080/api/planning/ai_batch_analyze', payload)
            .then(aiRes => {
              const aiData = aiRes.data;
              if (!aiData.error) {
                this.initRegionChart(regionData, aiData.regions);
                this.processSignalDataFromAi(aiData.roads);
                this.processConstructionDataFromAi(aiData.roads);
              } else {
                this.$message.error('AI大模型研判失败，触发降级展示');
              }
            });
      }).catch(() => {
        this.$message.warning(`获取数据或 AI 分析发生异常`);
      }).finally(() => {
        this.isAiLoading = false;
      });
    },

    processSignalDataFromAi(aiRoads) {
      this.signalAdviceList = aiRoads
          .filter(r => r.saturation > 0.4)
          .sort((a, b) => b.saturation - a.saturation)
          .map(r => ({
            roadName: r.roadName,
            saturation: r.saturation,
            tagType: r.signal.tag,
            strategyAction: r.signal.action,
            strategyDetail: r.signal.detail,
            impact: r.signal.impact
          }));
    },

    processConstructionDataFromAi(aiRoads) {
      this.constructionList = aiRoads
          .filter(r => r.hardware !== null)
          .sort((a, b) => b.saturation - a.saturation)
          .map(r => ({
            roadName: r.roadName,
            saturation: r.saturation,
            diagnose: r.hardware.diagnose,
            advice: r.hardware.advice,
            cost: r.hardware.cost,
            duration: r.hardware.duration
          }));
    },

    initLoadChart(data) {
      if (!this.$refs.loadChart) return;
      if (this.loadChartInst) this.loadChartInst.dispose();
      this.loadChartInst = echarts.init(this.$refs.loadChart);
      const viewData = data;
      const option = {
        tooltip: { trigger: 'axis', formatter: '{b} 负荷率: {c}%' },
        grid: { left: '3%', right: '12%', bottom: '20%', top: '10%', containLabel: true },
        dataZoom: [{ type: 'slider', show: true, xAxisIndex: [0], startValue: 0, endValue: 9, bottom: '2%' }],
        xAxis: {
          type: 'category',
          data: viewData.map((i, idx) => `${i.roadName} ${idx % 2 === 0 ? '(4车道)' : '(3车道)'}`),
          axisLabel: { rotate: 20, interval: 0, color: '#606266' },
          axisTick: { alignWithLabel: true }
        },
        yAxis: { type: 'value', name: '负荷率 (%)', splitLine: { lineStyle: { type: 'dashed' } } },
        series: [{
          name: '负荷率', type: 'bar', barWidth: '40%',
          data: viewData.map(i => (this.fixSat(i.maxSaturation) * 100).toFixed(0)),
          itemStyle: { borderRadius: [4, 4, 0, 0] },
          markLine: { symbol: 'none', data: [{ yAxis: 100, lineStyle: { color: '#F56C6C', type: 'solid', width: 2 }, label: { formatter: '超饱和阈值' } }] }
        }]
      };
      option.series[0].data = viewData.map(item => {
        let val = (this.fixSat(item.maxSaturation) * 100).toFixed(0);
        return {
          value: val,
          itemStyle: {
            color: val >= 100
                ? new echarts.graphic.LinearGradient(0, 0, 0, 1, [{offset: 0, color: '#FF7E5F'}, {offset: 1, color: '#FEB47B'}])
                : new echarts.graphic.LinearGradient(0, 0, 0, 1, [{offset: 0, color: '#00C6FB'}, {offset: 1, color: '#005BEA'}])
          }
        };
      });
      this.loadChartInst.setOption(option);
    },

    initRegionChart(data, aiRegions = []) {
      if (!this.$refs.regionChart) return;
      if (this.regionChartInst) this.regionChartInst.dispose();
      this.regionChartInst = echarts.init(this.$refs.regionChart);

      const processedData = data.map(i => {
        const aiMatch = aiRegions.find(r => r.name === i.regionName);
        let growth = aiMatch ? aiMatch.growthRate : 0.0;
        return { name: i.regionName, value: i.totalFlow, growthRate: growth };
      });

      const option = {
        tooltip: {
          trigger: 'item',
          formatter: function(params) {
            return `${params.name}<br/>当前承载流量: ${params.value}<br/>流量占比: ${params.percent}%<br/>未来交通AI预测增长: <span style="color:#F56C6C;font-weight:bold;">+${params.data.growthRate}%</span>`;
          }
        },
        legend: { bottom: '0', icon: 'circle' },
        series: [{
          name: '区域流量与预测', type: 'pie', radius: ['40%', '65%'], center: ['50%', '42%'],
          itemStyle: { borderRadius: 4, borderColor: '#fff', borderWidth: 2 },
          data: processedData,
          label: { show: true, formatter: '{b}\n预计增长 +{growth|{c}}%', rich: { growth: { color: '#F56C6C', fontWeight: 'bold' } } }
        }]
      };

      option.series[0].data = processedData.map(item => ({
        name: item.name, value: item.value, growthRate: item.growthRate,
        label: { formatter: `${item.name}\n预测增长 +{growth|${item.growthRate}}%` }
      }));
      this.regionChartInst.setOption(option);
    },

    getSaturationColor(val) {
      if (val > 1.15) return '#F56C6C';
      if (val > 0.95) return '#E6A23C';
      return '#67C23A';
    }
  }
}
</script>

<style scoped>
.planner-container {
  padding: 15px;
  background-color: #f5f7fa;
  min-height: 100vh;
  box-sizing: border-box;
}

.header-bar {
  display: flex;
  justify-content: space-between;
  align-items: center;
  margin-bottom: 20px;
  background: #fff;
  padding: 15px 20px;
  border-radius: 8px;
  box-shadow: 0 2px 12px 0 rgba(0,0,0,0.05);
}
.title { font-size: 20px; font-weight: bold; color: #303133; }
.label { font-size: 14px; color: #606266; margin-right: 10px; }

.flex-row {
  display: flex;
  align-items: stretch;
  flex-wrap: wrap;
}
.flex-col {
  display: flex;
  flex-direction: column;
}
.mt-20 { margin-top: 20px; }

.custom-card {
  border-radius: 8px;
  border: none;
  box-shadow: 0 4px 12px rgba(0, 0, 0, 0.05);
  flex: 1;
  display: flex;
  flex-direction: column;
}
::v-deep .el-card__body {
  flex: 1;
}

.card-header {
  font-weight: bold;
  font-size: 16px;
  display: flex;
  justify-content: space-between;
  align-items: center;
  padding-left: 10px;
}
.border-left-blue { border-left: 4px solid #409EFF; }
.border-left-purple { border-left: 4px solid #8e44ad; }
.border-left-orange { border-left: 4px solid #E6A23C; }
.border-left-green { border-left: 4px solid #67C23A; }

.advice-item-card {
  background-color: #fcfcfc;
  border: 1px solid #EBEEF5;
  transition: all 0.2s;
  margin-bottom: 5px;
}
.advice-item-card:hover {
  transform: translateY(-2px);
  box-shadow: 0 4px 12px rgba(0,0,0,0.08);
  border-color: #409EFF;
}
.advice-header {
  display: flex;
  justify-content: space-between;
  margin-bottom: 10px;
  align-items: center;
  padding-bottom: 8px;
  border-bottom: 1px dashed #EBEEF5;
}
.road-title { font-weight: bold; font-size: 15px; color: #303133; }

.advice-content {
  font-size: 13px;
  color: #606266;
  line-height: 1.8;
  margin-bottom: 10px;
}
.problem { margin-bottom: 4px; display: flex; align-items: flex-start; gap: 5px;}
.solution { display: flex; align-items: flex-start; gap: 5px; }

.project-meta {
  background: #f0f9eb;
  color: #67C23A;
  font-size: 12px;
  padding: 5px 10px;
  border-radius: 4px;
  display: inline-block;
  font-weight: bold;
}
.divider { margin: 0 8px; color: #C0C4CC; }

::-webkit-scrollbar { width: 6px; height: 6px; }
::-webkit-scrollbar-thumb { border-radius: 3px; background: #c0c4cc; }
::-webkit-scrollbar-track { border-radius: 3px; background: #f5f7fa; }

.impact-badge {
  display: inline-block;
  padding: 3px 8px;
  border-radius: 4px;
  font-size: 11px;
  font-weight: bold;
  white-space: nowrap;
}
.impact-up {
  background: #fdf6ec;
  color: #E6A23C;
  border: 1px solid #faecd8;
}
.impact-down {
  background: #f0f9eb;
  color: #67C23A;
  border: 1px solid #e1f3d8;
}

::v-deep .el-loading-mask {
  background-color: rgba(255, 255, 255, 0.7);
}
</style>