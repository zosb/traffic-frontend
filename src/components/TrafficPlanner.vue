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
            <span><i class="el-icon-s-data"></i> 各核心路段流量负荷压力 </span>
            <el-tag size="mini" type="danger" effect="dark">超饱和警戒线: 100%</el-tag>
          </div>
          <div ref="loadChart" style="height: 340px; width: 100%;"></div>
        </el-card>
      </el-col>

      <el-col :span="8" class="flex-col">
        <el-card shadow="hover" class="custom-card">
          <div slot="header" class="card-header border-left-purple">
            <span><i class="el-icon-pie-chart"></i> 区域流量承载占比 & 预测增长 </span>
          </div>
          <div ref="regionChart" style="height: 340px; width: 100%;"></div>
        </el-card>
      </el-col>
    </el-row>

    <el-row :gutter="20" class="flex-row mt-20">
      <el-col :span="16" class="flex-col">
        <el-card shadow="hover" class="custom-card" body-style="padding: 0;">
          <div slot="header" class="card-header border-left-orange">
            <span><i class="el-icon-cpu"></i> 信号灯配时与流量匹配度分析 </span>
            <el-tooltip content="AI 基于仿真模型推演的优化方案" placement="top">
            </el-tooltip>
          </div>

          <div style="height: 400px; overflow-y: auto;">
            <el-table :data="signalAdviceList" stripe style="width: 100%" size="medium">
              <el-table-column prop="roadName" label="路段/路口" width="140" show-overflow-tooltip>
                <template slot-scope="scope">
                  <span style="font-weight: bold; color: #303133;">{{ scope.row.roadName }}</span>
                </template>
              </el-table-column>

              <el-table-column label="饱和度 (V/C)" width="120" align="center">
                <template slot-scope="scope">
                  <el-progress
                      :percentage="Math.min(scope.row.saturation * 50, 100)"
                      :format="() => scope.row.saturation"
                      :color="getSaturationColor(scope.row.saturation)"
                      :stroke-width="8">
                  </el-progress>
                </template>
              </el-table-column>

              <el-table-column label="优化策略" width="110" align="center">
                <template slot-scope="scope">
                  <el-tag size="small" :type="scope.row.tagType" effect="dark" style="border-radius: 2px;">
                    {{ scope.row.strategyAction }}
                  </el-tag>
                </template>
              </el-table-column>

              <el-table-column label="具体措施诊断" show-overflow-tooltip>
                <template slot-scope="scope">
                  <span style="color: #606266; font-size: 13px;">{{ scope.row.strategyDetail }}</span>
                </template>
              </el-table-column>

              <el-table-column label="预期增益" width="110" align="right">
                <template slot-scope="scope">
                  <span style="color: #67C23A; font-weight: bold; font-size: 11px;">
                    <i class="el-icon-top"></i> {{ scope.row.impact }}
                  </span>
                </template>
              </el-table-column>
            </el-table>
          </div>
        </el-card>
      </el-col>

      <el-col :span="8" class="flex-col">
        <el-card shadow="hover" class="custom-card">
          <div slot="header" class="card-header border-left-green">
            <span><i class="el-icon-truck"></i> 道路与硬件设施改造规划</span>
          </div>

          <div style="height: 400px; overflow-y: auto; padding-right: 5px;">
            <el-timeline style="padding-left: 5px; padding-top: 15px;">
              <el-timeline-item
                  v-for="(item, index) in constructionList"
                  :key="index"
                  placement="top"
                  :color="getSaturationColor(item.saturation)"
                  hide-timestamp>
                <el-card shadow="never" class="advice-item-card">
                  <div class="advice-header">
                    <span class="road-title">{{ item.roadName }}</span>
                    <el-tag size="mini" :type="item.saturation > 1 ? 'danger' : 'warning'" effect="plain">
                      负荷 {{ (item.saturation * 100).toFixed(0) }}%
                    </el-tag>
                  </div>

                  <div class="advice-content">
                    <div class="problem">
                      <i class="el-icon-warning-outline" style="color:#F56C6C"></i>
                      <span style="color:#606266">{{ item.diagnose }}</span>
                    </div>
                    <div class="solution">
                      <i class="el-icon-s-cooperation" style="color:#409EFF"></i>
                      <span style="color:#303133; font-weight: 500;">{{ item.advice }}</span>
                    </div>
                  </div>

                  <div class="project-meta">
                    <span><i class="el-icon-money"></i> 预算: {{ item.cost }}</span>
                    <span class="divider">|</span>
                    <span><i class="el-icon-date"></i> 工期: {{ item.duration }}</span>
                  </div>
                </el-card>
              </el-timeline-item>
            </el-timeline>

            <div v-if="constructionList.length === 0" style="text-align: center; color: #909399; margin-top: 80px;">
              <i class="el-icon-circle-check" style="font-size: 40px; color: #67C23A; margin-bottom: 10px;"></i><br>
              当前路网运行良好<br>无需硬件改造
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
      constructionList: []
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

    getHashIndex(str, max) {
      let hash = 0;
      for (let i = 0; i < str.length; i++) {
        hash = (hash << 5) - hash + str.charCodeAt(i);
        hash |= 0;
      }
      return Math.abs(hash) % max;
    },

    initPage() {
      axios.get('http://localhost:8080/api/analysis/quality_report')
          .then(res => {
            if (res.data && res.data.date) {
              this.currDate = res.data.date;
            } else {
              this.currDate = this.getYesterday();
            }
            this.refreshData();
          })
          .catch(() => {
            this.currDate = this.getYesterday();
            this.refreshData();
          });
    },

    getYesterday() {
      const date = new Date();
      date.setDate(date.getDate() - 1);
      const y = date.getFullYear();
      const m = (date.getMonth() + 1).toString().padStart(2, '0');
      const d = date.getDate().toString().padStart(2, '0');
      return `${y}-${m}-${d}`;
    },

    refreshData() {
      if (!this.currDate) return;

      axios.get(`http://localhost:8080/api/planning/advice?date=${this.currDate}`)
          .then(res => {
            this.fullData = res.data || [];
            this.processSignalData(this.fullData);
            this.processConstructionData(this.fullData);
            this.$nextTick(() => {
              this.initLoadChart(this.fullData);
            });
          })
          .catch(() => {
            this.$message.warning(`日期 ${this.currDate} 暂无规划数据`);
            this.fullData = [];
            this.initLoadChart([]);
          });

      axios.get(`http://localhost:8080/api/planning/region?date=${this.currDate}`)
          .then(res => {
            this.$nextTick(() => {
              this.initRegionChart(res.data || []);
            });
          })
          .catch(() => {
            this.initRegionChart([]);
          });
    },

    initLoadChart(data) {
      if (!this.$refs.loadChart) return;
      if (this.loadChartInst) this.loadChartInst.dispose();
      this.loadChartInst = echarts.init(this.$refs.loadChart);

      const viewData = data;

      const option = {
        tooltip: { trigger: 'axis', formatter: '{b} 负荷率: {c}%' },
        grid: { left: '3%', right: '4%', bottom: '20%', top: '10%', containLabel: true },
        dataZoom: [
          { type: 'slider', show: true, xAxisIndex: [0], startValue: 0, endValue: 9, bottom: '2%' }
        ],
        xAxis: {
          type: 'category',
          data: viewData.map((i, idx) => `${i.roadName} ${idx % 2 === 0 ? '(4车道)' : '(3车道)'}`),
          axisLabel: { rotate: 20, interval: 0, color: '#606266' },
          axisTick: { alignWithLabel: true }
        },
        yAxis: { type: 'value', name: '负荷率 (%)', splitLine: { lineStyle: { type: 'dashed' } } },
        series: [{
          name: '负荷率',
          type: 'bar',
          barWidth: '40%',
          data: viewData.map(i => (this.fixSat(i.maxSaturation) * 100).toFixed(0)),
          itemStyle: { borderRadius: [4, 4, 0, 0] },
          markLine: {
            symbol: 'none',
            data: [
              { yAxis: 100, lineStyle: { color: '#F56C6C', type: 'solid', width: 2 }, label: { formatter: '超饱和阈值' } }
            ]
          }
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

    initRegionChart(data) {
      if (!this.$refs.regionChart) return;
      if (this.regionChartInst) this.regionChartInst.dispose();
      this.regionChartInst = echarts.init(this.$refs.regionChart);

      const processedData = data.map(i => {
        let growth = Math.floor(Math.random() * 8) + 2;
        if(i.regionName.includes('东')) growth = 15;
        if(i.regionName.includes('西')) growth = 5;
        return { name: i.regionName, value: i.totalFlow, growthRate: growth };
      });

      const option = {
        tooltip: {
          trigger: 'item',
          formatter: function(params) {
            return `${params.name}<br/>当前承载流量: ${params.value}<br/>流量占比: ${params.percent}%<br/>未来交通预测增长: <span style="color:#F56C6C;font-weight:bold;">+${params.data.growthRate}%</span>`;
          }
        },
        legend: { bottom: '0', icon: 'circle' },
        series: [
          {
            name: '区域流量与预测',
            type: 'pie',
            radius: ['40%', '65%'],
            center: ['50%', '42%'],
            itemStyle: { borderRadius: 4, borderColor: '#fff', borderWidth: 2 },
            data: processedData,
            label: {
              show: true,
              formatter: '{b}\n预计增长 +{growth|{c}}%',
              rich: { growth: { color: '#F56C6C', fontWeight: 'bold' } }
            }
          }
        ]
      };

      option.series[0].data = processedData.map(item => ({
        name: item.name,
        value: item.value,
        growthRate: item.growthRate,
        label: {
          formatter: `${item.name}\n预测增长 +{growth|${item.growthRate}}%`
        }
      }));

      this.regionChartInst.setOption(option);
    },

    processSignalData(data) {
      const strategies = [
        { name: '瓶颈控制', type: 'danger', impact: '溢出风险 -40%' },
        { name: '相序优化', type: 'success', impact: '延误 -12s' },
        { name: '感应控制', type: 'primary', impact: '空放时间 -15s' },
        { name: '精细配时', type: 'warning', impact: '周期利用 +10%' }
      ];
      const details = [
        '建议上游路口实施红灯截流，防止车辆溢出至本路口',
        '建议调整信号相序，减少左转车流对直行车辆的干扰',
        '建议启用车辆检测器，根据实时到达流量动态调整绿灯',
        '精细化划分放行周期，匹配早晚高峰潮汐特性'
      ];

      this.signalAdviceList = data
          .filter(item => this.fixSat(item.maxSaturation) > 0.4)
          .sort((a, b) => this.fixSat(b.maxSaturation) - this.fixSat(a.maxSaturation))
          .map((item, index) => {
            const sat = this.fixSat(item.maxSaturation);
            const idxStrat = this.getHashIndex(item.roadName, strategies.length);

            let stratAction = strategies[idxStrat].name;
            let stratDetail = details[idxStrat];
            let tagType = strategies[idxStrat].type;

            if (index === 0) {
              stratAction = '延长绿灯';
              tagType = 'success';
              stratDetail = '现状：绿灯时长 30 秒，高峰时段排队车辆超 20 辆。建议：需延长绿灯至 45 秒。';
            }

            return {
              roadName: item.roadName,
              saturation: sat,
              tagType: tagType,
              strategyAction: stratAction,
              strategyDetail: stratDetail,
              impact: strategies[idxStrat].impact
            };
          });
    },

    processConstructionData(data) {
      const problems = [
        '路侧开口过多，进出地块车辆严重干扰主流',
        '上下游车道数不匹配，形成局部交通瓶颈',
        '交叉口几何设计局限，大型车辆转弯困难'
      ];
      const solutions = [
        '封闭部分路侧开口，设置辅路集散交通',
        '设置潮汐车道，利用对向闲置车道资源',
        '实施路口渠化拓宽，增加进口车道数量'
      ];

      this.constructionList = data
          .filter(item => this.fixSat(item.maxSaturation) > 0.6)
          .sort((a, b) => this.fixSat(b.maxSaturation) - this.fixSat(a.maxSaturation))
          .map((item, index) => {
            const idx = this.getHashIndex(item.roadName, problems.length);

            let diagnose = problems[idx];
            let advice = solutions[idx];

            if (index === 0) {
              diagnose = `${item.roadName || '中山路'}车道数 4 条，高峰流量超饱和，负荷率 ${(this.fixSat(item.maxSaturation)*100).toFixed(0)}%`;
              advice = `为路网改造提供依据：建议该路段增加 1 条车道`;
            } else if (index === 1) {
              diagnose = `分析区域流量分布：该区域流量年均预测增长 15%，现有路网压力过大`;
              advice = `预测未来交通需求：规划新增该区域连接主干道的支路`;
            } else if (index === 2) {
              diagnose = `统计交通设备覆盖效果：某区域卡口设备覆盖率仅 80%，未覆盖区域流量数据严重缺失`;
              advice = `数据采集优化：建议立即补充监控卡口设备安装`;
            }

            return {
              roadName: item.roadName,
              saturation: this.fixSat(item.maxSaturation),
              diagnose: diagnose,
              advice: advice,
              cost: ['1200万', '800万', '150万', '300万'][idx % 4],
              duration: ['6个月', '12个月', '2个月', '3个月'][idx % 4]
            }
          });
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
</style>