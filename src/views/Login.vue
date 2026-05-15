<template>
  <div class="login-container">
    <ul class="bg-bubbles">
      <li v-for="i in 10" :key="i"></li>
    </ul>

    <div class="login-box">
      <div class="login-left">
        <div class="left-content">
          <i class="el-icon-odometer logo-huge"></i>
          <h2>城市交通流量<br>分析系统</h2>
          <p class="en-title">Urban Traffic Flow & Analysis System</p>
        </div>
      </div>

      <div class="login-right">
        <div class="right-header">
          <h3>欢迎登录</h3>
          <p>Sign in to your account</p>
        </div>

        <el-form :model="loginForm" :rules="rules" ref="loginForm" class="login-form" autocomplete="off">

          <el-form-item prop="username">
            <el-input
                v-model="loginForm.username"
                prefix-icon="el-icon-user"
                placeholder="请输入登录账号"
                size="large"
                clearable
                autocomplete="off"
                @keyup.enter.native="handleLogin">
            </el-input>
          </el-form-item>

          <el-tooltip v-model="capsTooltip" content="键盘大写锁定已开启" placement="right" manual>
            <el-form-item prop="password">
              <el-input
                  type="password"
                  v-model="loginForm.password"
                  prefix-icon="el-icon-lock"
                  placeholder="请输入登录密码"
                  size="large"
                  show-password
                  autocomplete="new-password"
                  @keyup.native="checkCapslock"
                  @keyup.enter.native="handleLogin">
              </el-input>
            </el-form-item>
          </el-tooltip>

          <el-form-item prop="role">
            <el-select v-model="loginForm.role" placeholder="请选择系统登录身份" size="large" style="width: 100%" @change="saveRolePreference">
              <template slot="prefix">
                <i class="el-icon-s-custom" style="margin-left: 5px; color: #c0c4cc;"></i>
              </template>
              <el-option label=" 交通管理人员 (Manager)" value="manager"></el-option>
              <el-option label=" 交通规划设计师 (Planner)" value="planner"></el-option>
              <el-option label=" 交通数据分析师 (Analyst)" value="analyst"></el-option>
            </el-select>
          </el-form-item>

          <el-form-item prop="verifyCode">
            <div class="verify-box">
              <el-input
                  v-model="loginForm.verifyCode"
                  prefix-icon="el-icon-key"
                  placeholder="请输入验证码"
                  size="large"
                  autocomplete="off"
                  @keyup.enter.native="handleLogin"
                  style="flex: 1; margin-right: 15px;">
              </el-input>
              <div class="canvas-box" @click="drawCaptcha" title="点击刷新验证码">
                <canvas ref="captchaCanvas" width="110" height="40"></canvas>
              </div>
            </div>
          </el-form-item>

          <div class="login-options">
            <el-checkbox v-model="loginForm.rememberMe">记住密码</el-checkbox>
            <span class="forgot-pwd" @click="$message.info('请联系系统管理员重置密码！')">忘记密码?</span>
          </div>

          <el-button
              type="primary"
              class="login-btn"
              size="large"
              :loading="loading"
              @click="handleLogin">
            {{ loading ? '安 全 认 证 中...' : '立 即 登 录' }}
          </el-button>
        </el-form>
      </div>
    </div>

    <div class="footer-copyright">
      Copyright © 2025-2026 城市交通流量分析平台 All Rights Reserved.
    </div>
  </div>
</template>

