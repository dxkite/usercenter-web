<template>
  <div>
    <SigninFrom
      :backend="this.backend"
      @error="onSignError"
      @success="onSignSuccess"
    />
  </div>
</template>

<script lang="ts">
import Vue from "vue";
import SigninFrom from "@/components/SigninForm.vue";

export default Vue.extend({
  data: () => {
    return {
      // backend: "http://127.0.0.1:2333/user",
      backend: "https://api.dxkite.cn/user/",
    };
  },
  components: {
    SigninFrom,
  },
  methods: {
    onSignError(err: { message: string }) {
      console.log(err);
      this.$notify.error(err.message || "系统繁忙，请稍后再试");
    },
    onSignSuccess() {
      this.$notify.success("登录成功");
    },
  },
});
</script>
