<template>
  <div class="login-container" :class="{ 'dark-theme': isDarkTheme }">
    
    <div class="top-right-wrapper">
      <div class="icon-btn setting-btn" @click="showGithubDialog = true" title="GitHub 云同步配置">
        <i class="el-icon-setting"></i>
      </div>
      <div class="theme-switch" @click="toggleTheme" title="切换主题">
        <i class="fas" :class="isDarkTheme ? 'fa-sun' : 'fa-moon'"></i>
      </div>
    </div>

    <div class="card" style="margin: 20px auto;">
      <el-form :model="sshInfo" label-position="top" class="form-grid">
         <el-row :gutter="20">
           <el-col :span="12">
             <el-form-item label="地址 (Hostname)">
               <el-input ref="hostnameInput" v-model="sshInfo.hostname" placeholder="IP or Hostname" />
             </el-form-item>
           </el-col>
           <el-col :span="12">
             <el-form-item label="端口 (Port)">
               <el-input v-model.number="sshInfo.port" placeholder="22" />
             </el-form-item>
           </el-col>
         </el-row>
         <el-row :gutter="20">
           <el-col :span="12">
             <el-form-item label="用户名 (Username)">
               <el-input ref="usernameInput" v-model="sshInfo.username" placeholder="username" />
             </el-form-item>
           </el-col>
           <el-col :span="12">
             <el-form-item label="密码 (Password)">
               <el-input ref="passwordInput" v-model="sshInfo.password" type="password" placeholder="password" show-password/>
             </el-form-item>
           </el-col>
         </el-row>
         <el-row :gutter="20">
           <el-col :xs="24" :sm="12">
             <el-form-item label="私钥 (Private Key)">
               <el-upload
                 class="upload-key"
                 :show-file-list="false"
                 :before-upload="handlePrivateKeyUpload"
                 accept=".pem,.ppk,.key,.rsa,.id_rsa,.id_dsa,.txt"
               >
                 <div class="upload-flex-row">
                   <div class="upload-btn">
                     <i class="el-icon-folder-opened" style="margin-right:8px;"></i>
                     上传密钥
                   </div>
                   <div class="upload-filename" style="width: 12rem">
                     {{ privateKeyFileName || '未上传密钥文件' }}
                   </div>
                 </div>
               </el-upload>
             </el-form-item>
           </el-col>
           <el-col :xs="24" :sm="12">
             <el-form-item label="密钥口令 (PIN)">
               <el-input v-model="sshInfo.passphrase" placeholder="如果有设置请输入密钥口令" />
             </el-form-item>
           </el-col>
         </el-row>
        <el-row>
          <el-col :span="24">
            <el-form-item label="初始命令 (Initial command)">
              <el-input v-model="sshInfo.command" placeholder="Command after login" />
            </el-form-item>
          </el-col>
        </el-row>
        
        <el-row type="flex" justify="center" style="margin-top: 10px; flex-wrap: wrap; gap: 10px;">
          <el-button type="danger" icon="el-icon-refresh" @click="onReset">重置</el-button>
          
          <el-button type="warning" icon="el-icon-upload2" :loading="githubLoading" @click="syncToGitHub">云端保存</el-button>
          
          <el-button type="primary" icon="el-icon-link" @click="onGenerateLink">生成链接</el-button>
          <el-button type="success" @click="onConnect"><i class="fas fa-terminal" style="margin-right: 6px;"></i>连接SSH</el-button>
        </el-row>

        <el-row v-if="generatedLink" style="margin-top: 18px;">
          <el-col :span="24">
            <el-input v-model="generatedLink" readonly class="gen-link-input">
              <template slot="append">
                <el-button style="color: #1adb6d;" @click="copyLink" icon="el-icon-document-copy"></el-button>
              </template>
            </el-input>
          </el-col>
        </el-row>
      </el-form>
    </div>

    <div class="footer">
      <a href="https://github.com/adamj001/webssh-lw" target="_blank" rel="noopener noreferrer">wssh</a>
    </div>

    <el-dialog title="GitHub 云同步配置" :visible.sync="showGithubDialog" width="90%" :custom-class="isDarkTheme ? 'dark-dialog' : ''" append-to-body>
      <el-form :model="githubConfig" label-width="100px" size="small">
        <el-form-item label="Token">
          <el-input v-model="githubConfig.token" type="password" show-password placeholder="ghp_xxxxxxxx..."></el-input>
          <div style="font-size:12px; opacity: 0.7;">需勾选 repo 权限</div>
        </el-form-item>
        <el-form-item label="用户名">
          <el-input v-model="githubConfig.owner" placeholder="GitHub Username"></el-input>
        </el-form-item>
        <el-form-item label="仓库名">
          <el-input v-model="githubConfig.repo" placeholder="Repository Name (私库)"></el-input>
        </el-form-item>
        <el-form-item label="文件路径">
          <el-input v-model="githubConfig.path" placeholder="例如: data/ssh_list.json"></el-input>
        </el-form-item>
      </el-form>
      <span slot="footer" class="dialog-footer">
        <el-button @click="showGithubDialog = false">取 消</el-button>
        <el-button type="primary" @click="saveGithubSettings">保存配置</el-button>
      </span>
    </el-dialog>

  </div>