<script>
import axios from 'axios';
export default {
  name: 'Login',
  data() {
    return {
      loading: false,
      capsTooltip: false,
      actualCaptcha: '',
      loginForm: {
        username: '',
        password: '',
        role: 'manager',
        verifyCode: '',
        rememberMe: false
      },
      rules: {
        username: [{ required: true, message: '请输入账号', trigger: 'blur' }],
        password: [{ required: true, message: '请输入密码', trigger: 'blur' }],
        role: [{ required: true, message: '请选择登录身份', trigger: 'change' }],
        verifyCode: [{ required: true, message: '请输入验证码', trigger: 'blur' }]
      }
    };
  },
  mounted() {
    this.loginForm.username = '';
    this.loginForm.password = '';
    this.loginForm.verifyCode = '';
    this.loginForm.rememberMe = false;

    const preferredRole = localStorage.getItem('preferred_login_role') || localStorage.getItem('userRole');
    if (preferredRole) {
      this.loginForm.role = preferredRole;
    }

    this.drawCaptcha();
    this.loadRememberedAccount();
  },
  methods: {
    saveRolePreference(val) {
      localStorage.setItem('preferred_login_role', val);
    },

    checkCapslock(e) {
      const { key } = e;
      this.capsTooltip = key && key.length === 1 && key >= 'A' && key <= 'Z';
    },

    loadRememberedAccount() {
      const savedInfo = localStorage.getItem('traffic_login_info');
      if (savedInfo) {
        try {
          const info = JSON.parse(savedInfo);
          this.loginForm.username = info.username;
          this.loginForm.password = info.password;
          this.loginForm.role = info.role || this.loginForm.role;
          this.loginForm.rememberMe = true;
        } catch (e) {
          localStorage.removeItem('traffic_login_info');
        }
      }
    },

    drawCaptcha() {
      const canvas = this.$refs.captchaCanvas;
      if (!canvas) return;
      const ctx = canvas.getContext('2d');
      const width = canvas.width;
      const height = canvas.height;

      const chars = 'ABCDEFGHJKMNPQRSTWXYZabcdefhijkmnprstwxyz123456789';
      let code = '';
      for (let i = 0; i < 4; i++) {
        code += chars.charAt(Math.floor(Math.random() * chars.length));
      }
      this.actualCaptcha = code.toLowerCase();

      ctx.fillStyle = this.randomColor(180, 240);
      ctx.fillRect(0, 0, width, height);

      for (let i = 0; i < code.length; i++) {
        ctx.font = 'bold 24px Arial';
        ctx.fillStyle = this.randomColor(50, 160);
        ctx.shadowOffsetX = randomNum(-3, 3);
        ctx.shadowOffsetY = randomNum(-3, 3);
        ctx.shadowBlur = randomNum(-3, 3);
        ctx.shadowColor = 'rgba(0, 0, 0, 0.3)';

        let x = 20 * i + 15;
        let y = 25 + Math.random() * 8;
        let deg = randomNum(-30, 30);

        ctx.translate(x, y);
        ctx.rotate((deg * Math.PI) / 180);
        ctx.fillText(code[i], 0, 0);
        ctx.rotate((-deg * Math.PI) / 180);
        ctx.translate(-x, -y);
      }

      for (let i = 0; i < 5; i++) {
        ctx.strokeStyle = this.randomColor(40, 180);
        ctx.beginPath();
        ctx.moveTo(randomNum(0, width), randomNum(0, height));
        ctx.lineTo(randomNum(0, width), randomNum(0, height));
        ctx.stroke();
      }

      for (let i = 0; i < 40; i++) {
        ctx.fillStyle = this.randomColor(0, 255);
        ctx.beginPath();
        ctx.arc(randomNum(0, width), randomNum(0, height), 1, 0, 2 * Math.PI);
        ctx.fill();
      }

      function randomNum(min, max) { return Math.floor(Math.random() * (max - min) + min); }
    },

    randomColor(min, max) {
      let r = Math.floor(Math.random() * (max - min) + min);
      let g = Math.floor(Math.random() * (max - min) + min);
      let b = Math.floor(Math.random() * (max - min) + min);
      return `rgb(${r},${g},${b})`;
    },

    handleLogin() {
      this.$refs.loginForm.validate((valid) => {
        if (!valid) return false;

        if (this.loginForm.verifyCode.toLowerCase() !== this.actualCaptcha) {
          this.$message.error('验证码错误，请重新输入！');
          this.loginForm.verifyCode = '';
          this.drawCaptcha();
          return false;
        }

        this.loading = true;

        axios.post('http://localhost:8080/api/login', {
          username: this.loginForm.username,
          password: this.loginForm.password,
          role: this.loginForm.role
        }).then(res => {
          if (res.data.success) {
            const realRole = res.data.role;
            const realName = res.data.name;

            if (this.loginForm.rememberMe) {
              localStorage.setItem('traffic_login_info', JSON.stringify({
                username: this.loginForm.username,
                password: this.loginForm.password,
                role: realRole
              }));
            } else {
              localStorage.removeItem('traffic_login_info');
            }

            localStorage.setItem('userRole', realRole);
            localStorage.setItem('preferred_login_role', realRole);
            localStorage.setItem('username', this.loginForm.username);

            this.$message.success(`安全认证通过！欢迎您，${realName}`);
            this.$router.push('/dashboard');
          } else {
            this.$message.error(res.data.msg);
            this.loginForm.verifyCode = '';
            this.drawCaptcha();
            this.loading = false;
          }
        }).catch(err => {
          this.$message.error('网络请求失败，请检查后端服务是否启动');
          this.loading = false;
        });
      });
    }
  }
};
</script>

