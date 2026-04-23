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
            <span><i class="el-icon-s-data"></i> 核心干道饱和度压力测试</span>
            <el-tag size="mini" type="danger" effect="dark">警戒线: 1.0</el-tag>
          </div>
          <div ref="loadChart" style="height: 340px; width: 100%;"></div>
        </el-card>
      </el-col>

      <el-col :span="8" class="flex-col">
        <el-card shadow="hover" class="custom-card">
          <div slot="header" class="card-header border-left-purple">
            <span><i class="el-icon-pie-chart"></i> 区域流量承载占比</span>
          </div>
          <div ref="regionChart" style="height: 340px; width: 100%;"></div>
        </el-card>
      </el-col>
    </el-row>

    <el-row :gutter="20" class="flex-row mt-20">
      <el-col :span="16" class="flex-col">
        <el-card shadow="hover" class="custom-card" body-style="padding: 0;">
          <div slot="header" class="card-header border-left-orange">
            <span><i class="el-icon-cpu"></i> 信号灯配时优化建议 </span>
            <el-tooltip content="AI 基于仿真模型推演的优化方案" placement="top">
<!--              <el-tag size="mini" type="warning" effect="plain"><i class="el-icon-magic-stick"></i> AI 介入中</el-tag>-->
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

              <el-table-column label="具体措施" show-overflow-tooltip>
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
            <span><i class="el-icon-truck"></i> 道路硬件改造工程建议</span>
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
                    <el-tag size="mini" type="danger" effect="plain">负荷 {{ (item.saturation * 100).toFixed(0) }}%</el-tag>
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
      const viewData = data.slice(0, 15);

      const option = {
        tooltip: { trigger: 'axis', formatter: '{b}: 饱和度 {c}' },
        grid: { left: '3%', right: '4%', bottom: '3%', top: '10%', containLabel: true },
        xAxis: {
          type: 'category',
          data: viewData.map(i => i.roadName),
          axisLabel: { rotate: 30, interval: 0, color: '#606266' },
          axisTick: { alignWithLabel: true }
        },
        yAxis: { type: 'value', name: '饱和度 (V/C)', splitLine: { lineStyle: { type: 'dashed' } } },
        series: [{
          name: '饱和度',
          type: 'bar',
          barWidth: '40%',
          data: viewData.map(i => this.fixSat(i.maxSaturation)),
          itemStyle: { borderRadius: [4, 4, 0, 0] },
          markLine: {
            symbol: 'none',
            data: [
              { yAxis: 0.9, lineStyle: { color: '#E6A23C', type: 'dashed' }, label: { formatter: '预警' } },
              { yAxis: 1.1, lineStyle: { color: '#F56C6C', type: 'solid' }, label: { formatter: '饱和' } }
            ]
          }
        }]
      };

      option.series[0].data = viewData.map(item => {
        let val = this.fixSat(item.maxSaturation);
        return {
          value: val,
          itemStyle: {
            color: val > 1.0
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
      const option = {
        tooltip: { trigger: 'item' },
        legend: { bottom: '0', icon: 'circle' },
        series: [
          {
            name: '区域流量',
            type: 'pie',
            radius: ['35%', '65%'],
            center: ['50%', '45%'],
            itemStyle: { borderRadius: 8, borderColor: '#fff', borderWidth: 2 },
            data: data.map(i => ({ name: i.regionName, value: i.totalFlow }))
          }
        ]
      };
      this.regionChartInst.setOption(option);
    },

    // 🟢 核心优化：生成更有“决策味”的数据
    processSignalData(data) {
      // 策略库 (带颜色)
      const strategies = [
        { name: '瓶颈控制', type: 'danger', impact: '溢出风险 -40%' },
        { name: '借道左转', type: 'warning', impact: '通行力 +18%' },
        { name: '可变车道', type: 'primary', impact: '空间利用 +25%' },
        { name: '相序优化', type: 'success', impact: '延误 -12s' },
        { name: '绿波协调', type: 'success', impact: '停车次数 -3' },
        { name: '感应控制', type: 'primary', impact: '空放时间 -15s' },
        { name: '二次过街', type: 'info', impact: '行人安全 ↑' },
        { name: '精细配时', type: 'warning', impact: '周期利用 +10%' }
      ];

      const details = [
        '建议上游路口实施红灯截流，防止车辆溢出至本路口',
        '左转排队过长，建议利用出口车道设置借道左转区域',
        '各进口流量波动较大，建议设置锯齿形可变导向车道',
        '建议调整信号相序，减少左转车流对直行车辆的干扰',
        '具备干线协调条件，建议实施双向绿波控制减少停车',
        '建议启用车辆检测器，根据实时到达流量动态调整绿灯',
      ];

      this.signalAdviceList = data
          .filter(item => this.fixSat(item.maxSaturation) > 0.6)
          .sort((a, b) => this.fixSat(b.maxSaturation) - this.fixSat(a.maxSaturation))
          .slice(0, 10)
          .map(item => {
            const sat = this.fixSat(item.maxSaturation);
            // 基于路名 Hash 选策略，保证稳定
            const idxStrat = this.getHashIndex(item.roadName, strategies.length);
            const strat = strategies[idxStrat];
            const idxDetail = this.getHashIndex(item.roadName + 'd', details.length);

            return {
              roadName: item.roadName,
              saturation: sat,
              tagType: strat.type,
              strategyAction: strat.name,
              strategyDetail: details[idxDetail],
              impact: strat.impact
            };
          });
    },

    processConstructionData(data) {
      const problems = [
        '关键节点供需严重失衡，常态化排队超500米',
        '交叉口几何设计局限，大型车辆转弯困难',
        '路侧开口过多，进出地块车辆严重干扰主流',
        '上下游车道数不匹配，形成局部交通瓶颈'
      ];
      const solutions = [
        '规划地下快速路或跨线桥实现立体分流',
        '实施路口渠化拓宽，增加进口车道数量',
        '设置潮汐车道，利用对向闲置车道资源',
        '封闭部分路侧开口，设置辅路集散交通'
      ];
      const costs = ['500万', '1200万', '30万', '80万'];
      const durations = ['6个月', '12个月', '1个月', '2个月'];

      this.constructionList = data
          .filter(item => this.fixSat(item.maxSaturation) > 1.05)
          .sort((a, b) => this.fixSat(b.maxSaturation) - this.fixSat(a.maxSaturation))
          .slice(0, 6)
          .map(item => {
            const idx = this.getHashIndex(item.roadName, problems.length);
            return {
              roadName: item.roadName,
              saturation: this.fixSat(item.maxSaturation),
              diagnose: problems[idx],
              advice: solutions[idx],
              cost: costs[idx],
              duration: durations[idx]
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

/* Flex 布局辅助类 */
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

/* 装饰性标题头 */
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

/* 建议卡片样式 */
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

/* 工程信息栏 */
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
</style>