</template>

<script>
import axios from 'axios' // 引入 axios

export default {
  data () {
    return {
      sshInfo: {
        hostname: '',
        port: '',
        username: '',
        password: '',
        privateKey: '',
        passphrase: '',
        command: ''
      },
      privateKeyFileName: '',
      generatedLink: '',
      isDarkTheme: false,
      
      // GitHub 相关数据
      showGithubDialog: false,
      githubLoading: false,
      githubConfig: {
        token: '',
        owner: '',
        repo: '',
        path: 'ssh_hosts.json' // 默认文件名
      }
    }
  },
  watch: {
    sshInfo: {
      handler(newVal) {
        localStorage.setItem('sshInfo', JSON.stringify(newVal));
      },
      deep: true
    }
  },
  created() {
    // 恢复 GitHub 配置
    const ghConfig = localStorage.getItem('gh_config');
    if (ghConfig) {
      this.githubConfig = JSON.parse(ghConfig);
    }

    // 从 localStorage 恢复完整的连接信息
    const savedInfo = localStorage.getItem('connectionInfo')
    if (savedInfo) {
      const info = JSON.parse(savedInfo)
      this.sshInfo = {
        hostname: info.hostname || '',
        port: info.port || '',
        username: info.username || '',
        password: info.password || '',
        privateKey: info.privateKey || '',
        passphrase: info.passphrase || '',
        command: info.command || ''
      }
      if (info.privateKey) {
        this.privateKeyFileName = '已保存的密钥文件'
      }
    }
    
    // 检查主题设置
    const savedTheme = localStorage.getItem('isDarkTheme')
    if (savedTheme !== null) {
      this.isDarkTheme = savedTheme === 'true'
    }
    
    // 添加 Font Awesome CSS
    const link = document.createElement('link')
    link.rel = 'stylesheet'
    link.href = 'https://cdnjs.cloudflare.com/ajax/libs/font-awesome/5.15.4/css/all.min.css'
    document.head.appendChild(link)
  },
  methods: {
    // 保存 GitHub 设置
    saveGithubSettings() {
      localStorage.setItem('gh_config', JSON.stringify(this.githubConfig));
      this.showGithubDialog = false;
      this.$message.success('GitHub 配置已保存');
    },

    // 核心：同步到 GitHub
    async syncToGitHub() {
      const { token, owner, repo, path } = this.githubConfig;
      if (!token || !owner || !repo) {
        this.$message.warning('请先点击右上角设置图标配置 GitHub 信息');
        this.showGithubDialog = true;
        return;
      }
      if (!this.sshInfo.hostname || !this.sshInfo.username) {
         this.$message.error('主机名和用户名不能为空');
         return;
      }

      this.githubLoading = true;
      const apiUrl = `https://api.github.com/repos/${owner}/${repo}/contents/${path}`;
      const headers = {
        'Authorization': `token ${token}`,
        'Accept': 'application/vnd.github.v3+json'
      };

      // 构造唯一索引 Key
      const uniqueKey = `${this.sshInfo.hostname}_${this.sshInfo.username}`;
      // 准备要保存的数据 (去掉了私钥，因为私钥太大且敏感，建议只存基本信息，或者你自己决定是否存)
      const dataToSave = {
        hostname: this.sshInfo.hostname,
        port: this.sshInfo.port,
        username: this.sshInfo.username,
        command: this.sshInfo.command,
        updated_at: new Date().toLocaleString(),
        remark: 'WebSSH Sync'
      };

      try {
        // 1. 获取现有文件 (为了拿到 sha 和现有内容)
        let existingContent = {};
        let sha = null;

        try {
          const res = await axios.get(apiUrl, { headers });
          sha = res.data.sha;
          // 解码 Base64 内容 (处理中文)
          const decodedContent = decodeURIComponent(escape(window.atob(res.data.content)));
          existingContent = JSON.parse(decodedContent);
        } catch (error) {
          if (error.response && error.response.status === 404) {
             // 文件不存在，跳过读取，直接新建
          } else {
            throw error;
          }
        }

        // 2. 更新或新增数据
        existingContent[uniqueKey] = dataToSave;

        // 3. 推送更新
        const contentBase64 = window.btoa(unescape(encodeURIComponent(JSON.stringify(existingContent, null, 2))));
        
        const payload = {
          message: `Update SSH: ${uniqueKey}`,
          content: contentBase64
        };
        if (sha) payload.sha = sha;

        await axios.put(apiUrl, payload, { headers });
        this.$message.success(`已保存到 GitHub: ${uniqueKey}`);

      } catch (e) {
        console.error(e);
        let msg = '同步失败';
        if(e.response && e.response.status === 401) msg = 'GitHub Token 无效';
        if(e.response && e.response.status === 404) msg = '仓库未找到';
        this.$message.error(msg);
      } finally {
        this.githubLoading = false;
      }
    },

    onConnect () {
      // 清除之前的认证信息
      sessionStorage.removeItem('sshInfo')
      
      if (!this.sshInfo.hostname) {
        this.$message.error('请输入主机地址！')
        this.$nextTick(() => {
          this.$refs.hostnameInput && this.$refs.hostnameInput.focus()
        })
        return
      }
      if (!this.sshInfo.username) {
        this.$message.error('请输入用户名！')
        this.$nextTick(() => {
          this.$refs.usernameInput && this.$refs.usernameInput.focus()
        })
        return
      }
      if (!this.sshInfo.password && !this.sshInfo.privateKey) {
        this.$message.error('请输入密码或上传密钥！')
        this.$nextTick(() => {
          this.$refs.passwordInput && this.$refs.passwordInput.focus()
        })
        return
      }

      // 根据实际使用的登录方式清理未使用的认证信息
      if (this.sshInfo.privateKey && this.sshInfo.privateKey.trim()) {
        this.sshInfo.password = ''
      } else if (this.sshInfo.password) {
        // 使用密码登录时，清除密钥相关信息
        this.sshInfo.privateKey = ''
        this.sshInfo.passphrase = ''
        this.privateKeyFileName = ''
      }

      const connectionInfo = {
        hostname: this.sshInfo.hostname,
        port: this.sshInfo.port || 22,
        username: this.sshInfo.username,
        password: this.sshInfo.password || '',
        privateKey: this.sshInfo.privateKey || '',
        passphrase: this.sshInfo.passphrase || '',
        command: this.sshInfo.command || ''
      }
      localStorage.setItem('connectionInfo', JSON.stringify(connectionInfo))

      const query = {
        hostname: encodeURIComponent(this.sshInfo.hostname),
        port: Number(this.sshInfo.port) || 22,
        username: encodeURIComponent(this.sshInfo.username),
        command: encodeURIComponent(this.sshInfo.command || '')
      }

      if (this.sshInfo.privateKey && this.sshInfo.privateKey.trim()) {
        sessionStorage.setItem('sshInfo', JSON.stringify(this.sshInfo))
        query.useKey = 1
      } else if (this.sshInfo.password) {
        query.password = btoa(this.sshInfo.password)
      }

      const url = this.$router.resolve({ path: '/terminal', query }).href
      window.open(url, '_blank')
    },
    onReset () {
      this.sshInfo = { 
        hostname: '', 
        port: '', 
        username: '', 
        password: '', 
        command: '', 
        privateKey: '', 
        passphrase: '' 
      }
      this.privateKeyFileName = ''
      this.generatedLink = ''
      localStorage.removeItem('connectionInfo')
      sessionStorage.removeItem('sshInfo')
      const fileInput = document.querySelector('.upload-key input[type="file"]')
      if (fileInput) {
        fileInput.value = ''
      }
    },
    onGenerateLink () {
      if (this.sshInfo.privateKey) {
        this.$message.warning('密钥方式登录不支持生成快捷链接，请改用密码登录方式')
        return
      }
      if (!this.sshInfo.hostname) {
        this.$message.error('请输入主机地址！')
        this.$nextTick(() => {
          this.$refs.hostnameInput && this.$refs.hostnameInput.focus()
        })
        return
      }
      if (!this.sshInfo.username) {
        this.$message.error('请输入用户名！')
        this.$nextTick(() => {
          this.$refs.usernameInput && this.$refs.usernameInput.focus()
        })
        return
      }
      if (!this.sshInfo.password && !this.sshInfo.privateKey) {
        this.$message.error('请输入密码或上传密钥以生成链接！')
        this.$nextTick(() => {
          this.$refs.passwordInput && this.$refs.passwordInput.focus()
        })
        return
      }
      const url = new URL(window.location.href)
      url.pathname = '/terminal'
      const cleanSshInfo = {}
      const infoToProcess = { ...this.sshInfo, port: this.sshInfo.port || 22 }
      for (const key in infoToProcess) {
        if (infoToProcess[key] !== '' && infoToProcess[key] !== null) {
          if (key === 'password') {
            cleanSshInfo[key] = btoa(infoToProcess[key])
          } else {
            cleanSshInfo[key] = infoToProcess[key]
          }
        }
      }
      url.search = new URLSearchParams(cleanSshInfo).toString()
      this.generatedLink = url.href
    },
    copyLink () {
      if (this.generatedLink) {
        navigator.clipboard.writeText(this.generatedLink).then(() => {
          this.$message.success('链接已复制！')
        }).catch(err => {
          this.$message.error('复制失败: ' + err)
        })
      }
    },
    toggleTheme () {
      this.isDarkTheme = !this.isDarkTheme;
      localStorage.setItem('isDarkTheme', this.isDarkTheme);
    },
    handlePrivateKeyUpload(file) {
      this.sshInfo.password = ''
      const reader = new FileReader()
      reader.onload = (e) => {
        this.sshInfo.privateKey = e.target.result
        this.privateKeyFileName = file.name
      }
      reader.readAsText(file)
      return false
    }
  }
}
</script>

