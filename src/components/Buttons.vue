<template>
  <section class="buttons">
    <div class="wrapper">
      <div class="col">
        <button v-for="btn in btnTypes"
                :key="btn.type"
                :class="`btn ${btn.type}`"
                @dragstart="onDragStart($event, btn.id, 'create')"
                draggable="true">
          {{ btn.name }}
        </button>     
        <p>Cols</p>
        <input type="number" min="1" max="4" v-model="cols" />   
        <p>Rows</p> 
        <input type="number" min="1" max="4" v-model="rows" />   
      </div>
      <div ref="container" class="col">
        <div v-for="(container, index) in (cols * rows)"
             :style="{ width: 100/cols + '%' }"
             class="cell"
             @drop="onDrop($event, index)"
             @dragover.prevent
             @dragenter.prevent>
          <template>
          </template>
          <div v-for="(btn, btnIndex) in buttons.filter(btn => btn.containerIndex == index)"
               v-show="btnIndex == buttons.filter(btn => btn.containerIndex == index).length-1"
               :key="btn.id"
               class="btn-border">
            <button :class="`btn ${btn.type}`"
                    @dblclick="removeButtonWithId(btn.id)"
                    @dragstart="onDragStart($event, btn.id, 'move')"
                    draggable="true">
              {{ btn.name }} ({{ 1 + btn.id }})</button> 
          </div> 
        </div>
      </div>
    </div>
  </section>
</template>

<script lang="ts">
import { ref } from 'vue'
 
export default {
  data() {
    return {
      
    }
  },

  setup() {
    const btnTypes: object[] = [
      {
        id: 0,
        type: 'blue',
        name: 'Blue button',
      },
      {
        id: 1,
        type: 'orange',
        name: 'Orange button',
      },
      {
        id: 2,
        type: 'green',
        name: 'Green button',
      },
    ]
    const cols: number = ref(4)
    const rows: number = ref(4)
    const buttons: object[] = ref([])
    let buttonsId: number = ref(0)
    function generateButtonId() { return buttonsId.value++ }

    

    function onDragStart(e: DragEvent, btnId: number, task: string) {
      e.dataTransfer.dropEffect = 'move'
      e.dataTransfer.effectAllowed = 'move'
      e.dataTransfer.setData('btnId', btnId)
      e.dataTransfer.setData('task', task)
    }
    function onDrop(e: DragEvent, containerIndex) {
      const btnId = e.dataTransfer.getData('btnId')
      const task = e.dataTransfer.getData('task')
      if ( task == 'create' ) {
        buttons.value.push({
          id: generateButtonId(),
          type: btnTypes.find(btnType => btnType.id == btnId).type,
          name: btnTypes.find(btnType => btnType.id == btnId).name,
          containerIndex
        })
      }
      if ( task == 'move' ) {
        const btnIndex = buttons.value.findIndex((btn) => btn.id == btnId)
        const btn = buttons.value[btnIndex]
        btn.containerIndex = containerIndex
        buttons.value.splice(btnIndex, 1)
        buttons.value.push(btn)
      }
      saveData()
    }
    function removeButtonWithId(id) {
      const index = buttons.value.findIndex((btn) => btn.id === id)
      buttons.value.splice(index, 1)
      saveData()
    }
    function saveData() {
      localStorage.removeItem('data')
      localStorage.setItem('data', JSON.stringify({
        buttons: buttons.value,
        buttonsId: buttonsId.value
      }))
    }

    return {
      btnTypes,
      cols,
      rows,
      buttonsId,
      buttons,
      onDragStart,
      onDrop
    }
  },

  mounted() {
    if ( localStorage.getItem('data') ) {
      const data = JSON.parse(localStorage.getItem('data'))
      if ( data.buttons.length ) {
        this.buttons = data.buttons
        this.buttonsId = data.buttonsId
      }
    }
  },
}
</script>

<style scoped lang="sass">
.buttons 
  margin: 40px 0
  .wrapper
    display: flex
    column-gap: 30px
  .col
    &:nth-child(1)
      width: 200px
      display: flex
      flex-direction: column
    &:nth-child(2)
      width: calc(100% - 230px)
      height: max-content
      display: flex
      flex-wrap: wrap
      .cell
        height: 30px
        border: 1px solid black
      .btn-border
        width: 100%
        height: 100%
        border: 2px solid black
        cursor: nesw-resize
      .btn
        width: 100%
        height: 100%
  .btn
    height: 30px
    border: none
    &:not(:last-child)
      margin: 0 0 10px 0
    &.blue
      background-color: blue
    &.orange
      background-color: orange
    &.green
      background-color: green
</style>