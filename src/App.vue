<script setup>
import { ref, watch, computed, onMounted } from 'vue'

/**
 * キー操作の定義
 */
const keyHandlers = [
  {
    shift: true,
    exceptInput: true,
    key: 'Enter',
    handle: () => {
      addItem(getActiveIndex()+1)
    }
  },
  {
    exceptInput: true,
    key: 'Enter',
    handle: editActiveItem
  },
  {
    shift: true,
    key: 'Tab',
    handle: promoteActiveItem
  },
  {
    key: 'Tab',
    handle: demoteActiveItem
  },
  {
    key: 'ArrowRight',
    handle: demoteActiveItem
  },
  {
    key: 'ArrowLeft',
    handle: promoteActiveItem
  },
  {
    key: 'ArrowDown',
    handle: activateBelow
  },
  {
    key: 'ArrowUp',
    handle: activateAbove
  },
  {
    key: 'Escape',
    handle: () => {
      // フォーカスの当たっているinputがあれば、editを終了する
      const activeId = document.activeElement.id
      if (activeId) {
        const editAndFocusItemIndex = getIndexById(activeId)
        finishEdit(editAndFocusItemIndex)
      } else {
        // フォーカスの当たっているinputがない場合は。activeを解除する
        deactivate()
      }
    }
  }
]

/**
 * ヘルパー
 */

function generateItemNameInputTagId(index) {
  return 'itemNameInputFor' + index
}

function levelClassOf(item) {
  return 'level-' + item.level
}

function getIndexOf(targetItem) {
  for (let i=0; i < items.value.length; i++) {
    if (items.value[i] === targetItem) return i
  }
  return -1
}

function getIndexById(id) {
  return document.getElementById(id).getAttribute('index')
}

function getDescendants(item) {
  let d = []
  const index = getIndexOf(item)
  const parentLevel = item.level
  for (let i = index+1; i in items.value; i++) {
    if (items.value[i].level > parentLevel) {
      d.push(i)
    } else {
      break
    }
  }
  console.log('getDescendants(index: '+index+')')
  console.log(d)
  return d
}

/**
 * ref
 */

const title = ref('todol')

let _items = []
const itemTemplate = {
  level: 0,
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
_items[3].level = 1
_items[4].level = 2

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
  deactivate()
  items.value[i].active = true
}

function deactivate() {
  items.value.forEach((e) => {e.active = false})
}

function handleClickItem(clickedItem, index) {
  console.log('handleClickItem')
  activate(index)
}

function getActiveIndex() {
  for (let i=0; i<items.value.length; i++) {
    if (items.value[i].active) return i
  }
  return -1
}

function hasActive() {
  return getActiveIndex() >= 0 ? true : false
}

async function focus(position) {
  // DOMが追加されるまで待機
  const targetId = generateItemNameInputTagId(position)
  await document.getElementById(targetId)
  const inputElement = document.getElementById(targetId)
  inputElement.focus()
}
/**
* position...追加する位置（任意）
*/
async function addItem(position) {
  // 0より小さい数字の場合は無視する ※getActiveIndexの結果吸収のため
  if (position < 0) return
  // 何も指定されていない場合はゼロと解釈する
  if (!position) position = 0
  let _item = {...itemTemplate}
  _item.edit = true
  items.value.splice(position, 0, _item)
  // 追加されたinput要素にフォーカスを当てる
  focus(position)
}

function deleteItem(deletedItem, index) {
  let confirmed = confirm('「' + deletedItem.name + '」を削除しますか？')
  if ( ! confirmed ) return
  // 子タスクの削除確認
  const descendants = getDescendants(deletedItem)
  if (descendants.length > 0) {
    confirmed = confirm('子タスク（' + descendants.length + '件）も同時に削除します。よろしいですか？')
  }
  if ( ! confirmed ) return
  // 削除
  let deleteList = [index]
  for(let i=0; i in descendants; i++) {
    deleteList.push(descendants[i])
  }
  deleteList.sort((a, b) => { return b - a })
  for(let i=0; i in deleteList; i++) {
    console.log('delete index: ' + deleteList[i])
    items.value.splice(deleteList[i], 1)
  }
}

/**
 * 昇格・降格
 */