<style lang="scss" scoped>
/* 为了适应新的右上角布局，修改了这里 */
.top-right-wrapper {
  position: absolute;
  top: 25px;
  right: 30px;
  z-index: 10;
  display: flex;
  gap: 15px; /* 图标之间的间距 */
}

.icon-btn, .theme-switch {
  width: 40px;
  height: 40px;
  border-radius: 50%;
  display: flex;
  align-items: center;
  justify-content: center;
  cursor: pointer;
  transition: background-color 0.3s;
  background-color: var(--switch-bg); /* 给按钮加个背景色，防止在深色图上看不清 */
  box-shadow: 0 2px 5px rgba(0,0,0,0.1);
}

.icon-btn:hover, .theme-switch:hover {
  filter: brightness(0.9);
}

.icon-btn i, .theme-switch i {
  font-size: 20px;
  color: var(--icon-color);
  transition: color 0.3s;
}

/* 修复原有的 icon 位置问题 */
.theme-switch i {
  margin-top: 0; /* 去掉了原来的 -30px，使用 flex 居中更稳 */
}


.login-container ::v-deep .el-input__inner {
  font-size: medium;
  border-radius: 10px;
  background: hsl(0deg 0% 100% / 5%);
  backdrop-filter: blur(5px);
  -webkit-backdrop-filter: blur(5px);
  border: 1px solid rgba(255, 255, 255, 0.3);
  transition: all 0.3s;
  color: #333;
}

