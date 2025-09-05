<!--********** /* Todoアプリ メインページ設定 */ **********-->

<template>
  <!--Todoアプリタイトル-->
  <h1>Todo管理アプリ</h1>

  <!--Todoリストの登録-->
  <div class="inputArea">
    <form>
      <p>
        <label for="taskNameForm" class="taskName" name="taskNameInput">
          新規Todo<input v-model="taskName" name="taskName" type="text" id="taskNameForm" placeholder="50字以内で記述" @input="detectRemovedStr($event)" @blur="stringCheck($event)" />
          <!-- 新規Todo<input v-model="taskName" name="taskName" type="text" id="taskNameForm" placeholder="50字以内で記述" /> -->
        </label>
        <label for="taskDeadlineForm" class="taskDeadline" name="taskDeadlineInput">
          期限（３か月以内）<input v-model="taskDeadline" name="taskDeadline" type="date" min="" max="2099-12-31" id="taskDeadlineForm" /> <!-- max=2099-12-31 のおかげでYYYY入力のみ受け付け、YYYYYY入力を受け付けない-->        
        </label>
        <button id="postButton" @click="validate($event)">追加</button> <!--イベント内容を関数に受け渡す-->
      </p>
      <div class="errorMsg">
        <p v-if="errorTaskName">{{ errorTaskName }}</p>        
        <!-- <p id="errorMsgTaskDeadline"></p> <- 古典的やり方の残骸 --> 
        <p v-if="errorTaskDeadline">{{ errorTaskDeadline }}</p>
        <p v-if="errorTaskDuplication">{{ errorTaskDuplication }}</p>
      </div>
    </form>
  </div>

  <!--境界線-->
  <hr id="hline">

  <!--Todoリストの表示-->
  <h2>Todoリスト</h2>
  <div class="self-header">
    <nav>
      <ul class="nav-list">
        <!-- <li id="item1">完了</li> -->
        <li id="item2">Todo内容</li>
        <li id="item3">期限</li>
      </ul>
    </nav>
  </div>
  <div class="todoList">
    <div v-for= "taskData in data" :key="taskData.id">
      <div :class="{ 'errorMsg': judgeExpired(taskData.taskDeadline, taskData.isDone)}">
        <div class="flexItem"> <!--要素を横１行に配置-->
          <input id="checkbox" type="checkbox" v-model="taskData.isDone" @change="isDone(taskData.id, taskData.isDone)" />
          <div :class="{ 'grayed-out': taskData.isDone }"> <!--完了時、グレーアウト-->
            <div class="outputTask">{{ taskData.taskName }}</div> 
            <div class="outputDeadline">{{ taskData.taskDeadline }}</div>
            <button id="editButton" @click="editTask(taskData.id, taskData.taskName, taskData.taskDeadline)" :disabled="taskData.isDone">編集</button> 
            <button id="deleteButton" @click="deleteTask(taskData.id)" :disabled="taskData.isDone">削除</button>
            <div class="errorOfExpired" v-if="judgeExpired(taskData.taskDeadline, taskData.isDone)">！Todoの期限が切れています！</div>
          </div>
        </div>
      </div>
    </div>
  </div>
  <div class="announcement">
    <p>
      完了したら、チェックボックスにチェックを付けてください。
      <br>
      再度チェックボックスを押すと、Todo完了を取り消すことができます。
    </p>
  </div>
</template>

<!--==============================================================================-->

<!--********** /*スクリプト*/ **********-->
<script  setup lang="ts">

const router = useRouter()

//リアクティブ型空配列の用意
// const taskNameList = ref<String[]>([]); 
const taskName = ref<String>(""); 
const taskDeadline = ref<String>("");
const errorTaskName = ref<String>("");
const errorTaskDeadline = ref<String>("");
const errorTaskDuplication = ref<String>("");

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
const today: Date = new Date() // 今日の日付

const minDate: Date = new Date(today) // 日付の下限は今日

const maxDate: Date = new Date(today) // 日付の上限は
maxDate.setMonth(maxDate.getMonth() + 3) // 今日の3か月後

/*YYYY-MM-DD形式にフォーマット*/
const formatDate = (d: Date) => {
  const year: Number = d.getFullYear()
  const month: Number = d.getMonth() + 1 // month の index は 0~11
  const day: Number = d.getDate()

  let checkDigits = function (num, digit) { //入力した年月日いずれかが、指定した桁数になるように調整　例）MMとして 1 が入力された場合、01 となるように変換
    num += '' //String型への変換
    if (num.length < digit) {
      num = '0' + num
    }
    return num
  }

  let YYYY: String = checkDigits(year, 4)
  let MM: String = checkDigits(month, 2)
  let DD: String = checkDigits(day, 2)

  return `${YYYY}-${MM}-${DD}`
  // return `${year}-${month}-${day}` ###2桁MM でないと、正しく認識されない###
}

/*期限の範囲設定*/
onMounted(() => { // DOMマウント時に <input>の min属性 設定
  document.getElementById('taskDeadlineForm').min = formatDate(minDate)
})

