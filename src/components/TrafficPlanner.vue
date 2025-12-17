<template>
  <div class="role-container">
    <el-row :gutter="24">
      <el-col :span="24">
        <el-card shadow="hover">
          <div slot="header"><span>🏗️ 路网负荷分析与扩建建议 </span></div>
          <div ref="barChart" style="height: 350px;"></div>
        </el-card>
      </el-col>
    </el-row>
    <el-row :gutter="24" style="margin-top: 24px;">
      <el-col :span="14">
        <el-card shadow="hover">
          <div slot="header"><span>🏙️ 区域流量结构分布 </span></div>
          <div ref="pieChart" style="height: 320px;"></div>
        </el-card>
      </el-col>
      <el-col :span="10">
        <el-card shadow="hover" style="height: 422px; overflow-y: auto;">
          <div slot="header"><span>🚦 设施优化建议 </span></div>
          <el-timeline style="padding-top: 10px;">
            <el-timeline-item v-for="(road, index) in topCongested" :key="index" :timestamp="'优先级 '+(index+1)" placement="top" :color="index===0?'#F56C6C':'#E6A23C'">
              <h4>{{ road.roadName }}</h4>
              <p style="font-size: 13px; color: #666;">饱和度 {{ road.maxSaturation }}，建议{{ road.maxSaturation > 1.5 ? '立体化改造' : '优化信号灯' }}。</p>
            </el-timeline-item>
          </el-timeline>
        </el-card>
      </el-col>
    </el-row>
  </div>
</template>

<script>
import * as echarts from 'echarts';
import axios from 'axios';
export default {
  data() { return { topCongested: [], barInst: null, pieInst: null } },
  mounted() { this.$nextTick(() => { this.initCharts(); }); },
  beforeDestroy() { if (this.barInst) this.barInst.dispose(); if (this.pieInst) this.pieInst.dispose(); },
  methods: {
    initCharts() {
      axios.get('http://localhost:8080/api/planning/advice?date=2025-11-04').then(res => {
        if (!this.$refs.barChart) return;
        this.topCongested = res.data.slice(0, 4);
        this.barInst = echarts.init(this.$refs.barChart);
        this.barInst.setOption({
          tooltip: { trigger: 'axis' },
          xAxis: { type: 'category', data: res.data.slice(0,15).map(i => i.roadName), axisLabel: { rotate: 30 } },
          yAxis: { type: 'value', name: '饱和度' },
          series: [{ type: 'bar', data: res.data.slice(0,15).map(i => i.maxSaturation), itemStyle: { color: '#409EFF' },
            markLine: { data: [{ yAxis: 1.0,name: '警戒线'}],lineStyle: { color: '#CC0000', width: 2, type: 'dashed'},symbol: ['none', 'none'] } }]
        });
      });

      axios.get('http://localhost:8080/api/planning/region?date=2025-11-04').then(res => {
        if (!this.$refs.pieChart) return;
        this.pieInst = echarts.init(this.$refs.pieChart);
        this.pieInst.setOption({
          tooltip: { trigger: 'item' }, legend: { top: 'center', left: 'left', orient: 'vertical' },
          series: [{ type: 'pie', radius: ['40%', '70%'], center: ['65%', '50%'], data: res.data.map(i => ({ name: i.regionName, value: i.totalFlow })) }]
        });
      });
    }
  }
}
</script>