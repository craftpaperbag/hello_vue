<script setup>
import { ref } from 'vue'

const title = ref('todol')

let _items = []
Array.from({length: 5}).forEach((element, i) => {
  _items.push({name: 'タスク'+i, active: false, done: false})
})
_items[1].active = true

const items = ref(_items)

function handleClickItem(clickedItem) {
  console.log('handleClickItem')
  items.value.forEach((e) => {e.active = false})
  clickedItem.active = true
}
function handleChangeItemStatus(changedItem, event) {
  console.log('handleChangeItemStatus')
  changedItem.done = event.target.checked
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
            <a
              class="list-group-item list-group-item-action"
              :class="{ active: item.active, 'area-current': item.active}"
              @click="handleClickItem(item)">
              <input
                class="form-check-input me-1"
                type="checkbox"
                :id="'checkbox'+index"
                @click.stop
                v-model="item.done"  
              >
              <label
                class="form-check-label"
                :class="{'done-item': item.done}"
                :for="'checkbox'+index"
                @click.stop
              >
                {{ item.name }}
              </label>
            </a>
          </template>
        </div>
      </div>
    </div>
  </div>
</template>



<style scoped>
  .done-item {
    text-decoration: line-through;
  }
</style>