.login-container ::v-deep .el-input__inner:focus {
  border-color: #409eff !important;
  box-shadow: 0 0 0 2px rgba(64, 158, 255, 0.2) !important;
  caret: 2px solid #409eff !important;
}

.login-container ::v-deep .el-input.is-focus .el-input__inner {
  border-color: #409eff !important;
  box-shadow: 0 0 0 2px rgba(64, 158, 255, 0.2) !important;
  caret: 2px solid #409eff !important;
}

.login-container ::v-deep .el-input__inner::placeholder {
  color: #565454 !important;
  opacity: 1;
}

.login-container ::v-deep .el-input__suffix {
  background: transparent;
  margin-right: 5px;
}

.login-container ::v-deep .el-input__suffix .el-input__suffix-inner {
  display: flex;
  align-items: center;
  justify-content: center;
}

.login-container ::v-deep .el-input__suffix .el-input__suffix-inner .el-input__icon {
  color: #666;
  font-size: 16px;
  transition: color 0.3s;
}

.login-container ::v-deep .el-input__suffix .el-input__suffix-inner .el-input__icon:hover {
  color: #409eff;
}

.login-container.dark-theme ::v-deep .el-input__inner {
  background: hsl(0deg 0% 100% / 5%);
  border: 1px solid rgba(255, 255, 255, 0.2);
  color: #fff;
}

