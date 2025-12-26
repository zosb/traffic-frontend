<template>
  <div class="login-container">
    <div class="login-box">
      <div class="login-header">
        <i class="el-icon-data-analysis" style="font-size: 48px; color: #409EFF; margin-bottom: 15px;"></i>
        <h2 class="system-title">城市交通流量分析平台</h2>
        <p class="sub-title">Urban Traffic Analysis Platform</p>
      </div>

      <div class="role-select">
        <div class="role-card manager" @click="handleLogin('manager')">
          <div class="icon-wrapper"><i class="el-icon-monitor"></i></div>
          <div class="info">
            <h3>交通管理人员</h3>
            <span>实时监控 / 疏导 / 执法</span>
          </div>
          <i class="el-icon-right arrow"></i>
        </div>

        <div class="role-card planner" @click="handleLogin('planner')">
          <div class="icon-wrapper"><i class="el-icon-office-building"></i></div>
          <div class="info">
            <h3>交通规划设计师</h3>
            <span>路网规划 / 改造 / 信号优化</span>
          </div>
          <i class="el-icon-right arrow"></i>
        </div>

        <div class="role-card analyst" @click="handleLogin('analyst')">
          <div class="icon-wrapper"><i class="el-icon-data-line"></i></div>
          <div class="info">
            <h3>交通数据分析师</h3>
            <span>数据清洗 / 质量管控 / AI归因</span>
          </div>
          <i class="el-icon-right arrow"></i>
        </div>
      </div>
    </div>
    <div class="footer-copyright">© 2025 Smart City Traffic System</div>
  </div>
</template>

<script>
export default {
  name: 'Login',
  methods: {
    handleLogin(role) {
      // 1. 模拟登录，将角色存入 localStorage
      localStorage.setItem('userRole', role);

      const roleNames = {
        'manager': '交通管理人员',
        'planner': '交通规划设计师',
        'analyst': '交通数据分析师'
      };

      // 2. 提示并跳转
      this.$message.success(`欢迎回来，${roleNames[role]}`);
      setTimeout(() => {
        this.$router.push('/dashboard'); // 跳转到主页
      }, 500);
    }
  }
}
</script>

<style scoped>
.login-container {
  height: 100vh;
  width: 100vw;
  background: linear-gradient(135deg, #2c3e50 0%, #4ca1af 100%);
  display: flex;
  flex-direction: column;
  justify-content: center;
  align-items: center;
  position: relative;
}

.login-box {
  width: 450px;
  background: rgba(255, 255, 255, 0.95);
  border-radius: 16px;
  padding: 40px;
  box-shadow: 0 15px 35px rgba(0,0,0,0.2);
  backdrop-filter: blur(10px);
}

.login-header {
  text-align: center;
  margin-bottom: 35px;
}
.system-title {
  margin: 0;
  font-size: 24px;
  color: #303133;
  letter-spacing: 1px;
}
.sub-title {
  margin-top: 8px;
  color: #909399;
  font-size: 14px;
  text-transform: uppercase;
  letter-spacing: 2px;
}

.role-card {
  display: flex;
  align-items: center;
  padding: 18px;
  border: 1px solid #EBEEF5;
  border-radius: 10px;
  margin-bottom: 18px;
  cursor: pointer;
  transition: all 0.3s cubic-bezier(0.25, 0.8, 0.25, 1);
  background: #fff;
}

.role-card:hover {
  transform: translateY(-3px) translateX(5px);
  box-shadow: 0 5px 15px rgba(0,0,0,0.1);
}

/* 角色特定颜色悬停效果 */
.role-card.manager:hover { border-color: #409EFF; }
.role-card.manager:hover .icon-wrapper { background: #409EFF; color: #fff; }

.role-card.planner:hover { border-color: #E6A23C; }
.role-card.planner:hover .icon-wrapper { background: #E6A23C; color: #fff; }

.role-card.analyst:hover { border-color: #67C23A; }
.role-card.analyst:hover .icon-wrapper { background: #67C23A; color: #fff; }

.icon-wrapper {
  width: 46px;
  height: 46px;
  border-radius: 10px;
  background: #F2F6FC;
  color: #606266;
  display: flex;
  justify-content: center;
  align-items: center;
  font-size: 22px;
  margin-right: 18px;
  transition: all 0.3s;
}

.info h3 { margin: 0 0 6px; font-size: 16px; color: #303133; }
.info span { font-size: 12px; color: #909399; }
.arrow { margin-left: auto; color: #C0C4CC; }

.footer-copyright {
  position: absolute;
  bottom: 20px;
  color: rgba(255,255,255,0.6);
  font-size: 12px;
}
</style>