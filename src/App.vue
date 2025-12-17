<template>
  <div id="app">
    <div class="nav-header">
      <div class="logo-container">
        <i class="el-icon-data-analysis logo-icon"></i>
        <span class="title"> 城市交通流量分析平台 </span>
      </div>

      <div class="right-panel">
        <div class="info-item weather">
          <i :class="weatherIcon" :style="{ color: weatherColor }"></i>
          <span>北京 {{ weatherStatus }} {{ currentTemp }}°C</span>
        </div>

        <div class="divider"></div>

        <div class="info-item time">
          <i class="el-icon-time"></i>
          <span>{{ currentTime }}</span>
        </div>
      </div>
    </div>

    <div class="main-content">
      <el-tabs v-model="activeRole" type="border-card" class="role-tabs">
        <el-tab-pane name="manager">
          <span slot="label"><i class="el-icon-monitor"></i> 交通管理人员 </span>
          <keep-alive><traffic-manager v-if="activeRole === 'manager'"></traffic-manager></keep-alive>
        </el-tab-pane>

        <el-tab-pane name="planner">
          <span slot="label"><i class="el-icon-office-building"></i> 交通规划设计师 </span>
          <keep-alive><traffic-planner v-if="activeRole === 'planner'"></traffic-planner></keep-alive>
        </el-tab-pane>

        <el-tab-pane name="analyst">
          <span slot="label"><i class="el-icon-data-line"></i> 交通数据分析师 </span>
          <keep-alive><data-analyst v-if="activeRole === 'analyst'"></data-analyst></keep-alive>
        </el-tab-pane>
      </el-tabs>
    </div>
  </div>
</template>

<script>
import TrafficManager from './components/TrafficManager.vue'
import TrafficPlanner from './components/TrafficPlanner.vue'
import DataAnalyst from './components/DataAnalyst.vue'
import axios from 'axios'

export default {
  name: 'App',
  components: {
    TrafficManager,
    TrafficPlanner,
    DataAnalyst
  },
  data() {
    return {
      activeRole: 'manager',
      currentTime: '',
      timer: null,
      weatherTimer: null,

      weatherIcon: 'el-icon-loading',
      weatherStatus: '加载中..',
      currentTemp: '--',
      weatherColor: '#fff'
    }
  },
  mounted() {
    this.updateTime();
    this.timer = setInterval(this.updateTime, 1000);

    this.fetchRealWeather();
    this.weatherTimer = setInterval(this.fetchRealWeather, 600000);
  },
  beforeDestroy() {
    if (this.timer) clearInterval(this.timer);
    if (this.weatherTimer) clearInterval(this.weatherTimer);
  },
  methods: {
    updateTime() {
      const now = new Date();
      const year = now.getFullYear();
      const month = String(now.getMonth() + 1).padStart(2, '0');
      const day = String(now.getDate()).padStart(2, '0');
      const hour = String(now.getHours()).padStart(2, '0');
      const minute = String(now.getMinutes()).padStart(2, '0');
      const second = String(now.getSeconds()).padStart(2, '0');
      this.currentTime = `${year}-${month}-${day} ${hour}:${minute}:${second}`;
    },

    fetchRealWeather() {
      const url = 'https://api.open-meteo.com/v1/forecast?latitude=39.9042&longitude=116.4074&current_weather=true&timezone=Asia%2FShanghai';

      axios.get(url).then(res => {
        if (res.data && res.data.current_weather) {
          const w = res.data.current_weather;
          this.currentTemp = w.temperature;
          this.parseWeatherCode(w.weathercode);
        }
      }).catch(err => {
        console.error("天气请求失败", err);
        this.weatherStatus = '多云';
        this.currentTemp = '24';
        this.weatherIcon = 'el-icon-cloudy';
      });
    },

    parseWeatherCode(code) {

      if (code === 0) {
        this.weatherStatus = '晴';
        this.weatherIcon = 'el-icon-sunny';
        this.weatherColor = '#ffcc00';
      }

      else if ([1, 2, 3].includes(code)) {
        this.weatherStatus = '多云';
        this.weatherIcon = 'el-icon-cloudy';
        this.weatherColor = '#ffffff';
      }

      else if (code >= 51 && code <= 67) {
        this.weatherStatus = '雨';
        this.weatherIcon = 'el-icon-light-rain';
        this.weatherColor = '#409EFF';
      }

      else if (code >= 71 && code <= 77) {
        this.weatherStatus = '雪';
        this.weatherIcon = 'el-icon-light-rain';
        this.weatherColor = '#E4E7ED';
      }
      else {
        this.weatherStatus = '阴';
        this.weatherIcon = 'el-icon-cloudy-and-sunny';
        this.weatherColor = '#ffffff';
      }
    }
  }
}
</script>

<style>


html, body {
  margin: 0;
  padding: 0;
  width: 100%;
  height: 100%;
  overflow: hidden;
  font-family: "Helvetica Neue", Helvetica, "PingFang SC", "Microsoft YaHei", Arial, sans-serif;
  background-color: #f5f7fa;
}

#app {
  display: flex;
  flex-direction: column;
  height: 100vh;
}


.nav-header {
  flex-shrink: 0;
  background: linear-gradient(to right, #2c3e50, #4ca1af);
  color: #fff;
  height: 64px;
  display: flex;
  align-items: center;
  justify-content: space-between;
  padding: 0 30px;
  box-shadow: 0 2px 8px rgba(0,0,0,0.15);
  z-index: 1000;
}

.logo-container {
  display: flex;
  align-items: center;
}

.logo-icon {
  font-size: 26px;
  margin-right: 12px;
}

.title {
  font-size: 22px;
  font-weight: bold;
  letter-spacing: 1px;
}


.right-panel {
  display: flex;
  align-items: center;
  font-size: 14px;
  background: rgba(255, 255, 255, 0.1);
  padding: 8px 20px;
  border-radius: 20px;
  border: 1px solid rgba(255,255,255,0.1);
}

.info-item {
  display: flex;
  align-items: center;
  color: #fff;
}

.info-item i {
  margin-right: 6px;
  font-size: 16px;
}

.weather {
  margin-right: 15px;
}

.weather span {
  font-weight: 500;
}

.time span {
  font-family: Consolas, Monaco, monospace;
  font-weight: 600;
  letter-spacing: 0.5px;
}

.divider {
  width: 1px;
  height: 14px;
  background-color: rgba(255,255,255,0.4);
  margin: 0 15px;
}

.main-content {
  flex: 1;
  padding: 20px;
  overflow: hidden;
  box-sizing: border-box;
}

.role-tabs {
  height: 100%;
  display: flex;
  flex-direction: column;
  border: none;
  box-shadow: 0 4px 12px rgba(0,0,0,0.05);
}

.role-tabs .el-tabs__header {
  margin: 0 !important;
  background: #fff;
  flex-shrink: 0;
}

.role-tabs .el-tabs__content {
  flex: 1;
  overflow-y: auto;
  padding: 24px !important;
  background-color: #fff;
}

.role-container {
  padding-bottom: 40px;
  min-height: 100%;
}
</style>