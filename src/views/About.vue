<template>
  <div class="about">
    <h1>This is an about page</h1>
    <img :src="image" alt="" />
  </div>
</template>
<script>
import MODEL_PATH_LOCAL from "../assets/logo.png";
export default {
  data() {
    return {
      hello: "hello",
      image: MODEL_PATH_LOCAL
    };
  }
};
</script>