.login-container.dark-theme ::v-deep .el-input__inner:focus {
  border-color: #409eff !important;
  box-shadow: 0 0 0 2px rgba(64, 158, 255, 0.2) !important;
  caret: 2px solid #409eff !important;
}

.login-container.dark-theme ::v-deep .el-input.is-focus .el-input__inner {
  border-color: #409eff !important;
  box-shadow: 0 0 0 2px rgba(64, 158, 255, 0.2) !important;
  caret: 2px solid #409eff !important;
}

.login-container.dark-theme ::v-deep .el-input__inner::placeholder {
  color: #ccc !important;
  opacity: 1;
}

.login-container.dark-theme ::v-deep .el-input__suffix {
  background: transparent;
  margin-right: 5px;
}

.login-container.dark-theme ::v-deep .el-input__suffix .el-input__suffix-inner .el-input__icon {
  color: #ccc;
  font-size: 16px;
  transition: color 0.3s;
}

.login-container.dark-theme ::v-deep .el-input__suffix .el-input__suffix-inner .el-input__icon:hover {
  color: #409eff;
}

.login-container ::v-deep .el-form-item {
  margin-bottom: 15px;
}

.login-container {
  display: flex;
  min-height: 100vh;
  flex-direction: column;
  align-items: center;
  background: var(--bg-color);
  background-image: var(--bg-image);
  background-size: cover;
  background-position: center;
  background-repeat: no-repeat;
  background-attachment: fixed;
  position: relative;
  padding-top: 5vh;
  padding-bottom: 60px;
  transition: background-color 0.3s, color 0.3s, background-image 0.3s;
  overflow-y: auto;
}

