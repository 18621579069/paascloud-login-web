<template>
  <div style="display: inline-block; margin-bottom: 60px;">
    <img class="main-img" src="http://img.paascloud.net/login/paascloud/paascloud.png" alt="">
    <div class="res">
      <div class="main-top">
        <i class="fa fa-user-o" aria-hidden="true"></i>
        <span class="login">登录</span>
        <a href="#" class="login-register" @click="loadPage('Register')">注册</a>
      </div>
      <div class="main-bot">
        <input class="login-input" @keyup.13="doLogin" v-model="loginForm.loginName" type="text" placeholder="请输入帐号"/>
        <input class="login-input" @keyup.13="doLogin" v-model="loginForm.loginPwd" type="password"
               placeholder="请输入密码"/>
        <div class="image-code-div">
          <input type="text" v-model="loginForm.captchaCode" placeholder="验证码">
          <img v-lazy="imageCode" :key="imageCodeKey" @click="getImage"/>
        </div>
        <div class="main-password">
          <input id="button" type="checkbox" class="button" @click="rememberMeFn" :checked="rememberMe" value="Bike" title=""/>
          <label for="button" class="remMe">记住我</label>
          <a class="fog" @click="loadPage('ResetPwdEmail')">忘记密码?</a>
        </div>
        <button class="login" @click="doLogin">登录</button>
        <div class="other">
          <hr/><p>其他方式登录</p><hr/>
        </div>
        <div class="last">
          <a href="javascript:alert('目前只实现微信登陆')"><img src="../../../assets/images/login/csdn.png" alt="微信可用"></a>
          <a href="javascript:alert('目前只实现微信登陆')"><img src="../../../assets/images/login/github.png" alt=""></a>
          <a href="javascript:void(0)" @click="toWxLogin();return false;" >
            <img src="../../../assets/images/login/wechat.png" alt="">
          </a>
          <a href="javascript:alert('目前只实现微信登陆')"><img src="../../../assets/images/login/weibo.png" alt=""></a>
          <a href="javascript:alert('目前只实现微信登陆')"><img src="../../../assets/images/login/QQ.png" alt=""></a>
        </div>
      </div>
    </div>
  </div>
</template>
<script type="text/ecmascript-6">
  import { PcCookie } from '../../../utils/';
  export default {
    data () {
      return {
        imageCode: '',
        imageCodeKey: '',
        deviceId: '',
        loginForm: {
          loginName: '',
          loginPwd: '',
          captchaCode: ''
        }
      };
    },
    mounted () {
      this.getImage();
      this.weChatLogin();
    },
    methods: {
      weChatLogin() {
        let weixinKey = PcCookie.get('SOCIAL_WXKEY');
        if (!weixinKey) {
          return;
        }
        let code = this.getUrlParam('code');
        if (!code) {
          return;
        }
        // 如果 code 存在则调用微信登录接口
        this.$http({
          method: 'POST',
          headers: {
            'Content-Type': 'application/x-www-form-urlencoded',
            'deviceId': weixinKey
          },
          url: '/sac/socialLogin/weixin',
          auth: {
            username: 'paascloud-client-auth',
            password: 'paascloudClientSecret'
          },
          params: {
            code: this.getUrlParam('code'),
            state: this.getUrlParam('state')
          }
        }).then((res) => {
          if (res && res.code === 200) {
            this.$store.dispatch('update_auth_token', res.result)
            debugger;
            window.location.href = this.redirectUri;
          }
        }).catch((err) => {
          console.log(err);
        });
      },
      toWxLogin() {
        this.$http({
          method: 'GET',
          headers: {
            'Content-Type': 'application/x-www-form-urlencoded'
          },
          url: '/sac/social/qrconnect'
        }).then((res) => {
          if (res && res.code === 200) {
            window.location.href = res.result;
          }
        }).catch((err) => {
          console.log(err);
        });
      },
      doLogin () {
        let validateResult = this.formValidate();
        if (!validateResult.status) {
          this.$store.dispatch('new_notice', {
            autoClose: true,
            content: validateResult.msg
          });
          return;
        }
        this.login();
      },
      login () {
        let loginName = this.loginForm.loginName;
        let loginPwd = this.loginForm.loginPwd;
        this.$http({
          method: 'POST',
          headers: {
            'Content-Type': 'application/x-www-form-urlencoded',
            'deviceId': this.deviceId
          },
          url: '/sac/auth/form',
          auth: {
            username: 'paascloud-client-auth',
            password: 'paascloudClientSecret'
          },
          params: {
            username: loginName,
            password: loginPwd,
            imageCode: this.loginForm.captchaCode
          }
        }).then((res) => {
          this.getImage();
          if (res && res.code === 200) {
            console.info('登录成功, ', res);
            this.$store.dispatch('update_auth_token', res.result);
            window.location.href = this.redirectUri;
          }
        }).catch((err) => {
          console.log(err);
        });
      },
      getImage () {
        this.deviceId = new Date().getTime();
        let that = this;
        that.$http({
          method: 'POST',
          url: '/sac/auth/code/image',
          headers: {
            'deviceId': that.deviceId
          }
        }).then((res) => {
          that.imageCode = 'data:image/jpg;base64,' + res.result;
          that.imageCodeKey = new Date().getTime();
        });
      },
      rememberMeFn() {
        this.$store.dispatch('update_remember_me');
      },
      formValidate () {
        let result = {
          status: false,
          msg: ''
        };
        if (!this.validate(this.loginForm.loginName, 'require')) {
          result.msg = '用户名不能为空';
          return result;
        }
        if (!this.validate(this.loginForm.loginPwd, 'require')) {
          result.msg = '密码不能为空';
          return result;
        }
        if (!this.validate(this.loginForm.captchaCode, 'require')) {
          result.msg = '验证码不能为空';
          return result;
        }
        // 通过验证，返回正确提示
        result.status = true;
        result.msg = '验证通过';
        return result;
      }
    },
    computed: {
      rememberMe() {
        return this.$store.getters.getRememberMe;
      },
      redirectUri() {
        return this.$store.getters.getRedirectUri;
      }
    }
  };
