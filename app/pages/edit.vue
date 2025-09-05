<!--********** /* Todoアプリ 更新専用ページ設定 */ **********-->

<template>
  <!--Todoアプリタイトル-->
  <h1>Todo管理アプリ</h1>

  <!--境界線-->
  <hr id="hline">

  <!--Todoリストの更新-->
  <h2>Todoの更新</h2>

  <div class="editTodoList">
    <p>
      <label for="taskNameForm" class="taskName" name="taskNameInput">
        対象Todo<input v-model="afterTaskName" name="taskName" type="text" id="taskNameForm" placeholder="50字以内で記述" /> 
      </label>
      <label for="taskDeadlineForm" class="taskDeadline" name="taskDeadlineInput">
        期限（３か月以内）<input v-model="afterTaskDeadline" name="taskDeadline" type="date" min="" max="2099-12-31" id="taskDeadlineForm"/> <!-- max=2099-12-31 のおかげでyyyy入力のみ受け付ける-->        
      </label>
      <button id="updateButton" @click="validate($event)">更新</button>
      <div class="errorOfExpired" v-if="judgeExpired(beforeTaskDeadline)">
        <div :class="{ 'errorMsg': judgeExpired(beforeTaskDeadline)}">！Todoの期限が切れています！</div> <!--初期状態で期限切れの場合のみ警告文表示-->
      </div>
    </p>
    <div class="errorMsg">
      <p v-if="errorTaskName">{{ errorTaskName }}</p>        
      <!-- <p id="errorMsgTaskDeadline"></p> <- 古典的やり方の残骸 --> 
      <p v-if="errorTaskDeadline">{{ errorTaskDeadline }}</p>
      <p v-if="errorTaskDuplication">{{ errorTaskDuplication }}</p>
    </div>
  </div>
</template>

<!--==============================================================================-->

<!--********** /*スクリプト*/ **********-->
<script setup lang="ts">
const route = useRoute();
const router = useRouter();
const query = ref(route.query);

// console.log(query.value.taskName); //Debug用
// console.log(query.value.taskDeadline); //Debug用

const beforeTaskDeadline = query.value.taskDeadline; //期限切れ判定用

/*----------------------------------------------------------------------------------*/

//リアクティブ型空配列の用意
const afterTaskName = ref<String>(query.value.taskName); //初期値を設定
const afterTaskDeadline = ref<String>(query.value.taskDeadline.split("/").join("-")); //初期値を設定（日付形式は初期設定に戻す）
const errorTaskName = ref<string>(""); //なぜ、こっちはletなの？
const errorTaskDeadline = ref<string>("");
const errorTaskDuplication = ref<string>("");

/*----------------------------------------------------------------------------------*/

/*===DBデータ取得用===*/
const { data, refresh } = await useAsyncData(
  () => $fetch('/api/db', { 
    method: 'GET' ,
  }
  ),
);

/*----------------------------------------------------------------------------------*/

/*===日付取得用===*/

const today = new Date() // 今日の日付

const minDate = new Date(today) // 日付の下限は今日

const maxDate = new Date(today) // 日付の上限は
maxDate.setMonth(maxDate.getMonth() + 3) // 今日の3か月後

/*YYYY-MM-DD形式にフォーマット*/
const formatDate = (d: Date) => {
  const year = d.getFullYear()
  const month = d.getMonth() + 1 // month の index は 0~11
  const day = d.getDate()

  let checkDigits = function (num, digit) { //mm が一桁の場合、二桁になるように調整
    num += ''
    if (num.length < digit) {
      num = '0' + num
    }
    return num
  }

  let yyyy = checkDigits(year, 4)
  let mm = checkDigits(month, 2)
  let dd = checkDigits(day, 2)

  return `${yyyy}-${mm}-${dd}`
}

/*期限の範囲設定*/
onMounted(() => { // DOMがマウントされた後にmin/max属性を設定
  document.getElementById('taskDeadlineForm').min = formatDate(minDate)
})

/*----------------------------------------------------------------------------------*/

/*更新機能用*/
const updateTask = async () => {
  await $fetch('/api/db', {
  method: 'PUT',
  body: JSON.stringify({ 
    taskId : query.value.id,
    beforeTaskName : query.value.taskName, 
    beforeTaskDeadline : query.value.taskDeadline, 
    afterTaskName : afterTaskName.value as string,
    afterTaskDeadline : afterTaskDeadline.value.split("-").join("/"),
  })
  });

  router.back()
}

