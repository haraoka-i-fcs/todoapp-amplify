<template>
  <!--Todoリストの登録-->
  <div class="title">
    <NuxtRouteAnnouncer />
    
    <h1>タイトル</h1>
    <div class="inputArea">
      <p>
        <label for="new">新規Todo
          <input v-model="taskName" />
          <button id="addButton" @click="addTask()">追加</button><!--JavaScript ボタン押下で動的処理（ここでは、addTask関数の実行）-->
        </label>
      </p>
    </div>
  </div>


  <!--境界線-->
  <hr id="hline">

  <!--Todoリストの表示-->
  <div class="list">
    <h2>Todoリスト</h2>
    <div class="modifyButton" v-for="taskName in taskNameList" :key="taskName"><!--JavaScript v-for で俗にいう for文処理の実行）-->
      {{ taskName }} <!--変数の出力-->
      <button @click="modifyTask(taskName)" id="modifybutton">編集</button><!--JavaScript ボタン押下で動的処理（ここでは、modifyTask関数の実行）-->
      <button @click="deleteTask(taskName)">削除</button><!--JavaScript ボタン押下で動的処理（ここでは、deleteTask関数の実行）-->
    </div>
  </div>
</template>

<!--この分はJavaScript???-->
<script  setup lang="ts">
const taskNameList = ref<String[]>([]); /*リアクティブな空の配列(String[])を用意*/
const taskName = ref<String>(""); /*リアクティブな空の文字列を用意*/

/*追加用*/
const addTask = () => {
  if (taskName.value === '') { /*入力ボックスが空だった場合*/
    return;
  }
  taskNameList.value.push(taskName.value); /*入力ボックスが空でなかった場合 -> リアクティブ型の配列に要素を追加 -> 自動的に反映される*/
  taskName.value = ""; /*値の初期化*/
}

/*修正用　何修正する仕様なの？　【現状は押下しても何も起きない仕組みにしている】*/
const modifyTask = (deletedTaskName: string) => {
  /*taskNameList.value = taskNameList.value.filter((taskName) => deletedTaskName !== taskName);*/
}

/*削除用*/
const deleteTask = (deletedTaskName: string) => {
  taskNameList.value = taskNameList.value.filter((taskName) => deletedTaskName !== taskName); /*deletedTaskName と等しくない taskName だけのリアクティブ配列をセットする*/
}
</script>