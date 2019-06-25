English | [简体中文](./README.zh.md)

## 功能

```
- 登录 / 注销

- 权限验证
  - 页面权限
  - 指令权限
  - 权限配置
  - 二步登录


- 错误页面
  - 401
  - 404

```

## 开发

```bash
# 克隆项目
git clone https://github.com/PanJiaChen/vue-element-admin.git

# 进入项目目录
cd vue-element-admin

# 安装依赖
npm install

# 建议不要直接使用 cnpm 安装以来，会有各种诡异的 bug。可以通过如下操作解决 npm 下载速度慢的问题
npm install --registry=https://registry.npm.taobao.org

# 启动服务
npm run dev
```

浏览器访问 http://localhost:9527

## 发布

```bash
# 构建测试环境
npm run build:stage

# 构建生产环境
npm run build:prod
```

## 其它

```bash
# 预览发布环境效果
npm run preview

# 预览发布环境效果 + 静态资源分析
npm run preview -- --report

# 代码格式检查
npm run lint

# 代码格式检查并自动修复
npm run lint -- --fix
```

# 二次开发：

## 1.接入真实接口（以登录为例子）

  ```bash
  # 配置代理
    <1>. 在vue.config.js里，配置proxy,将Mock的proxy注释掉,并写入:
        proxy: {
          // change xxx-api/login => mock/login
          // detail: https://cli.vuejs.org/config/#devserver-proxy

          // [process.env.VUE_APP_BASE_API]: {
          //   target: `http://127.0.0.1:${port}/mock`,
          //   changeOrigin: true,
          //   pathRewrite: {
          //     ['^' + process.env.VUE_APP_BASE_API]: ''
          //   }
          // }

          [process.env.VUE_APP_BASE_API]: {
            target: `http://localhost:5000/api/workflow`,
            changeOrigin: true,
            pathRewrite: {
              ['^' + process.env.VUE_APP_BASE_API]: ''
            }
          }
        },

    <2>.在.env.development文件中写入:
          VUE_APP_BASE_API = '/api/workflow'

    <3>.在src/api/user.js里，写入接口地址:
        export function login(data) {
          return request({
            url: '/common/login',
            method: 'post',
            data
          })
        }

    <4>.在src/views/login/index.vue里，将username改为userNo, password改为passWord

    <5>.在src/store/modules/user.js里，将action里的username改为userNo, password改为passWord

    <6>.在src/utils/validate.js里的validUsername方法，添加一个用户名
        export function validUsername(str) {
          const valid_map = ['admin', 'editor', '0684007']
          return valid_map.indexOf(str.trim()) >= 0
        }

    <7>.再点击登录的时候接口就成功返回了

  # 存储token
    <1>.在src/utils/request.js里，修改返回拦截的代码:
        if (res.fault.faultCode !== 'success') {
          Message({
            message: res.fault.faultString || 'Error',
            type: 'error',
            duration: 5 * 1000
          })

          // 50008: Illegal token; 50012: Other clients logged in; 50014: Token expired;
          if (res.code === 50008 || res.code === 50012 || res.code === 50014) {
            // to re-login
            MessageBox.confirm('You have been logged out, you can cancel to stay on this page, or log in again', 'Confirm logout', {
              confirmButtonText: 'Re-Login',
              cancelButtonText: 'Cancel',
              type: 'warning'
            }).then(() => {
              store.dispatch('user/resetToken').then(() => {
                location.reload()
              })
            })
          }
          return Promise.reject(new Error(res.fault.faultString || 'Error'))
        } else {
          return res
        }
  ```

## 2.新建页面(以login-info页面为例)
  <1>. 在src/views下，新建login-info文件夹，并写入内容

  <2>. 在router/index.js里的constantRoutes里，引入login-info
      {
        path: '/login-info',
        component: () => import('@/views/login-info/index'), // 按需加载
        hidden: true // 菜单隐藏
      },

## 3.修改element-ui的默认样式 (以form表单为例)
    <1>. 需求: input的高度为40px, label的大小为16px
        <el-form-item label="请选择角色:" prop="rule" class="formItem">
          <el-select v-model="ruleForm.rule" clearable placeholder="请选择">
            <el-option
              v-for="item in ruleForm.options"
              :key="item.value"
              :label="item.label"
              :value="item.value"
            />
          </el-select>
        </el-form-item>

    <2>.如果页面中样式带有__的(如: el-form-item__label),只能在 src/styles/index.scss进行全局修改
    没有带__的(如： el-form-item)，可以直接通过这个类名在页面中修改

## 4.权限集成
  <1>.在src/views/login/login.vue里，将跳转路径修改了:
      handleLogin() {
        this.$refs.loginForm.validate(valid => {
          if (valid) {
            this.loading = true
            this.$store.dispatch('user/loginInfo', this.loginForm)
              .then(() => {
                this.$router.push({ path: '/login-info' }) // 跳转到选角色页面
                this.loading = false
              })
              .catch(() => {
                this.loading = false
              })
          } else {
            console.log('error submit!!')
            return false
          }
        })
      },

  <2>.在src/views/login-info/index.vue里，将user这个选的值存入localstorage，并跳转到首页

  <3>.接口配置,报文头设置在src/utils/request.js里面配置

  <3>.修改src/store/user.js，将getInfo方法重写:          
        // 目的就是返回roles（角色），并把roles存到vuex
        getInfo({ commit, state }) {
          const userInfo = JSON.parse(localStorage.getItem('userInfo'))
          const obj = {}
          const roles = []
          roles.push(userInfo.ruleObj.ROLE_NO)
          const name = userInfo.userNm
          const avatar = 'https://wpimg.wallstcn.com/f778738c-e4f8-4870-b634-56703b4acafe.gif'
          const introduction = 'I am a super administrator'
          obj.roles = roles
          obj.name = name
          obj.avatar = avatar
          obj.introduction = introduction
          commit('SET_ROLES', roles)
          commit('SET_NAME', name)
          commit('SET_AVATAR', avatar)
          commit('SET_INTRODUCTION', introduction)
          return obj
        }

  <4>.可在router/index.js，在asyncRoutes里，添加角色（ROL4350701）进行测试  

## 5.mapState的用法
  <1>.mapState = this.$store.state, 但是，在页面中使用了mapState放出值之后，data里不能定义相同的值,
    并且，由于使用modules分割store, mapstate只能取到store里的模块名(如chooseSys,user)         
      <template>
        <el-col :span='6' v-for="item in chooseSys.list" :key="item.SystemNo" class="box">
          <el-card>
            <h2>{{item.SystemNm}}</h2>
            <h5>{{item.SysUsNm}}</h5>
          </el-card>
        </el-col>
      </template>

      <script>
        import { mapState } from 'vuex';
        export default {
          data() {
            return {
              
            }
          },
          computed: {
            ...mapState(['chooseSys'])
          },
      </script>

## 6. v-if的用法
  <1>.需求: 所属系统页面，根据icon的数值的不同，展示不同的图片

  <2>.代码实现:
      <el-col :span='6' v-for="item in chooseSys.list" :key="item.SystemNo" class="box">
        <el-card class="box_list" @click.native.prevent="turnHome">
          <img :src="imgUrl1" v-if="item.Icon === '12'" />
          <img :src="imgUrl2" v-else />
          <h2>{{item.SystemNm}}</h2>
          <h5>{{item.SysUsNm}}</h5>
        </el-card>
      </el-col>

      data() {
        return {
          imgUrl1: img1,
          imgUrl2: img2,
        }
      },

    

    