/*----------------------------------------------------------------------------------*/

/*===入力チェック用===*/

//Todo内容に関するバリデーションチェック設定
type taskNameValidationResult = { //初期値設定（オブジェクト型）
  NisValid: boolean;
  NerrorMessage?: string;
};

/*エラー有無判定関数（Todo内容）*/
function taskNameValidator(taskName: string): taskNameValidationResult {
  // console.log(value || null);
  if (taskName.length === 0) {
    return {
      NisValid: false,
      NerrorMessage: 'Todo内容の入力は必須です。'
    };
  }
  else if (taskName.length > 50) {
    return {
      NisValid: false,
      NerrorMessage: 'Todo内容は50文字以内で入力してください。'
    };
  }
  return { NisValid: true };
}

//Todo期限に関するバリデーションチェック設定
type taskDeadlineValidationResult = { //初期値設定（オブジェクト型）
  DisValid: boolean;
  DerrorMessage?: string;
};

/*エラー判定関数（Todo期限）*/
function taskDeadlineValidator(taskDeadline: Date): taskDeadlineValidationResult {
  // console.log(taskDeadline);
  // console.log(formatDate(maxDate))
  if (taskDeadline === "") { // type="date" のとき、未入力時は ""(空文字) 
    return {
      DisValid: false,
      DerrorMessage: 'Todo期限に正しい日付を入力してください。'
    };
  }
  else if (taskDeadline < formatDate(minDate) || taskDeadline > formatDate(maxDate)) {
    return {
      DisValid: false,
      DerrorMessage: 'Todo期限は ' + formatDate(minDate).split("-").join("/") + ' から '+ formatDate(maxDate).split("-").join("/") + ' で設定してください。'
    };
  }

  return { DisValid: true };
}

//Todo重複に関するバリデーションチェック設定
type taskDuplicatonValidationResult = { //初期値設定（オブジェクト型）
  DuplicationIsValid: boolean;
  DuplicationErrorMessage?: string;
};

/*エラー判定関数（Todo重複）*/
function taskDuplicationValidator(DuplicatedTaskName: String, taskId: number): taskDuplicationValidationResult {
  // console.log(taskDeadline);
  // console.log(formatDate(maxDate))
  for (let task of data.value) {
    if (DuplicatedTaskName === task.taskName && !task.isDone && taskId != task.id) {  //異なるidで同じ内容のTodoで未完了のものがあれば更新不可
      return {
        DuplicationIsValid: false,
        DuplicationErrorMessage: '更新できません。未完了の同じTodoがすでに登録されています。'
      };
    }
  }
  return { DuplicationIsValid: true };
}

/*エラー判定処理実行関数*/
function validate(event: Event) {
  event.preventDefault(); //既定のクリック処理のブロック

  // Todo内容に関するバリデーションチェック
  let inputTaskName: String = afterTaskName.value;
  const resultTaskName: {NisValid:boolean, NerrorMessage:String} = taskNameValidator(inputTaskName);

  errorTaskName.value = resultTaskName.NisValid ? '' : resultTaskName.NerrorMessage; //バリデーション結果（偽ならエラー文を登録）


  // Todo期限に関するバリデーションチェック
  let inputTaskDeadline = afterTaskDeadline.value;
  const resultTaskDeadline: {DisValid:boolean, DerrorMessage:String} = taskDeadlineValidator(inputTaskDeadline);

  errorTaskDeadline.value = resultTaskDeadline.DisValid ? '' : resultTaskDeadline.DerrorMessage; //バリデーション結果（偽ならエラー文を登録）

  // Todo重複に関するバリデーションチェック
  const resultTaskDuplicated: {DuplicationIsValid:boolean, DuplicationErrorMessage:String} = taskDuplicationValidator(inputTaskName, query.value.id);

  errorTaskDuplication.value = resultTaskDuplicated.DuplicationIsValid ? '' : resultTaskDuplicated.DuplicationErrorMessage; //バリデーション結果（偽ならエラー文を登録）

  if (resultTaskName.NisValid && resultTaskDeadline.DisValid && resultTaskDuplicated.DuplicationIsValid) {
    updateTask(); // バリデーションが通ったらDB更新処理へ
  }
}

/*===期限切れ判定用===*/
function judgeExpired(taskDeadline: Date) {
  // console.log(taskDeadline)
  return (taskDeadline < formatDate(today).split("-").join("/"));
}

</script>

<!--********** /* Todoアプリ 更新専用ページ設定  END*/ **********-->