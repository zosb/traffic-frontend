<template>
  <div class="role-container">
    <el-card shadow="never" body-style="padding: 15px;">
      <el-row type="flex" align="middle">
        <el-col :span="18">
          <span style="font-weight: bold; margin-right: 15px;"><i class="el-icon-time"></i> 监控时空设定：</span>
          <el-date-picker v-model="currDate" type="date" value-format="yyyy-MM-dd" placeholder="选择日期" size="small" style="width: 150px;"></el-date-picker>
          <el-select v-model="currHour" placeholder="时段" size="small" style="width: 100px; margin-left: 10px;">
            <el-option v-for="h in 24" :key="h-1" :label="(h-1)+'点'" :value="h-1"></el-option>
          </el-select>
          <el-button type="primary" icon="el-icon-refresh" size="small" @click="loadData" style="margin-left: 15px;">刷新历史态势</el-button>
        </el-col>
      </el-row>
    </el-card>

    <el-row :gutter="24" style="margin-top: 24px;">
      <el-col :span="15">
        <el-card shadow="hover" :body-style="{ padding: '0px' }">
          <div slot="header"><span>🗺️ 交通流量热力分布图 (GIS可视化) </span></div>
          <div ref="heatmapChart" style="height: 660px;"></div>
        </el-card>
      </el-col>

      <el-col :span="9">

        <el-card shadow="always" style="margin-bottom: 20px; border-top: 3px solid #F56C6C;">
          <div slot="header" class="clearfix">
            <span style="font-weight: bold; color: #303133;">
              <i class="el-icon-video-camera-solid" style="color: #F56C6C; margin-right: 5px;"></i>
              实时路况监控流
            </span>
            <el-tag size="mini" type="danger" effect="dark" style="float: right; border-radius: 20px;">
              <i class="el-icon-loading" style="margin-right:3px" v-if="realTimeList.length > 0"></i> Live
            </el-tag>
          </div>

          <el-table :data="realTimeList" style="width: 100%" size="mini" height="200" :show-header="true">
            <el-table-column prop="windowEnd" label="更新时间" width="90" align="center">
              <template slot-scope="scope">
                <span style="color: #909399; font-size: 12px;">{{ formatTime(scope.row.windowEnd) }}</span>
              </template>
            </el-table-column>
            <el-table-column prop="roadName" label="监测路段" show-overflow-tooltip></el-table-column>
            <el-table-column label="实时流量" width="70" align="center">
              <template slot-scope="scope">
                <span style="font-weight: bold; color: #409EFF">{{ scope.row.vehicleCount }}</span>
              </template>
            </el-table-column>
            <el-table-column label="平均车速" width="70" align="center">
              <template slot-scope="scope">
                <span :style="{fontWeight: 'bold', color: scope.row.avgSpeed < 20 ? '#F56C6C' : '#67C23A'}">
                  {{ scope.row.avgSpeed.toFixed(1) }}
                </span>
              </template>
            </el-table-column>
          </el-table>
        </el-card>
        <el-card shadow="hover">
          <div slot="header">
            <span>🚦 历史时段拥堵排行 </span>
            <el-tag size="mini" type="info" style="float: right;">离线统计</el-tag>
          </div>
          <el-table :data="congestionList" height="300" stripe style="width: 100%">
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
      timer: null
    }
  },
  mounted() {
    this.loadData();
    this.fetchRealTimeData();
    this.timer = setInterval(this.fetchRealTimeData, 3000);
  },
  beforeDestroy() {
    if (this.chartInst) this.chartInst.dispose();

    if (this.timer) {
      clearInterval(this.timer);
    }
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
          .then(res => {
            this.realTimeList = res.data.slice(0, 50);
          })
          .catch(err => console.error("Realtime data fetch failed:", err));
    },
    formatTime(datetime) {
      if(!datetime) return '--';
      return datetime.length > 11 ? datetime.substring(11, 19) : datetime;
    },

    loadData() {
      axios.get(`http://localhost:8080/api/monitor/rank?date=${this.currDate}&hour=${this.currHour}`)
          .then(res => this.congestionList = res.data);
      axios.get(`http://localhost:8080/api/monitor/heatmap?date=${this.currDate}&hour=${this.currHour}`)
          .then(res => this.initHeatmap(res.data));
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
        amap: {
          center: [116.405285, 39.904989],
          zoom: 11,
          roam: true,
          renderOnMoving: false,
          echartsLayerZIndex: 2000
        },
        tooltip: { trigger: 'item' },
        visualMap: {
          min: 0,
          max: 3000,
          calculable: true,
          orient: 'vertical',
          left: 'right',
          bottom: '20',
          inRange: {
            color: ['#313695', '#4575b4', '#74add1', '#abd9e9', '#e0f3f8', '#ffffbf', '#fee090', '#fdae61', '#f46d43', '#d73027', '#a50026']
          },
          textStyle: { color: '#333' }
        },
        series: [{
          name: '交通流量热力',
          type: 'heatmap',
          coordinateSystem: 'amap',
          data: points,
          pointSize: 15,
          blurSize: 20,
          opacity: 0.7
        }]
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
.el-row {
  margin-bottom: 15px;
}
.el-row:last-child {
  margin-bottom: 0;
}
::v-deep .el-table--mini th, ::v-deep .el-table--mini td {
  padding: 3px 0;
}
</style>