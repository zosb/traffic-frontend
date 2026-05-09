<template>
  <div class="role-container">
    <el-card shadow="never" body-style="padding: 15px;" class="mb-20">
      <el-row type="flex" align="middle">
        <el-col :span="24">
          <span style="font-weight: bold; margin-right: 15px;"><i class="el-icon-time"></i> 监控时空设定：</span>
          <el-date-picker v-model="currDate" type="date" value-format="yyyy-MM-dd" placeholder="选择日期" size="small" style="width: 150px;"></el-date-picker>
          <el-select v-model="currHour" placeholder="时段" size="small" style="width: 100px; margin-left: 10px;">
            <el-option v-for="h in 24" :key="h-1" :label="(h-1)+'点'" :value="h-1"></el-option>
          </el-select>
          <el-button type="primary" icon="el-icon-refresh" size="small" @click="loadData" style="margin-left: 15px;">刷新历史态势</el-button>
        </el-col>
      </el-row>
    </el-card>

    <el-row :gutter="20" class="flex-row">
      <el-col :span="12" class="flex-col">
        <el-card shadow="hover" class="custom-card heatmap-card">
          <div slot="header"><span>🗺️ 交通流量热力分布图 (GIS可视化) </span></div>
          <div ref="heatmapChart" class="chart-container"></div>
        </el-card>
      </el-col>

      <el-col :span="6" class="flex-col">
        <el-card shadow="always" class="custom-card mb-20" style="border-top: 3px solid #409EFF;">
          <div slot="header" class="clearfix">
            <span style="font-weight: bold; color: #303133;"><i class="el-icon-video-camera-solid" style="color: #409EFF; margin-right: 5px;"></i> 实时路况监控流</span>
            <el-tag size="mini" type="primary" effect="dark" style="float: right; border-radius: 20px;"><i class="el-icon-loading" style="margin-right:3px" v-if="realTimeList.length > 0"></i> Live</el-tag>
          </div>
          <el-table :data="realTimeList" style="width: 100%; flex: 1;" size="mini" height="100%">
            <el-table-column prop="windowEnd" label="时间" width="70" align="center">
              <template slot-scope="scope"><span style="color: #909399; font-size: 12px;">{{ formatTime(scope.row.windowEnd) }}</span></template>
            </el-table-column>
            <el-table-column prop="roadName" label="路段" show-overflow-tooltip></el-table-column>
            <el-table-column label="流量" width="60" align="center">
              <template slot-scope="scope"><span style="font-weight: bold; color: #409EFF">{{ scope.row.vehicleCount }}</span></template>
            </el-table-column>
            <el-table-column label="车速" width="60" align="center">
              <template slot-scope="scope">
                <span :style="{fontWeight: 'bold', color: scope.row.avgSpeed < 20 ? '#F56C6C' : '#67C23A'}">{{ scope.row.avgSpeed.toFixed(1) }}</span>
              </template>
            </el-table-column>
          </el-table>
        </el-card>

        <el-card shadow="hover" class="custom-card">
          <div slot="header">
            <span>🚦 历史时段拥堵排行 </span>
            <el-tag size="mini" type="info" style="float: right;">离线统计</el-tag>
          </div>
          <el-table :data="congestionList" stripe style="width: 100%; flex: 1;" size="mini" height="100%">
            <el-table-column type="index" label="排名" width="50" align="center"></el-table-column>
            <el-table-column prop="roadName" label="路段名称" show-overflow-tooltip></el-table-column>
            <el-table-column prop="avgSpeed" label="均速" width="80" align="center">
              <template slot-scope="scope">
                <el-tag :type="scope.row.avgSpeed < 20 ? 'danger' : 'warning'" size="small" effect="dark">{{ scope.row.avgSpeed.toFixed(1) }}</el-tag>
              </template>
            </el-table-column>
          </el-table>
        </el-card>
      </el-col>

      <el-col :span="6" class="flex-col">
        <el-card shadow="hover" class="custom-card ai-terminal" style="border-top: 3px solid #F56C6C;">
          <div slot="header">
            <span style="font-weight: bold; color: #F56C6C;"><i class="el-icon-cpu"></i> 智能调度与归因终端 </span>
            <el-button type="text" style="float: right; padding: 3px 0;" @click="generateAiReport"><i class="el-icon-refresh"></i> 重新诊断</el-button>
          </div>

          <div class="ai-report-container" v-loading="isAiThinking" element-loading-text="AI 大模型正在读取底层数仓并生成诊断..." element-loading-background="rgba(255, 255, 255, 0.9)">
            <div v-if="!isAiThinking" class="ai-layout-wrapper">

              <div class="ai-scroll-area">
                <div v-if="aiReportContent" v-html="aiReportContent" class="ai-dynamic-content"></div>
                <div v-else style="color:#909399; text-align:center; padding: 40px 0;">
                  <i class="el-icon-cpu" style="font-size: 40px; margin-bottom: 10px; color: #DCDFE6;"></i><br>
                  等待 AI 大模型接入...
                </div>
              </div>

              <div class="ai-fixed-bottom">
                <el-divider><i class="el-icon-s-opportunity"></i> 指挥中心方案推演 </el-divider>
                <div class="ai-section" style="margin-bottom: 0;">
                  <div style="margin-bottom: 10px; font-size: 13px; font-weight: bold; color: #303133;">请选择干预指令下发：</div>
                  <el-radio-group v-model="actionType" size="small" style="display: flex; flex-direction: column; gap: 10px;">
                    <el-radio-button label="限行">方案 A：外围临时限行截流</el-radio-button>
                    <el-radio-button label="疏导">方案 B：增加交警现场疏导</el-radio-button>
                  </el-radio-group>
                  <el-button type="danger" size="small" style="width: 100%; margin-top: 15px;" @click="executeControl">下发执行指令</el-button>
                </div>

                <transition name="el-fade-in-linear">
                  <div v-if="showResult" class="ai-result-box">
                    <div style="color: #67C23A; font-weight: bold; margin-bottom: 10px;"><i class="el-icon-success"></i> 策略执行效果评估：</div>
                    <ul class="result-list">
                      <li>实施 [{{ actionType }}] 后，该路段高峰流量预计 <strong>降至合理区间</strong>。</li>
                      <li>通行效率显著改善，平均车速 <strong>预计提升 15-20%</strong>。</li>
                      <li>区域运行状态将由 <el-tag type="danger" size="mini">饱和</el-tag> 转为 <el-tag type="success" size="mini">缓解</el-tag>。</li>
                    </ul>
                  </div>
                </transition>
              </div>

            </div>
          </div>
        </el-card>
      </el-col>
    </el-row>
  </div>
