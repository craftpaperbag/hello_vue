<script setup>
import { ref, watch, computed, onMounted } from 'vue'

const VERSION = '0.0.0'
const LOCALSTORAGE_KEY_OF_ITEMS = 'todolItemsVersion' + VERSION
const DEBUG = true
let loadCompleted = false

/**
 * ロガー
 */
function debug(m) {
  if (DEBUG) console.log(m)
}

/**
 * キー操作の定義
 */
const keyUpHandlers = [
]
const keyDownHandlers = [
  {
    meta: true,
    handle: startMoving,
    continue: true
  },
  {
    shift: true,
    exceptInput: true,
    key: 'Enter',
    handle: addItem
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
    handle: () => {
      if ( getMovingIndex() >= 0 ) {
        moveDown()
      } else {
        activateBelow()
      }
    }
  },
  {
    key: 'ArrowUp',
    handle: () => {
      if ( getMovingIndex() >= 0 ) {
        moveUp()
      } else {
        activateAbove()
      }
    }
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

function generateItemIdByIndex(index) {
  return 'item' + index
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
  return d
}

function getActiveIndex() {
  for (let i=0; i<items.value.length; i++) {
    if (items.value[i].active) return i
  }
  return -1
}

function getMovingIndex() {
  for (let i=0; i in items.value; i++) {
    if (items.value[i].moving) return i
  }
  return -1
}

/**
 * データの保存・読み込み
 */
function save() {
  const itemsForSave = items.value.map(item => { return {...item, moving: false} })
  localStorage.setItem(LOCALSTORAGE_KEY_OF_ITEMS, JSON.stringify(itemsForSave))
}
function load() {
  const data = localStorage.getItem(LOCALSTORAGE_KEY_OF_ITEMS)
  if (data) {
    items.value = JSON.parse(data)
  } else {
    save()
  }
  loadCompleted = true
}
function deleteSaveData() {
  localStorage.clear()
}

/**
 * タスクのテンプレートの準備
 */

const itemTemplate = {
  level: 0,
  name: '',
  active: false,
  done: false,
  edit: false,
  moving: false
}

/**
 * 定数
 */
const title = 'todol'

/**
 * リアクティブ
 */
const items = ref([])
const filterStatusMessage = {
  all: '全て',
  filterDoneItems: '未完了のみ'
}
const filterStatus = ref(filterStatusMessage.all)

const filterDoneItems = ref(false)

/**
 * フィルター状態の表示
 */
watch(filterDoneItems, async()=>{
  filterStatus.value = filterDoneItems.value ?
  filterStatusMessage.filterDoneItems : filterStatusMessage.all
})

/**
 * タスクの操作
 */
async function activate(i) {
  if (!loadCompleted) return

  deactivate()

  if (i in items.value) {
    items.value[i].active = true
    // スクロール
    debug('scroll to: '+generateItemIdByIndex(i))
    await document.getElementById(generateItemIdByIndex(i))
    document.getElementById(generateItemIdByIndex(i)).scrollIntoView({  
      behavior: 'smooth',
      block: 'center'
    })
  }
}

function deactivate() {
  if (!loadCompleted) return
  items.value.forEach((e) => {e.active = false})
}

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
        activate(i+delta)
      }
      return
    }
  }
  // activeがない場合、一番上or一番下をアクティブにする
  activate(pioneer)

}

async function focus(position) {
  // DOMが追加されるまで待機
  const targetId = generateItemNameInputTagId(position)
  await document.getElementById(targetId)
  const inputElement = document.getElementById(targetId)
  inputElement.focus()
}
/**
 * 移動
 */

function startMoving() {
  if ( ! loadCompleted ) return
  const ai = getActiveIndex()
  if ( ai < 0 ) return
  const targets = getDescendants(items.value[ai])
  targets.push(ai)
  for (let i=0; i in targets; i++) {
    items.value[targets[i]].edit = false
    items.value[targets[i]].moving = true
  }
}

function finishMoving() {
  items.value.forEach(item => {
    if ( item.moving ) {
      item.moving = false
      return true
    }
  })
  return false
}

function moveUp() {
  moveUpOrDown(true)
}

function moveDown() {
  moveUpOrDown(false)
}

