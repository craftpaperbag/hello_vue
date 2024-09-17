<script setup>
import { ref, watch, computed, onMounted } from 'vue'

const title = ref('todol')

let _items = []
const itemTemplate = {
  name: '',
  active: false,
  done: false,
  edit: false
}

/**
 * ダミーデータの設定
 */

Array.from({length: 5}).forEach((element, i) => {
  _items.push({...itemTemplate})
  _items[i].name = 'タスク' + i
})

_items[1].active = true
_items[2].edit = true

const items = ref(_items)
const filterStatusMessage = {
  all: '全て',
  filterDoneItems: '未完了のみ'
}
const filterStatus = ref(filterStatusMessage.all)

const filterDoneItems = ref(false)
watch(filterDoneItems, async()=>{
  filterStatus.value = filterDoneItems.value ?
  filterStatusMessage.filterDoneItems : filterStatusMessage.all
})

function activate(i) {
  console.log('activate('+i+')')
  items.value.forEach((e) => {e.active = false})
  items.value[i].active = true
}

function handleClickItem(clickedItem, index) {
  console.log('handleClickItem')
  activate(index)
}

async function focus(position) {
  // DOMが追加されるまで待機
  await document.getElementById('inputItemNameForItem0')
  const inputElement = document.getElementById('inputItemNameForItem' + position)
  inputElement.focus()
}
/**
* position...追加する位置（任意）
*/
async function addItem(position) {
  console.log('addItem('+position+')')
  if (!position) position = 0
  let _item = {...itemTemplate}
  _item.edit = true
  items.value.splice(position, 0, _item)
  // 追加されたinput要素にフォーカスを当てる
  focus(position)
}

function deleteItem(deletedItem, index) {
  const confirmed = confirm('「' + deletedItem.name + '」を削除しますか？')
  if (confirmed) {
    items.value.splice(index,1)
  }
  
}

/**
 * activeタスクを編集モードにする
 * activeタスクがない場合は何もしない
 */
async function editActiveItem() {
  for (let i=0; i < items.value.length; i++) {
    if (items.value[i].active) {
      items.value[i].edit = true
      focus(i)
      break
    }
  }
}

/**
* editのwatch用
*/
const editChanges = computed(() => {
  return items.value.map(item => item.edit)
})

/**
* editがtrueからfalseになったときactivateする
*/
watch(editChanges,(newValue, oldValue) => {
  console.log('watch(items) deep: true')
  console.log(newValue)
  console.log(oldValue)
  for (let i=0; i < newValue.length; i++){
    if (newValue[i] == false && oldValue[i] == true) {
      activate(i)
      break
    }
  }
},{ deep: true })

onMounted(() => {
  // enterを押すとactiveタスクの編集を開始する
  document.addEventListener('keydown', function(event) {
    if (event.key === 'Enter') {
        // イベントが発生した要素がinput要素でない場合のみ
        if (!event.target.matches('input')) {
            // Enterキーが押された時の処理
            editActiveItem()
        }
    }
})
})

</script>

<template>
  <div class="container">
    <div class="row mb-2">
      <div class="col">
        <h1>{{ title }}</h1>
      </div>
      <div class="col d-flex justify-content-end">
        <button class="btn"
        @click="filterDoneItems=!filterDoneItems">
        <label v-if="filterDoneItems">
          全て表示する
        </label>
        <label v-else>
          完了を隠す
        </label>
      </button>
      <button class="btn"
      @click="addItem">
      <i class="bi-plus-circle"></i>
    </button>
  </div>
</div>
<div class="row">
  <div class="col">
    <span class="text-muted">
      {{ filterStatus }}
    </span>
    <div class="list-group">
      <template v-for="(item, index) in items" >
        <!-- 編集モード -->
        <template v-if="item.edit">
          <div class="list-group-item">
            <div class="input-group">
              <input
              type="text"
              v-model="item.name"
              class="form-control"
              :id="'inputItemNameForItem'+index"
              placeholder="何をしますか？"
              @keydown.enter="(e)=>{e.isComposing ? 'nothing to do' : item.edit=false}"
              @keydown.shift.enter="(e)=>{e.isComposing ? 'nothing to do' : addItem(index+1)}"
              >
              <button
              class="btn btn-outline-secondary"
              @click.stop="item.edit=false"
              value="保存"
              >
              <i class="bi-check"></i>
              
            </button>
          </div>
        </div>
      </template>
      <!-- 通常モード -->
      <template v-else>
        <a
        v-show="!filterDoneItems||!item.done"
        class="list-group-item list-group-item-action"
        :class="{ 'active-item': item.active }"
        @click="handleClickItem(item, index)">
        <input
        class="form-check-input me-1"
        type="checkbox"
        :id="'checkbox'+index"
        @click.stop
        v-model="item.done">
        <label
        class="form-check-label"
        :class="{'done-item': item.done}"
        :for="'checkbox'+index"
        @click.stop>
        {{ item.name }}
      </label>
      <button class="btn btn-sm edit-button ml-5"
      v-if="!item.done"
      @click.stop="item.edit=true"
      value="編集"
      >
      <i class="bi-pencil-fill"></i>
    </button>
    <button class="btn btn-sm delete-button"
    @click.stop="deleteItem(item, index)"
    value="削除"
    >
    <i class="bi-trash-fill"></i>
  </button>
</a>
</template>
</template>
</div>
</div>
</div>
</div>
</template>



<style scoped>
input::placeholder {
  color: lightgrey;
}

.active-item {
  background-color: lightskyblue;
}

.done-item {
  text-decoration: line-through;
}
.edit-button {
  color: darkgrey;
}
.edit-button:hover {
  color: blue;
}

.delete-button {
  color: darkgrey;
}
.delete-button:hover {
  color: red;
}
</style>