</template>

<script>
import * as echarts from 'echarts';
import 'echarts-extension-amap';
import axios from 'axios';

export default {
  data() {
    return {
      currDate: this.getYesterday(),
      currHour: 18,
      congestionList: [],
      chartInst: null,
      realTimeList: [],
      timer: null,

      isAiThinking: false,
      targetRoad: {},
      aiReportContent: '',
      actionType: '限行',
      showResult: false
    }
  },
  mounted() {
    this.loadData();
    this.fetchRealTimeData();
    this.timer = setInterval(this.fetchRealTimeData, 3000);
  },
  beforeDestroy() {
    if (this.chartInst) this.chartInst.dispose();
    if (this.timer) clearInterval(this.timer);
  },
  methods: {
    getYesterday() {
      const date = new Date();
      date.setDate(date.getDate() - 1);
      const year = date.getFullYear();
      const month = String(date.getMonth() + 1).padStart(2, '0');
      const day = String(date.getDate()).padStart(2, '0');
      return `${year}-${month}-${day}`;
    },

    fetchRealTimeData() {
      axios.get('http://localhost:8080/api/monitor/realtime_list')
          .then(res => { this.realTimeList = res.data.slice(0, 50); }).catch(() => {});
    },
    formatTime(datetime) {
      if(!datetime) return '--';
      return datetime.length > 11 ? datetime.substring(11, 19) : datetime;
    },

    loadData() {
      this.showResult = false;
      this.aiReportContent = '';

      axios.get(`http://localhost:8080/api/monitor/rank?date=${this.currDate}&hour=${this.currHour}`)
          .then(res => {
            this.congestionList = res.data;
            if(this.congestionList && this.congestionList.length > 0) {
              this.targetRoad = this.congestionList[0];
              this.generateAiReport();
            } else {
              this.targetRoad = {};
            }
          });
      axios.get(`http://localhost:8080/api/monitor/heatmap?date=${this.currDate}&hour=${this.currHour}`)
          .then(res => this.initHeatmap(res.data));
    },

    generateAiReport() {
      if (!this.targetRoad || !this.targetRoad.roadName) return;

      this.isAiThinking = true;
      this.showResult = false;

      const payload = {
        roadName: this.targetRoad.roadName,
        hr: parseInt(this.currHour),
        flow: parseInt(this.targetRoad.realFlow || 500),
        speed: parseFloat(this.targetRoad.avgSpeed || 15.0),
        role: 'manager'
      };

      axios.post('http://localhost:8080/api/monitor/ai_analyze', payload)
          .then(res => {
            if (res.data && res.data.report) {
              this.aiReportContent = res.data.report;
            } else {
              throw new Error("返回数据异常");
            }
          })
          .catch(err => {
            console.error("AI 接口调用失败:", err);
            this.aiReportContent = `
            <div style="color:#F56C6C; background:#fef0f0; padding:15px; border-radius:5px; border-left:4px solid #F56C6C;">
              <i class="el-icon-circle-close"></i> <b>连接后端 AI 代理接口失败</b><br/>
              请确保 Java SpringBoot 后端正常运行，且能访问 Python 服务。
            </div>`;
          })
          .finally(() => {
            this.isAiThinking = false;
          });
    },

    executeControl() {
      this.showResult = false;
      this.$message.success("已向联动指挥中心下发干预指令...");
      setTimeout(() => {
        this.showResult = true;
      }, 600);
    },

    initHeatmap(data) {
      if (!this.$refs.heatmapChart) return;
      if (this.chartInst) this.chartInst.dispose();
      this.chartInst = echarts.init(this.$refs.heatmapChart);

      const points = data.map(item => {
        if (!item.geometryWkt) return null;
        try {
          const matches = item.geometryWkt.match(/([\d.]+) ([\d.]+)/);
          if (matches) return [parseFloat(matches[1]), parseFloat(matches[2]), item.realFlow];
        } catch(e) { return null; }
      }).filter(p => p !== null);

      const option = {
        amap: { center: [116.405285, 39.904989], zoom: 11, roam: true, renderOnMoving: false, echartsLayerZIndex: 2000 },
        tooltip: { trigger: 'item' },
        visualMap: { show: false, min: 0, max: 3000, calculable: true, inRange: { color: ['#313695', '#4575b4', '#74add1', '#abd9e9', '#e0f3f8', '#ffffbf', '#fee090', '#fdae61', '#f46d43', '#d73027', '#a50026'] } },
        series: [{ type: 'heatmap', coordinateSystem: 'amap', data: points, pointSize: 15, blurSize: 20, opacity: 0.7 }]
      };
      this.chartInst.setOption(option);
    }
  }
}
</script>