.card {
  background: var(--card-bg);
  backdrop-filter: blur(10px);
  -webkit-backdrop-filter: blur(10px);
  box-shadow: var(--shadow);
  border-radius: 20px;
  padding-top: 15px;
  padding-bottom: 25px;
  width: 100%;
  max-width: 42rem;
  position: relative;
  transition: background-color 0.3s, box-shadow 0.3s, backdrop-filter 0.3s;
  border: 1px solid rgba(255, 255, 255, 0.1);
}

.form-grid ::v-deep .el-form-item__label {
  padding-bottom: 0;
  font-size: 15px;
  color: var(--text-color);
  line-height: 30px;
  transition: color 0.3s;
}

.form-grid ::v-deep .el-button {
  font-size: 1rem;
  font-weight: 600;
  padding: 0.9rem 1rem;
  border-radius: 10px;
  transition: all 0.3s;
}

.login-container ::v-deep .el-form-item .el-upload.upload-key {
  width: 100% !important;
  height: 48px !important;
  background: none !important;
  border: none !important;
  box-shadow: none !important;
  margin: 0 !important;
  padding: 0 !important;
}

.login-container ::v-deep .upload-flex-row {
  display: flex !important;
  flex-direction: row !important;
  align-items: stretch !important;
  width: 100%;
  height: 100%;
}

.login-container ::v-deep .upload-flex-row .upload-btn,
.login-container ::v-deep .upload-flex-row .upload-filename {
  height: 100%;
  display: flex;
  align-items: center;
}

.login-container ::v-deep .upload-btn {
  display: flex;
  align-items: center;
  background: var(--primary);
  color: #fff;
  font-weight: 600;
  font-size: 16px;
  border-radius: 10px 0 0 10px;
  padding: 0 28px;
  cursor: pointer;
  transition: background 0.2s;
  height: 100%;
  backdrop-filter: blur(5px);
  -webkit-backdrop-filter: blur(5px);
}

.login-container ::v-deep .upload-btn:hover {
  background: #202f3e;
}

.login-container ::v-deep .upload-filename {
  display: flex;
  align-items: center;
  background: hsl(0deg 0% 100% / 15%);
  backdrop-filter: blur(5px);
  -webkit-backdrop-filter: blur(5px);
  color: #333;
  font-size: 15px;
  border-radius: 0 12px 12px 0;
  padding: 0 10px;
  height: 100%;
  flex: 1;
  white-space: nowrap;
  overflow: hidden;
  text-overflow: ellipsis;
  border: 1px solid rgba(255, 255, 255, 0.3);
}

@media (max-width: 768px) {
  .card{
    width: 98% !important;
  }
  
  .form-grid ::v-deep .el-button {
    font-size: 0.9rem !important;
    padding: 0.7rem 0.8rem !important;
    margin: 0 2px !important;
  }
  
  .el-row[type="flex"] {
    margin-top: 8px !important;
  }
  
  .login-container {
    padding-bottom: 120px !important;
    min-height: auto !important;
  }
  
  .footer {
    position: relative !important;
    bottom: auto !important;
    margin-top: 20px !important;
  }
  
  .card {
    margin: 10px auto !important;
  }
  
  /* 移动端调整一下顶部图标位置 */
  .top-right-wrapper {
    top: 15px;
    right: 15px;
  }
}

.login-container.dark-theme ::v-deep .upload-key {
  border-color: #4d4d4d;
  background-color: var(--card-bg);
}
.login-container.dark-theme ::v-deep .upload-key .el-button {
  background: #232323;
  color: #409eff;
}
.login-container.dark-theme ::v-deep .upload-key .el-button:hover,
.login-container.dark-theme ::v-deep .upload-key .el-button:focus {
  background: #409eff !important;
  color: #fff !important;
}
.login-container.dark-theme ::v-deep .upload-key span {
  background: #23232339;
  color: #aaa !important;
}