function promoteOrDemote(promoteMode, item, parentLevel, list) {
  const demoteMode = ! promoteMode
  let finish = false
  let first = false
  let index = -1
  if (list == null) list = []

  if (item == null) {
    finish = true
  } else {
    index = getIndexOf(item)
    // 初めて呼び出したか区別する
    if (parentLevel == null) {
      parentLevel = item.level
      first = true
    }
    // レベルが深すぎるものが見つかった場合は操作しない
    if (demoteMode && item.level >= 5) return
    // 先頭は昇格・降格できないのでこの時点で終了
    if (index == 0) return
    // 降格は一つ上のタスク＋１まで
    if (demoteMode && first && index-1 in items.value && items.value[index-1].level < item.level) {
      return
    }
    // レベル0は昇格できない
    if (promoteMode && item.level == 0) finish = true

  }
  if ( ! finish && (first || item.level > parentLevel) ) {
    // リストへ追加して一つ下のタスクへ伝播する
    list.push(index)
    promoteOrDemote(promoteMode, items.value[index+1], parentLevel, list)
  } else {
    // 実際の昇格・降格を行う
    const delta = promoteMode ? -1 : 1
    list.forEach(i => {
      if ( ! i in items.value ) return
      items.value[i].level += delta
    })
    // 実際の昇格・降格が終わったら終了
    return
  }
}
function promote(item) {
  promoteOrDemote(true, item)
}
function promoteActiveItem() {
  const activeIndex = getActiveIndex()
  if (activeIndex >= 0) promote(items.value[activeIndex])
}
function demote(item) {
  promoteOrDemote(false, item)
}
function demoteActiveItem() {
  const activeIndex = getActiveIndex()
  if (activeIndex >= 0) demote(items.value[activeIndex])
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
 * アクティブなタスクの変更
 */
function activateAbove() {
  activateAboveOrBelow(true)
}

function activateBelow() {
  activateAboveOrBelow(false)
}

function activateAboveOrBelow(above) {
  let pioneer = 0
  let delta = 1
  if (above) {
    delta = -1
    pioneer = items.value.length - 1
  }

  for (let i=0; i<items.value.length; i++) {
    if (items.value[i].active) {
      if (i+delta in items.value) {
        items.value[i].active = false
        items.value[i+delta].active = true
      }
      return
    }
  }
  // activeがない場合、一番上or一番下をアクティブにする
  items.value[pioneer].active = true

}

/**
 * indexで指定したアイテムのeditを終了する
 */
function finishEdit(index) {
  items.value[index].edit = false
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
  // 要素数が変わっている場合は、追加・削除のため、activateしない
  if (newValue.length != oldValue.length) return;

  for (let i=0; i < newValue.length; i++){
    if (newValue[i] == false && oldValue[i] == true) {
      activate(i)
      break
    }
  }
},{ deep: true })

onMounted(() => {
  /**
  * キー操作の設定
  */
  
  document.addEventListener('keydown', function(event) {
    let kh
    // inputで発生したのかどうか調べる
    const input = event.target.matches('input')
    for (let i=0; i in keyHandlers; i++) {
      kh = keyHandlers[i]
      if (kh.shift && ! event.shiftKey) continue
      if (kh.exceptInput && input) continue
      if (kh.key != event.key) continue
      kh.handle()
      return
    }
  })
  
})

</script>

<template>
  <div class="container">

    <!-- アプリケーション名とコントロール -->
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

<!-- タスク一覧及び状態 -->
<div class="row">
  <div class="col">

    <!-- フィルターの状態 -->
    <span class="text-muted">
      {{ filterStatus }}
    </span>

    <!-- タスク一覧 -->
    <div class="list-group">
      <template v-for="(item, index) in items" >

        <!-- 編集モード -->
        <template v-if="item.edit">
          <div
            class="list-group-item"
            :class="[{ 'active-item': item.active }, levelClassOf(item)]">
            <div class="input-group active-line">
              <input
                type="text"
                v-model="item.name"
                class="form-control"
                :id="generateItemNameInputTagId(index)"
                :index
                placeholder="何をしますか？"
                @keydown.enter="(e)=>{e.isComposing ? 'nothing to do' : item.edit=false}"
                @keydown.shift.enter="(e)=>{e.isComposing ? 'nothing to do' : addItem(index+1)}"
                >
              <button
                class="btn btn-primary"
                @click.stop="item.edit=false"
                value="保存"
                >
              <i class="bi-check-circle-fill"></i>
            </button>
          </div>
        </div>
      </template>

      <!-- 通常モード -->
      <template v-else>
        <a
          v-show="!filterDoneItems||!item.done"
          class="list-group-item list-group-item-action"
          :class="[{ 'active-item': item.active }, levelClassOf(item)]"
          @click="handleClickItem(item, index)">
          <div class="active-line">
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
              value="編集">
              <i class="bi-pencil-fill"></i>
            </button>
            <button class="btn btn-sm delete-button"
              @click.stop="deleteItem(item, index)"
              value="削除">
              <i class="bi-trash-fill"></i>
            </button>
          </div>
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

.active-line {
  border-left: 5px solid transparent;
}
.active-item .active-line {
  border-left: 5px solid royalblue;
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
.level-1 { padding-left:  3rem; }
.level-2 { padding-left:  6rem; }
.level-3 { padding-left:  9rem; }
.level-4 { padding-left: 12rem; }
.level-5 { padding-left: 15rem; }
</style>