<style scoped>
.role-container {
  padding: 15px;
  background-color: #f5f7fa;
  min-height: 100vh;
  box-sizing: border-box;
}

.mb-20 { margin-bottom: 20px; }
::v-deep .el-table--mini th, ::v-deep .el-table--mini td { padding: 3px 0; }

/* 响应式对齐布局核心样式 */
.flex-row {
  display: flex;
  align-items: stretch;
  height: 680px;
}
.flex-col {
  display: flex;
  flex-direction: column;
  height: 100%;
}
.custom-card {
  border-radius: 8px;
  border: none;
  box-shadow: 0 4px 12px rgba(0, 0, 0, 0.05);
  display: flex;
  flex-direction: column;
  flex: 1;
}
.custom-card ::v-deep .el-card__body {
  flex: 1;
  display: flex;
  flex-direction: column;
  overflow: hidden;
  padding: 15px;
}
.heatmap-card ::v-deep .el-card__body {
  padding: 0;
}
.chart-container {
  flex: 1;
  width: 100%;
  height: 100%;
}

/* ⭐ AI 控制台新版上下分离架构 */
.ai-report-container {
  flex: 1;
  display: flex;
  flex-direction: column;
  height: 0; /* 强迫容器收紧，内部接管滚动 */
}

.ai-layout-wrapper {
  display: flex;
  flex-direction: column;
  height: 100%;
}

/* 上半部分：滚动文字区 */
.ai-scroll-area {
  flex: 1;
  overflow-y: auto;
  padding-right: 8px;
}

/* 定制优美的滚动条 */
.ai-scroll-area::-webkit-scrollbar { width: 5px; }
.ai-scroll-area::-webkit-scrollbar-thumb { background: #dcdfe6; border-radius: 3px; }
.ai-scroll-area::-webkit-scrollbar-track { background: transparent; }

/* 下半部分：固定控制区 */
.ai-fixed-bottom {
  flex-shrink: 0; /* 禁止被压缩 */
  background: #fff;
  padding-top: 10px;
}

.ai-terminal ::v-deep .el-divider__text {
  white-space: nowrap;
  font-size: 13px;
  padding: 0 10px;
}
.ai-dynamic-content {
  margin-bottom: 15px;
}
.ai-section {
  margin-bottom: 15px;
}
.ai-title {
  font-size: 14px;
  font-weight: bold;
  color: #303133;
  margin-bottom: 8px;
}
.ai-text {
  font-size: 13px;
  color: #606266;
  line-height: 1.6;
  background: #f4f4f5;
  padding: 10px;
  border-radius: 4px;
}
.highlight-red {
  background: #fef0f0;
  color: #F56C6C;
  border-left: 3px solid #F56C6C;
}
.ai-result-box {
  background: #f0f9eb;
  border: 1px solid #e1f3d8;
  padding: 12px;
  border-radius: 4px;
  margin-top: 15px;
}
.result-list {
  margin: 0;
  padding-left: 20px;
  font-size: 12px;
  color: #606266;
  line-height: 1.8;
}
</style>