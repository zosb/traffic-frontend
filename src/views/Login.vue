<template>
  <div class="login-container">
    <div class="login-box">
      <div class="login-header">
        <i class="el-icon-data-analysis logo-icon"></i>
        <h2 class="system-title">城市交通流量分析平台</h2>
        <p class="sub-title">Urban Traffic Analysis Platform</p>
      </div>

      <el-form :model="loginForm" :rules="rules" ref="loginForm" class="login-form">

        <el-form-item prop="username">
          <el-input
              v-model="loginForm.username"
              prefix-icon="el-icon-user"
              placeholder="请输入账号"
              @keyup.enter.native="handleLogin">
          </el-input>
        </el-form-item>

        <el-form-item prop="password">
          <el-input
              type="password"
              v-model="loginForm.password"
              prefix-icon="el-icon-lock"
              placeholder="请输入密码"
              show-password
              @keyup.enter.native="handleLogin">
          </el-input>
        </el-form-item>

        <el-form-item prop="role">
          <el-select v-model="loginForm.role" placeholder="请选择登录身份" style="width: 100%">
            <el-option label=" 交通管理人员 (Manager)" value="manager"></el-option>
            <el-option label=" 交通规划设计师 (Planner)" value="planner"></el-option>
            <el-option label=" 交通数据分析师 (Analyst)" value="analyst"></el-option>
          </el-select>
        </el-form-item>

        <div class="login-options">
          <el-checkbox v-model="rememberMe">记住密码</el-checkbox>
          <span class="forgot-pwd">忘记密码?</span>
        </div>

        <el-button
            type="primary"
            class="login-btn"
            :loading="loading"
            @click="handleLogin">
          {{ loading ? '登 录 中...' : '立 即 登 录' }}
        </el-button>

      </el-form>

      <div class="tips">
        <i class="el-icon-info"></i> 测试账号: admin / 123456
      </div>
    </div>

    <div class="footer-copyright">© 2025 Smart City Traffic System</div>
  </div>
</template>

<script>
export default {
  name: 'Login',
  data() {
    return {
      loading: false,
      rememberMe: false,
      loginForm: {
        username: '',
        password: '',
        role: 'manager'
      },
      rules: {
        username: [{ required: true, message: '请输入用户名', trigger: 'blur' }],
        password: [{ required: true, message: '请输入密码', trigger: 'blur' }],
        role: [{ required: true, message: '请选择登录身份', trigger: 'change' }]
      }
    };
  },
  methods: {
    handleLogin() {
      this.$refs.loginForm.validate((valid) => {
        if (valid) {
          this.loading = true;
          setTimeout(() => {
            if (this.loginForm.username === 'admin' && this.loginForm.password === '123456') {

              localStorage.setItem('userRole', this.loginForm.role);
              localStorage.setItem('username', this.loginForm.username);

              const roleNames = {
                'manager': '交通管理人员',
                'planner': '交通规划设计师',
                'analyst': '交通数据分析师'
              };
              this.$message.success(`登录成功！欢迎回来，${roleNames[this.loginForm.role]}`);

              this.$router.push('/dashboard');
            } else {
              this.$message.error('账号或密码错误 (admin / 123456)');
              this.loading = false;
            }
          }, 800);
        } else {
          return false;
        }
      });
    }
  }
};
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
  overflow: hidden;
}

.login-box {
  width: 420px;
  background: rgba(255, 255, 255, 0.96);
  border-radius: 12px;
  padding: 40px 45px;
  box-shadow: 0 15px 35px rgba(0,0,0,0.25);
  backdrop-filter: blur(10px);
  animation: slideUp 0.6s cubic-bezier(0.25, 0.8, 0.25, 1);
}

.login-header {
  text-align: center;
  margin-bottom: 30px;
}
.logo-icon {
  font-size: 48px;
  color: #409EFF;
  margin-bottom: 15px;
  display: inline-block;
}
.system-title {
  margin: 0;
  font-size: 26px;
  color: #303133;
  letter-spacing: 1px;
  font-weight: 600;
}
.sub-title {
  margin-top: 8px;
  color: #909399;
  font-size: 13px;
  text-transform: uppercase;
  letter-spacing: 2px;
}

.login-form .el-input__inner {
  height: 45px;
  line-height: 45px;
}
.login-form .el-form-item {
  margin-bottom: 22px;
}

.login-options {
  display: flex;
  justify-content: space-between;
  align-items: center;
  margin-bottom: 25px;
  font-size: 14px;
  color: #606266;
}
.forgot-pwd {
  color: #409EFF;
  cursor: pointer;
}
.forgot-pwd:hover {
  text-decoration: underline;
}

.login-btn {
  width: 100%;
  padding: 12px 0;
  font-size: 16px;
  letter-spacing: 2px;
  background: linear-gradient(to right, #4ca1af, #2c3e50);
  border: none;
  transition: all 0.3s;
}
.login-btn:hover {
  opacity: 0.9;
  transform: translateY(-1px);
  box-shadow: 0 4px 12px rgba(44, 62, 80, 0.3);
}

.tips {
  margin-top: 20px;
  text-align: center;
  font-size: 12px;
  color: #909399;
  background: #f4f4f5;
  padding: 8px;
  border-radius: 4px;
}

.footer-copyright {
  position: absolute;
  bottom: 25px;
  color: rgba(255,255,255,0.7);
  font-size: 12px;
  letter-spacing: 0.5px;
}

@keyframes slideUp {
  from {
    opacity: 0;
    transform: translateY(30px);
  }
  to {
    opacity: 1;
    transform: translateY(0);
  }
}
</style>