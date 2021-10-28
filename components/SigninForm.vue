<template>
  <div>
    <el-carousel
      ref="sign"
      arrow="never"
      indicator-position="none"
      :height="height"
      :autoplay="false"
    >
      <el-carousel-item name="password">
        <el-form ref="signForm" :model="signForm" :rules="signRules">
          <el-form-item :error="passwordError" prop="name">
            <el-input
              v-model="signForm.name"
              placeholder="请输入用户名"
              clearable
            ></el-input>
          </el-form-item>
          <el-form-item :error="passwordError" prop="password">
            <el-input
              v-model="signForm.password"
              placeholder="请输入密码"
              clearable
              show-password
            ></el-input>
          </el-form-item>
          <el-form-item>
            <el-button type="primary" @click="onSubmit">登录账号</el-button>
          </el-form-item>
        </el-form>
      </el-carousel-item>
      <el-carousel-item name="captcha">
        <el-form ref="captchaForm" :model="captchaForm" :rules="captchaRules">
          <el-form-item :error="captchaError">
            <el-image
              style="width: 240px; height: 80px"
              :src="captchaQuestion"
              @click="refreshCaptcha"
            ></el-image>
            <el-input
              v-model="captchaForm.answer"
              placeholder="请输入验证码"
              clearable
            ></el-input>
          </el-form-item>
          <el-form-item>
            <el-button type="primary" @click="onSubmitVerify"
              >确认提交</el-button
            >
          </el-form-item>
        </el-form>
      </el-carousel-item>
      <el-carousel-item name="success">
        <el-result icon="success" title="登录成功" />
      </el-carousel-item>
    </el-carousel>
  </div>
</template>

<script lang="ts">
import { ElCarousel } from "element-ui/types/carousel";
import { ElForm } from "element-ui/types/form";
import Vue from "vue";

const ErrCodeUserPasswordEmpty = 50000;
const ErrCodeUserPasswordError = 50001;
const ErrCodeCaptcha = 50002;
const ErrCaptcha = "验证码错误";
const ErrUserPasswordEmpty = "账号或密码为空";
const ErrUserPasswordError = "账号或密码错误";

export default Vue.extend({
  props: {
    height: {
      type: String,
      default: "200px",
    },
    backend: {
      type: String,
    },
  },
  data() {
    return {
      signForm: {
        name: "",
        password: "",
      },
      captchaForm: {
        answer: "",
      },
      captchaQuestion: "",
      captchaType: "",
      requireCaptcha: false,
      captchaError: "",
      passwordError: "",
      signRules: {
        name: [
          {
            required: true,
            message: "请输入账号",
            trigger: "blur",
          },
        ],
        password: [
          {
            required: true,
            message: "请输入密码",
            trigger: "blur",
          },
        ],
      },
      captchaRules: {
        answer: [
          {
            required: true,
            message: "请输入验证答案",
            trigger: "blur",
          },
        ],
      },
    };
  },
  created() {
    this.refreshCaptcha();
  },
  methods: {
    async onSubmitVerify() {
      // 验证表单
      if (!(await this.verifyCaptchaForm())) {
        return;
      }
      try {
        // 验证验证码
        await this.submitVerify();
        // 验证密码
        await this.submitPassword();
        // 登录成功
      } catch (error) {
        console.log(error);
      }
    },
    async onSubmit() {
      // 验证表单
      if (!(await this.verifySignForm())) {
        return;
      }

      // 如果需要验证码就跳过去验证
      if (this.requireCaptcha) {
        this.setCurrentForm("captcha");
        return;
      }

      try {
        // 验证密码
        await this.submitPassword();
      } catch (error) {
        console.log(error);
      }
    },
    verifySignForm() {
      this.passwordError = "";
      return new Promise<boolean>((resolve, reject) => {
        (this.$refs.signForm as ElForm).validate((err) => {
          if (err === false) {
            reject(err);
            return;
          }
          resolve(err);
        });
      });
    },
    verifyCaptchaForm() {
      this.captchaError = "";
      return new Promise<boolean>((resolve, reject) => {
        (this.$refs.captchaForm as ElForm).validate((err) => {
          if (err === false) {
            reject(err);
            return;
          }
          resolve(err);
        });
      });
    },
    async submitVerify() {
      if (this.requireCaptcha) {
        const err = await this.verifyCaptcha();
        if (err != 0) {
          this.setCaptchaError();
          throw new Error(ErrCaptcha);
        }
      }
    },
    async submitPassword() {
      const err = await this.verifyPassword();
      if (err === ErrCodeCaptcha) {
        this.setCaptchaError();
        throw new Error(ErrCaptcha);
      }
      if (err === ErrCodeUserPasswordEmpty) {
        this.setUserPasswordError(ErrUserPasswordEmpty);
        throw new Error(ErrUserPasswordEmpty);
      }
      if (err === ErrCodeUserPasswordError) {
        this.setUserPasswordError(ErrUserPasswordError);
        throw new Error(ErrUserPasswordError);
      }
      this.setCurrentForm("success");
      this.$emit("success");
    },
    setCaptchaError() {
      this.setCurrentForm("captcha");
      this.captchaError = ErrCaptcha;
    },
    setUserPasswordError(message: string) {
      this.setCurrentForm("password");
      this.passwordError = message;
    },
    setCurrentForm(name: string) {
      const signFrom = this.$refs["sign"] as ElCarousel;
      signFrom.setActiveItem(name);
    },
    async refreshCaptcha() {
      await this.$axios
        .$request({
          method: "GET",
          url: "/captcha",
          baseURL: this.backend,
        })
        .then(
          (res: {
            data: { required: boolean; type: string; data: string };
          }) => {
            const result = res.data;
            this.requireCaptcha = result.required;
            this.captchaQuestion = result.data;
            this.captchaType = result.type;
          }
        )
        .catch((err) => {
          const data = err?.response?.data || err;
          this.$emit("error", data);
        });
    },
    async verifyCaptcha() {
      return await this.$axios
        .$request({
          method: "POST",
          url: "/verify_captcha",
          baseURL: this.backend,
          data: {
            answer: this.captchaForm.answer,
          },
        })
        .then(() => {
          this.requireCaptcha = false;
          return 0;
        })
        .catch((err) => {
          this.refreshCaptcha();
          const data = err?.response?.data || err;
          if (data?.code > 0) {
            return data.code;
          }
          this.$emit("error", data);
        });
    },
    async verifyPassword() {
      const data = await this.$axios
        .$request({
          method: "POST",
          url: "/signin",
          baseURL: this.backend,
          withCredentials: true,
          data: {
            name: this.signForm.name,
            password: this.signForm.password,
          },
        })
        .then(() => {
          return 0;
        })
        .catch((err) => {
          this.refreshCaptcha();
          const data = err?.response?.data || err;
          if (data?.code > 0) {
            return data.code;
          }
          this.$emit("error", data);
        });
      return data;
    },
  },
});
</script>

<style scoped>
</style>