<style scoped>
.login-container {
  height: 100vh;
  width: 100vw;
  background: linear-gradient(to bottom right, #141e30, #243b55);
  display: flex;
  justify-content: center;
  align-items: center;
  position: relative;
  overflow: hidden;
}

.bg-bubbles {
  position: absolute;
  top: 0;
  left: 0;
  width: 100%;
  height: 100%;
  z-index: 0;
  overflow: hidden;
  margin: 0;
  padding: 0;
}
.bg-bubbles li {
  position: absolute;
  list-style: none;
  display: block;
  width: 40px;
  height: 40px;
  background-color: rgba(255, 255, 255, 0.08);
  bottom: -160px;
  animation: square 25s infinite;
  transition-timing-function: linear;
  border-radius: 4px;
}
.bg-bubbles li:nth-child(1) { left: 10%; }
.bg-bubbles li:nth-child(2) { left: 20%; width: 80px; height: 80px; animation-delay: 2s; animation-duration: 17s; }
.bg-bubbles li:nth-child(3) { left: 25%; animation-delay: 4s; }
.bg-bubbles li:nth-child(4) { left: 40%; width: 60px; height: 60px; animation-duration: 22s; background-color: rgba(255, 255, 255, 0.15); }
.bg-bubbles li:nth-child(5) { left: 70%; }
.bg-bubbles li:nth-child(6) { left: 80%; width: 120px; height: 120px; animation-delay: 3s; background-color: rgba(255, 255, 255, 0.04); }
.bg-bubbles li:nth-child(7) { left: 32%; width: 160px; height: 160px; animation-delay: 7s; }
.bg-bubbles li:nth-child(8) { left: 55%; width: 20px; height: 20px; animation-delay: 15s; animation-duration: 40s; }
.bg-bubbles li:nth-child(9) { left: 25%; width: 10px; height: 10px; animation-delay: 2s; animation-duration: 40s; background-color: rgba(255, 255, 255, 0.1); }
.bg-bubbles li:nth-child(10) { left: 90%; width: 160px; height: 160px; animation-delay: 11s; }

@keyframes square {
  0% { transform: translateY(0) rotate(0deg); opacity: 1; }
  100% { transform: translateY(-1000px) rotate(600deg); opacity: 0; }
}

.login-box {
  z-index: 1;
  display: flex;
  width: 850px;
  height: 500px;
  background: rgba(255, 255, 255, 0.95);
  border-radius: 16px;
  box-shadow: 0 20px 50px rgba(0,0,0,0.4);
  overflow: hidden;
  animation: slideUp 0.6s cubic-bezier(0.25, 0.8, 0.25, 1);
}

.login-left {
  width: 45%;
  background: linear-gradient(135deg, #1d2b64 0%, #f8cdda 100%);
  background: url('https://images.unsplash.com/photo-1449965408869-eaa3f722e40d?q=80&w=800&auto=format&fit=crop') center/cover no-repeat;
  position: relative;
  display: flex;
  align-items: center;
  justify-content: center;
  color: #fff;
}
.login-left::before {
  content: '';
  position: absolute;
  top: 0; left: 0; right: 0; bottom: 0;
  background: rgba(18, 29, 51, 0.75);
}
.left-content {
  position: relative;
  z-index: 1;
  padding: 30px;
}
.logo-huge {
  font-size: 56px;
  color: #409EFF;
  margin-bottom: 20px;
  text-shadow: 0 2px 10px rgba(64,158,255,0.5);
}
.left-content h2 {
  font-size: 26px;
  margin: 0;
  letter-spacing: 2px;
  line-height: 1.4;
}
.en-title {
  font-size: 12px;
  opacity: 0.7;
  text-transform: uppercase;
  margin-top: 10px;
  margin-bottom: 30px;
  letter-spacing: 1px;
}
.feature-list {
  list-style: none;
  padding: 0;
  margin: 0;
  font-size: 14px;
  line-height: 2.2;
  color: #e4e7ed;
}
.feature-list i {
  color: #67C23A;
  margin-right: 8px;
  font-weight: bold;
}

.login-right {
  width: 55%;
  padding: 40px 50px;
  display: flex;
  flex-direction: column;
  justify-content: center;
}
.right-header {
  margin-bottom: 30px;
}
.right-header h3 {
  font-size: 24px;
  color: #303133;
  margin: 0 0 5px 0;
}
.right-header p {
  margin: 0;
  color: #909399;
  font-size: 13px;
}

::v-deep .el-input__inner {
  border-radius: 6px;
  background-color: #f5f7fa;
  border: 1px solid #e4e7ed;
  transition: all 0.3s;
}
::v-deep .el-input__inner:focus {
  background-color: #fff;
  border-color: #409EFF;
  box-shadow: 0 0 0 2px rgba(64, 158, 255, 0.2);
}

.verify-box {
  display: flex;
  align-items: center;
}
.canvas-box {
  width: 110px;
  height: 40px;
  border-radius: 6px;
  overflow: hidden;
  cursor: pointer;
  border: 1px solid #dcdfe6;
}

.login-options {
  display: flex;
  justify-content: space-between;
  align-items: center;
  margin-bottom: 25px;
  font-size: 14px;
}
.forgot-pwd {
  color: #409EFF;
  cursor: pointer;
  transition: color 0.3s;
}
.forgot-pwd:hover {
  color: #66b1ff;
  text-decoration: underline;
}

.login-btn {
  width: 100%;
  font-size: 16px;
  letter-spacing: 4px;
  border-radius: 6px;
  background: linear-gradient(90deg, #2f4050 0%, #409EFF 100%);
  border: none;
  transition: transform 0.2s, box-shadow 0.2s;
}
.login-btn:hover {
  transform: translateY(-2px);
  box-shadow: 0 6px 15px rgba(64, 158, 255, 0.3);
}

.footer-copyright {
  position: absolute;
  bottom: 20px;
  color: rgba(255,255,255,0.4);
  font-size: 12px;
  letter-spacing: 1px;
  z-index: 1;
}

@keyframes slideUp {
  from { opacity: 0; transform: translateY(40px); }
  to { opacity: 1; transform: translateY(0); }
}
</style>