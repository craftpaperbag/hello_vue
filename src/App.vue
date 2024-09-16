<script setup>
import { ref } from 'vue'

const title = ref('todol')

let _items = []
Array.from({length: 5}).forEach((element, i) => {
  _items.push({
    name: 'タスク'+i,
    active: false,
    done: false,
    creation: false
  })
})
_items[1].active = true
_items[2].creation = true

const items = ref(_items)

function handleClickItem(clickedItem) {
  console.log('handleClickItem')
  items.value.forEach((e) => {e.active = false})
  clickedItem.active = true
}
function saveItem(savedItem) {
  console.log('saveItem')
  savedItem.creation = false
}
</script>

<template>
  <div class="container">
    <div class="row">
      <h1>{{ title }}</h1>
    </div>
    <div class="row">
      <div class="col">
        <div class="list-group">
          <template v-for="(item, index) in items" >
            <template v-if="item.creation">
              <div class="list-group-item">
                <div class="input-group">
                  <input
                  type="text"
                  v-model="item.name"
                  class="form-control"
                  >
                  <button
                    class="btn btn-outline-secondary"
                    @click="saveItem(item)"
                    value="保存"
                    >
                    <i class="bi-check"></i>

                  </button>
                </div>
              </div>
            </template>
            <template v-else>
                <a
                  class="list-group-item list-group-item-action"
                  :class="{ 'active-item': item.active }"
                  @click="handleClickItem(item)">
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
                  <button class="btn btn-sm edit-button"
                    @click="item.creation=true"
                    value="編集"
                    v-if="!item.done"
                    >
                    <i class="bi-pencil-fill"></i>
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
.active-item {
  background-color: lightskyblue;
}

.done-item {
  text-decoration: line-through;
}
.edit-button {
  color: white;
}
.edit-button:hover {
  color: darkgrey;
}
</style>