.login-container.dark-theme ::v-deep .upload-filename {
  background: rgba(45, 45, 45, 0.2) !important;
  backdrop-filter: blur(5px) !important;
  -webkit-backdrop-filter: blur(5px) !important;
  color: #fff !important;
  border: 1px solid rgba(255, 255, 255, 0.2) !important;
}

.footer {
  position: absolute;
  bottom: 8px;
  text-align: center;
  width: 100%;
  color: var(--text-color);
  opacity: 0.6;
  transition: color 0.3s;
}

.footer a {
  font-size: 0.9rem;
  color: #000000;
  font-family: system-ui;
  color: #fefefe;
  text-decoration: none;
  transition: color 0.3s;
}

.footer a:hover {
  color: #05d899;
}

/* Light theme variables */
.login-container {
  --bg-color: #ffff;
  --bg-image: url('/static/img/bg_light.webp');
  --card-bg: hsl(0deg 0% 100% / 15%);
  --title-color: #1b58c9;
  --text-color: #3b3d3d;
  --shadow: 0 4px 6px rgba(0, 0, 0, 0.1);
  --success: #13af54;
  --success-hover: #0e8942; 
  --danger: #d63031;
  --danger-hover: #b247c2; 
  --primary: #409eff;
  --primary-hover: #0f9281; 
  --switch-bg: #f0f0f0;
  --icon-color: #232323;
}

/* Dark theme variables */
.login-container.dark-theme {
  --bg-color: #ffffff;
  --bg-image: url('/static/img/bg_dark.webp');
  --card-bg: hsl(0deg 0% 100% / 5%);
  --title-color: #ffffff;
  --text-color: #e0e0e0;
  --shadow: 0 4px 6px rgba(0, 0, 0, 0.3);
  --success: #13af54;
  --success-hover: #0e8942;
  --danger: #d63031;
  --danger-hover: #b247c2;
  --primary: #409eff;
  --primary-hover: #0f9281;
  --switch-bg: #333333;
  --icon-color: #f5f5f5;
}

.el-button--success {
  background-color: var(--success);
  border-color: var(--success);
  color: white;
}
.el-button--danger {
  background-color: var(--danger);
  border-color: var(--danger);
  color: white;
}
.el-button--primary {
  background-color: var(--primary);
  border-color: var(--primary);
  color: white;
}
.el-button--success:hover {
  background-color: var(--success-hover);
  border-color: var(--success-hover);
  color: white;
}
.el-button--danger:hover {
  background-color: var(--danger-hover);
  border-color: var(--danger-hover);
  color: white;
}
.el-button--primary:hover {
  background-color: var(--primary-hover);
  border-color: var(--primary-hover);
  color: white;
}

.login-container ::v-deep .gen-link-input .el-input__inner {
  border-radius: 10px 0 0 10px !important;
  padding-right: 5px;
}
.login-container ::v-deep .gen-link-input .el-input-group__append {
  border-radius: 0 10px 10px 0 !important;
}

.login-container ::v-deep .el-input-group__append {
  background-color: rgba(255, 255, 255, 0.2) !important;
  backdrop-filter: blur(5px) !important;
  -webkit-backdrop-filter: blur(5px) !important;
  border: 1px solid rgba(255, 255, 255, 0.3) !important;
  transition: background-color 0.3s;
}
.login-container.dark-theme ::v-deep .el-input-group__append {
  background-color: rgba(45, 45, 45, 0.2) !important;
  backdrop-filter: blur(5px) !important;
  -webkit-backdrop-filter: blur(5px) !important;
  border: 1px solid rgba(255, 255, 255, 0.2) !important;
}

/* 深色主题下 Dialog 的适配 */
::v-deep .dark-dialog {
  background: #2d2d2d;
}
::v-deep .dark-dialog .el-dialog__title {
  color: #fff;
}
::v-deep .dark-dialog .el-form-item__label {
  color: #ccc;
}
::v-deep .dark-dialog .el-input__inner {
  background-color: #333;
  border-color: #555;
  color: #fff;
}
</style>