/*----------------------------------------------------------------------------------*/

/*===追加機能用===*/
async function postTask () {
  //[taskObj: {task:△△, deadline:YYYY/MM/DD}, {}, {}, ・・・] 形式
  let taskObj:{isDone:boolean, task:string, deadline:string} = {isDone: false, task: taskName.value as string, deadline:taskDeadline.value.split("-").join("/")}; 
  
  /*新規Todoの登録*/
  await $fetch('/api/db', {
  method: 'POST',
  body: taskObj
  }
  );

  refresh();

  //リアクティブ型変数の初期化
  taskName.value = "";
  taskDeadline.value = "";
  errorTaskName.value = "";
  errorTaskDeadline.value = "";
  errorTaskDuplication.value = "";

};

/*----------------------------------------------------------------------------------*/

/*===完了機能用===*/

const isDone = async (doneTaskId: string, taskIsDone: boolean) => {
  await $fetch('/api/done/db', { 
    method: 'POST', 
    body: JSON.stringify({ 
      doneTaskId: doneTaskId,
      taskIsDone: taskIsDone,
    })
  });

  refresh();

  //リアクティブ型変数の初期化
  // taskName.value = "";
  // taskDeadline.value = "";
  errorTaskName.value = "";
  errorTaskDeadline.value = "";
  errorTaskDuplication.value = "";

};

/*----------------------------------------------------------------------------------*/

/*===編集機能用===*/
function editTask(editTaskId: number, editTaskName: string, editTaskDeadline: string) {
  router.push({
    path: "/edit" ,
    query: {
        id: editTaskId,
        taskName: editTaskName,
        taskDeadline: editTaskDeadline,
    } 
  });
}

/*----------------------------------------------------------------------------------*/

/*===削除機能用===*/
const deleteTask = async (deletedID: string) => {
  await $fetch('/api/db', {
    method: 'DELETE',
    body: JSON.stringify({ taskId : deletedID }), /* fetchは本来、JSON形式(連想配列)の文字列型で受け渡し*/
    }
  );
  
  refresh();

  //リアクティブ型変数の初期化
  taskName.value = "";
  taskDeadline.value = "";
  errorTaskName.value = "";
  errorTaskDeadline.value = "";
  errorTaskDuplication.value = "";

};

/*----------------------------------------------------------------------------------*/

//IME51字以上強制制御
// function onInput(event: Event) {
//   let prevValue: String = event.target.value;
//   if (event.target.value.length > 50) {
//     // event.target.blur()
//     event.target.value = prevValue
//   }
//   else {
//     prevValue = event.target.value 
//   }
// }

/*@NOTE -- haraoka

###############################################
#　　文字コードチェックは設定不完全状態           #
#　　・最後の追加　　　　　　　　　　　　　　　　　#
#　　・途中の追加　　　　　　　　　　　　　　　　　#
#　　・複数の追加　　　　　　　　　　　　　　　　　#
#　　それぞれ対応できるようにしコード作成が必要    #
#　　getInsertedSegments が未完成               #
###############################################

*/

let preValue = ""; // 入力要素一時保存用
let preservedTaskName = ref<String>("");

function detectRemovedStr(event : Event) {
  // console.log(preservedTaskName)
  // if (event.target.value.length < preservedTaskName.value.length)  {
    preservedTaskName.value = event.target.value;
    console.log("detectRemovedStr後：" + preservedTaskName.value)
  // }  
}

/*入力文字チェック用*/ //デコード時発火
import {encode, decode} from 'iconv-lite';


function getInsertedSegments(oldStr: string, newStr: string): { index: number, text: string }[] {
  const inserts: { index: number, text: string }[] = [];

  // if (preValue != preservedTaskName) {
    console.log("true")
    preValue = preservedTaskName.value
  // }

  console.log(preValue)

  let i = 0, j = 0;
  // for (let j: number = 0; j < newStr.length; j++) {
  while (j < newStr.length) {
    // for (let i: number = 0; i < oldStr.length; i++) {
      if (oldStr[i] === newStr[j]) {
        i++; j++; // 一致している → 両方進める
        // continue;
      }
      else {
        // 挿入された文字の検出開始
        const insertStart = j;
        let insertedText = "";
        while (j < newStr.length && oldStr[i] !== newStr[j]) {
          insertedText += newStr[j];
          j++;
        }
        inserts.push({ index: insertStart, text: insertedText });
      }
    }
  // }
  console.log(inserts)
  return inserts;
}

function stringCheck(event: Event) {

  const prev = preValue;
  const current = event.target.value;

  console.log(prev);
  console.log(current);
  const inserts = getInsertedSegments(prev, current);

  for (const { index, text } of inserts) {
    console.log("in for")
    const encoded = encode(text, 'cp932');
    const decoded = decode(encoded, 'cp932');

    for (let i = 0; i < text.length; i++) {
      if (text[i] !== decoded[i]) {
        alert(`CP932に対応していない文字があります（位置 ${index + i + 1}）：${text[i]}`);
        return prev;
      }
    }
  }

  preValue = current;

  return current;
}

