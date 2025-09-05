<template>
  <div id="app">
    <NuxtLayout> <!--前頁共通のフォーマットや書式を設定。デフォは layouts/default.vue を指定。name属性で変更可能。-->
      <NuxtPage /> <!--最初の頁を指定。デフォは pages/index.vue を指定。-->
    </NuxtLayout>
  </div>
</template>