function moveUpOrDown(up) {
  let delta
  let start
  const movingIndex = getMovingIndex()
  const movingItem = items.value[movingIndex]
  const movingItems = items.value.filter(item => item.moving)
  if (up) {
    delta = -1
    start = movingIndex - 1
  } else {
    delta = 1
    start = movingIndex + movingItems.length
  }

  if (start in items.value) {
    if (items.value[start].level == movingItem.level) {
      // 同じレベルのタスクが見つかったら、そのタスクの上/下に移動する
      const insertStart = up ? start : start+1
      if (insertStart > items.value.length) return
      items.value.splice(insertStart, 0, ...movingItems)
      // 元のアイテムは削除する
      let deleteStart = movingIndex
      if (up) deleteStart += movingItems.length
      items.value.splice(deleteStart, movingItems.length)
      return
    } else {
      // 同じレベルでない場合は移動できない
      return
    }
  }
}
/**
 * 追加・削除
 */

async function addItem() {
  let position = 0 // 先頭に追加する
  let level = 0
  const ai = getActiveIndex()
  if (ai in items.value) {
    // アクティブがある場合：子タスクの下に追加する
    const d = getDescendants(items.value[ai])
    position = d.length > 0 ? d.pop() + 1 : ai + 1
    level = items.value[ai].level // 同じレベルに追加する
  }
  let _item = {...itemTemplate}
  _item.edit = true
  _item.level = level
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
 * データを保存すべきかどうか判断するための算出プロパティ
 */
const shouldSave = computed(() => {
  return {
    length: items.value.length,
    level: items.value.map(item => item.level),
    edit: items.value.map(item => item.edit),
    name: items.value.map(item => item.name),
    active: items.value.map(item => item.active),
    done: items.value.map(item => item.done)
  }
}) 

/**
* editが変わった時にactivateする
*/
watch(editChanges,(newValue, oldValue) => {
  // ロード完了前は無視する
  if (!loadCompleted) return
  // 削除の場合はactiveを変更しない。
  if (newValue.length < oldValue.length) return

  for (let i=0; i < newValue.length; i++){
    // 追加かつ最後の要素の場合
    if (oldValue.length < i) {
      // 新しいタスクが編集中ならactivateする
      if (newValue[i] == true) activate(i)
      return
    }

    // 追加でも削除でもない場合
    if (newValue.length == oldValue.length && newValue[i] !=  oldValue[i]) {
      activate(i)
      return
    }
  }
},{ deep: true })

/**
 * saveすべきならsaveする
 */
watch(shouldSave, (n, o) => {
  if (!loadCompleted) return
  debug('should save.')
  save()  
}, {deep: true})

function handleKeyEvent(handlers, event) {
  let kh
  // inputで発生したのかどうか調べる
  const input = event.target.matches('input')
  for (let i=0; i in handlers; i++) {
    kh = handlers[i]
    if (kh.meta && ! event.metaKey) continue
    if (kh.shift && ! event.shiftKey) continue
    if (kh.exceptInput && input) continue
    if (kh.key && kh.key != event.key) continue
    kh.handle()
    event.preventDefault()
    if (kh.continue) continue
    return
  }
}

onMounted(() => {
  /**
  * キー操作の設定
  */
  
  document.body.addEventListener('keydown', function(event) {
    handleKeyEvent(keyDownHandlers, event)
  })

  document.body.addEventListener('keyup', function(event) {
    // moving中にmetaキーを離した状態を検知したらfinishmoving
    if ( !event.metaKey ) finishMoving()
    handleKeyEvent(keyUpHandlers, event)
  })

  /**
   * データのロード
   */
  load()
  
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
          @click="deleteSaveData">
          【デバッグ用】全データを削除
        </button>
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
            :id="generateItemIdByIndex(index)"
            class="list-group-item"
            :class="[{ 'active-item': item.active, 'moving-item': item.moving }, levelClassOf(item)]">
            <div class="input-group active-line">
              <button
                class="btn btn-outline-secondary"
                @click.stop="item.edit=false"
                value="保存"
                >
                <i class="bi-check"></i>
              </button>
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
          </div>
        </div>
      </template>

      <!-- 通常モード -->
      <template v-else>
        <a
          :id="generateItemIdByIndex(index)"
          v-show="!filterDoneItems||!item.done"
          class="list-group-item list-group-item-action"
          :class="[{ 'active-item': item.active, 'moving-item': item.moving }, levelClassOf(item)]"
          @click="activate(index)">
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

.moving-item {
  background-color: orange;
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