</script>
<style rel="stylesheet/scss" lang="scss">
  .image-code-div {
    width: 320px;
    height: 40px;
    margin-bottom: 14px;
    input {
      width: 210px;
      height: 40px;
      outline: none;
      font-size: 14px;
      text-indent: 1em;
      border-radius: 2px;
      background: #F3F6F8;
      border: 1px solid #dce3e8;
    }
    img {
      width: 106px !important;
      height: 40px;
      border: 1px solid #DCE4E6;
      padding-left: 0;
      float: right;
    }
  }

  .main-img {
    width: 375px;
    margin: 27px 52.5px;
    float: left;
  }

  .main .res {
    float: right;
    width: 420px;
    margin: 0 30px;
    background-color: #ffffff;
    .main-top {
      height: 25px;
      line-height: 25px;
      margin: 10px 20px;
      i {
        font-size: 14px;
      }
      .login {
        display: inline-block;
        color: #40485b;
        font-size: 16px;
      }
      .login-register {
        float: right;
        color: #e97b21;
        font-size: 16px;
        cursor: pointer;
      }
    }
    .main-bot {
      width: 320px;
      margin: 20px 50px 30px;
      .login-input {
        width: 100%;
        height: 40px;
        margin-bottom: 20px;
        outline: 0;
        font-size: 14px;
        color: #a1a1a2;
        text-indent: 1em;
        background-color: #f3f6f8;
        border: 1px solid #dce3e8;
      }
      .button {
        width: 15px;
        height: 15px;
        vertical-align: middle;
      }
      .main-password {
        height: 20px;
        line-height: 20px;
        margin-bottom: 10px;
        .remMe {
          vertical-align: middle;
          cursor: pointer;
          color: #7e8c8d;
        }
      }
      .login {
        border: 0;
        outline: 0;
        width: 100%;
        height: 40px;
        color: #ffffff;
        cursor: pointer;
        font-size: 16px;
        margin-bottom: 20px;
        background-color: #fe7300;
      }
    }
    .other {
      width: 100%;
      height: 20px;
      line-height: 20px;
      margin-bottom: 20px;
      hr {
        width: 90px;
        display: inline-block;
        vertical-align: middle;
      }
      p {
        width: 136px;
        color: #9b9b9b;
        text-align: center;
        display: inline-block;
      }
    }
    .last {
      width: 100%;
      img {
        width: 32px;
        height: 32px;
        margin: 0 18px;
      }
      a:first-child img {
        margin-left: 0;
      }
      a:last-child img {
        margin-right: 0;
      }
    }
  }

  .fog {
    float: right;
    cursor: pointer;
  }
</style>