/*==========*/

// import { encode, decode } from 'iconv-lite';

// let preValue = ref<String>(""); // 入力要素一時保存用

// function stringCheck(event: Event) {
// // function stringCheck() {

//   // Unicode → CP932
//   // const encodedValue: Buffer = encode(taskName.value, 'cp932'); //符号化
//   const encodedValue: Buffer = encode(event.target.value, 'cp932');
//   // CP932 → Unicode
//   const decodedValue: String = decode(encodedValue, 'cp932'); //復号

//   let erroredString: String = "" 

//   for(let n:number = 0 ; n < event.target.value.length ; n++) {
//     // if (taskName.value[n] != decodedValue[n]) {
//     if (event.target.value[n] != decodedValue[n]) {
//       erroredString += event.target.value[n];
//       if (n < event.target.value.length - 1 ) {
//         erroredString += "  ";
//       }
//     }
//   }

//   if(erroredString != "") {
//     alert("以下の文字は使用できません。(  " + erroredString + "  )")
//     taskName.value = preValue; //入力要素を直前のものに修正
//   }
//   else{
//     preValue = decodedValue; //一次保存用変数の更新
//     // preValue = taskName.value;
//   }

// }  

/*----------------------------------------------------------------------------------*/

/*===入力チェック用===*/

//Todo内容に関するバリデーションチェック設定
type taskNameValidationResult = { //初期値設定（オブジェクト型）
  NisValid: boolean;
  NerrorMessage?: string;
};

/*エラー判定関数（Todo内容）*/
function taskNameValidator(isCheckedTaskName: string): taskNameValidationResult {
  // console.log(value || null);
  if (isCheckedTaskName.length === 0) {
    return {
      NisValid: false,
      NerrorMessage: 'Todo内容の入力は必須です。'
    };
  }
  else if (isCheckedTaskName.length > 50) {
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
function taskDeadlineValidator(isCheckedTaskDeadline: Date): taskDeadlineValidationResult {
  // console.log(taskDeadline.value)
  // console.log(isCheckedTaskDeadline);
  // console.log(formatDate(maxDate))
  if (isCheckedTaskDeadline === "") { // type="date" のとき、未入力時は ""(空文字) 
    return {
      DisValid: false,
      DerrorMessage: 'Todo期限に正しい日付を入力してください。'
    };
  }
  else if (isCheckedTaskDeadline < formatDate(minDate) || isCheckedTaskDeadline > formatDate(maxDate)) {
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
function taskDuplicationValidator(isDuplicatedTaskName: String): taskDuplicationValidationResult {
  // console.log(taskDeadline);
  // console.log(formatDate(maxDate))
  for (let task of data.value) {
    if (isDuplicatedTaskName === task.taskName && !task.isDone) { 
      return {
        DuplicationIsValid: false,
        DuplicationErrorMessage: '同じTodoが登録されています。先に完了にしてください。'
      };
    }
  }
  return { DuplicationIsValid: true };
}

/*エラー判定処理実行関数*/
function validate(event: Event) {
  event.preventDefault(); //既定のクリック処理のブロック

  // Todo内容に関するバリデーションチェック
  let inputTaskName: String = taskName.value;
  const resultTaskName: {NisValid:boolean, NerrorMessage:String} = taskNameValidator(inputTaskName);
  errorTaskName.value = resultTaskName.NisValid ? '' : resultTaskName.NerrorMessage; //バリデーション結果（偽ならエラー文を登録）


  // Todo期限に関するバリデーションチェック
  let inputTaskDeadline = taskDeadline.value;
  const resultTaskDeadline: {DisValid:boolean, DerrorMessage:String} = taskDeadlineValidator(inputTaskDeadline);
  errorTaskDeadline.value = resultTaskDeadline.DisValid ? '' : resultTaskDeadline.DerrorMessage; //バリデーション結果（偽ならエラー文を登録）

  // Todo重複に関するバリデーションチェック
  const resultTaskDuplicated: {DuplicationIsValid:boolean, DuplicationErrorMessage:String} = taskDuplicationValidator(inputTaskName);
  errorTaskDuplication.value = resultTaskDuplicated.DuplicationIsValid ? '' : resultTaskDuplicated.DuplicationErrorMessage; //バリデーション結果（偽ならエラー文を登録）

  if (resultTaskName.NisValid && resultTaskDeadline.DisValid && resultTaskDuplicated.DuplicationIsValid) {
    postTask(); // バリデーションが通ったらDB登録処理へ <- 引数不要。　変数名.value でいつでもデータ取得可能。
  }
}

/*----------------------------------------------------------------------------------*/

/*===期限切れ判定用===*/
function judgeExpired(taskDeadline: Date, taskIsDone: boolean) {
  return (taskDeadline < formatDate(today).split("-").join("/") && !taskIsDone);
}

</script>

<!-- ********** /* Todoアプリ メインページ設定 END */ ********** -->