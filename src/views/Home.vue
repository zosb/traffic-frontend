<template>
  <div class="home-wrapper">
    <div class="nav-header">
      <div class="logo-container">
        <i class="el-icon-data-analysis logo-icon"></i>
        <span class="title"> 城市交通流量分析平台 </span>
      </div>

      <div class="right-panel">
        <div class="current-role-tag">
          <i class="el-icon-user-solid"></i> {{ roleName }}
        </div>

        <div class="divider"></div>

        <div class="info-item weather">
          <i :class="weatherIcon" :style="{ color: weatherColor }"></i>
          <span>北京 {{ weatherStatus }} {{ currentTemp }}°C</span>
        </div>

        <div class="divider"></div>

        <div class="info-item time">
          <i class="el-icon-time"></i>
          <span>{{ currentTime }}</span>
        </div>

        <el-button
            type="text"
            icon="el-icon-switch-button"
            class="logout-btn"
            @click="logout">退出
        </el-button>
      </div>
    </div>

    <div class="main-content">
      <transition name="fade" mode="out-in">
        <keep-alive>
          <traffic-manager v-if="currentRole === 'manager'" />
          <traffic-planner v-else-if="currentRole === 'planner'" />
          <data-analyst v-else-if="currentRole === 'analyst'" />
        </keep-alive>
      </transition>
    </div>
  </div>
</template>

<script>

import TrafficManager from '../components/TrafficManager.vue'
import TrafficPlanner from '../components/TrafficPlanner.vue'
import DataAnalyst from '../components/DataAnalyst.vue'
import axios from 'axios'

export default {
  name: 'Home',
  components: {
    TrafficManager,
    TrafficPlanner,
    DataAnalyst
  },
  data() {
    return {
      currentRole: '',
      currentTime: '',
      timer: null,
      weatherTimer: null,
      weatherIcon: 'el-icon-loading',
      weatherStatus: '加载中',
      currentTemp: '--',
      weatherColor: '#fff'
    }
  },
  computed: {
    roleName() {
      const map = {
        'manager': '交通管理人员',
        'planner': '交通规划设计师',
        'analyst': '交通数据分析师'
      };
      return map[this.currentRole] || '访客';
    }
  },
  created() {
    const role = localStorage.getItem('userRole');
    if (!role) {
      this.$router.push('/login');
    } else {
      this.currentRole = role;
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
    logout() {
      this.$confirm('确定要退出当前账号吗?', '提示', {
        confirmButtonText: '确定',
        cancelButtonText: '取消',
        type: 'warning'
      }).then(() => {
        localStorage.removeItem('userRole');
        this.$router.push('/login');
      }).catch(() => {});
    },
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
        this.weatherStatus = '晴'; this.weatherIcon = 'el-icon-sunny'; this.weatherColor = '#ffcc00';
      } else if ([1, 2, 3].includes(code)) {
        this.weatherStatus = '多云'; this.weatherIcon = 'el-icon-cloudy'; this.weatherColor = '#ffffff';
      } else if (code >= 51 && code <= 67) {
        this.weatherStatus = '雨'; this.weatherIcon = 'el-icon-light-rain'; this.weatherColor = '#409EFF';
      } else if (code >= 71 && code <= 77) {
        this.weatherStatus = '雪'; this.weatherIcon = 'el-icon-light-rain'; this.weatherColor = '#E4E7ED';
      } else {
        this.weatherStatus = '阴'; this.weatherIcon = 'el-icon-cloudy-and-sunny'; this.weatherColor = '#ffffff';
      }
    }
  }
}
</script>

<style scoped>

.home-wrapper {
  display: flex;
  flex-direction: column;
  height: 100vh;
  overflow: hidden;
  background-color: #f5f7fa;
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

.logo-container { display: flex; align-items: center; }
.logo-icon { font-size: 26px; margin-right: 12px; }
.title { font-size: 22px; font-weight: bold; letter-spacing: 1px; }

.right-panel {
  display: flex;
  align-items: center;
  font-size: 14px;
  background: rgba(255, 255, 255, 0.1);
  padding: 8px 20px;
  border-radius: 20px;
  border: 1px solid rgba(255,255,255,0.1);
}

.info-item { display: flex; align-items: center; color: #fff; }
.info-item i { margin-right: 6px; font-size: 16px; }
.weather { margin-right: 15px; }
.time span { font-family: Consolas, Monaco, monospace; font-weight: 600; }
.divider { width: 1px; height: 14px; background-color: rgba(255,255,255,0.4); margin: 0 15px; }

.current-role-tag {
  background: rgba(0,0,0,0.2);
  padding: 4px 10px;
  border-radius: 4px;
  font-size: 12px;
  margin-right: 15px;
}

.logout-btn {
  color: #ffdddd !important;
  margin-left: 20px;
  padding: 0;
}
.logout-btn:hover { color: #ff4949 !important; }

.main-content {
  flex: 1;
  overflow-y: auto;
}

.fade-enter-active, .fade-leave-active { transition: opacity 0.3s; }
.fade-enter, .fade-leave-to { opacity: 0; }
</style>