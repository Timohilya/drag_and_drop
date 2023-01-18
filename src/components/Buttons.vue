<template>
  <section class="buttons">
    <div class="wrapper">
      <div class="col">

        <button v-for="btn in types"
                :key="btn.type"
                :class="`btn ${btn.type}`"
                @dragstart="onDragStart($event, btn.id)"
                draggable="true">
          {{ btn.name }}
        </button>     

        <p>Cols</p>
        <input type="number" min="1" max="4" v-model="table.cols" @change="fillCells()" />   
        <p>Rows</p> 
        <input type="number" min="1" max="4" v-model="table.rows" @change="fillCells()" />  
        
      </div>
      <div ref="container"
           class="col col-container">

        <div v-for="(container, index) in (table.cols * table.rows)"
             :style="{ width: `${100/table.cols}%` }"
             @drop="onDrop($event, index)"
             @dragover.prevent
             @dragenter.prevent
             class="cell">

          <div v-for="(btn, btnIndex) in buttons.list.filter(btn => btn.containerIndex == index)"
               :key="btn.id"
               v-show="!btn.hidden"
               :style="{ width: btn.style.width, 
                         height: btn.style.height, 
                         top: btn.style.top, 
                         left: btn.style.left,
                         zIndex: btn.style.zIndex }"
               @mouseover.self="showCursor($event, btn.id)"
               @mousemove.self="showCursor($event, btn.id)"
               @mousedown.self="btnMouseDown($event, btn.id, 'resize')"
               class="btn-border">

            <button :class="`btn ${btn.type}`"
                    @dblclick="removeButton(btn.id)"
                    @mousedown="btnMouseDown($event, btn.id, 'move')"
                    draggable="true">
              {{ btn.name }} ({{ btn.id }})
            </button> 

          </div> 

        </div>

      </div>
    </div>
  </section>
</template>

<script lang="ts" setup>
import { reactive, ref, onMounted } from 'vue'

  const types: object[] = [{
      id: 0,
      type: 'blue',
      name: 'Blue button'
    },{
      id: 1,
      type: 'orange',
      name: 'Orange button'
    },{
      id: 2,
      type: 'green',
      name: 'Green button'
    },]
  const table: object = reactive({
    cols: 4,
    rows: 4,
    model: [],
    cells: [],
  })
  const buttons: object = reactive({
    list: [],
    lastId: 1,
    lastZindex: 1
  })
  const currentBtn: object = reactive({
    el: {},
    id: null,
    offset: {}
  })

  function onDragStart(e: DragEvent, id: number) {
    e.dataTransfer.dropEffect = 'move'
    e.dataTransfer.effectAllowed = 'move'
    e.dataTransfer.setData('id', id)
  }

  function onDrop(e: DragEvent) {
    const id: string = e.dataTransfer.getData('id')
    if ( !id ) return false
    
    const cellIndex: number = getCellIndex(e)
    if ( cellIndex < 0 ) return false

    buttons.list.push({
      id: generateButtonId(),
      type: types.find(btnType => btnType.id == id).type,
      name: types.find(btnType => btnType.id == id).name,
      width: 1,
      height: 1,
      directions: {
        top: 0,
        right: 0,
        bottom: 0,
        left: 0
      },
      style: {
        width: '100%',
        height: '100%',
        top: '0px',
        left: '0px',
        zIndex: generateButtonZindex(),
      },
      xClickedPosition: null,
      yClickedPosition: null,
      containerIndex: cellIndex,
      hidden: false
    })
    saveData()
  }

  function removeButton(id: number) {
    const {index} = getButton(id)
    buttons.list.splice(index, 1)
    saveData()
  }
  
  function btnMouseDown(e, id: number, task: string){
    e = fixEvent(e) 
    if ( e.which != 1 ) return

    currentBtn.id = id
    currentBtn.el = e
    currentBtn.offset = getMouseOffset(e.target, e)
    // эти обработчики отслеживают процесс и окончание переноса
    if ( task == 'move' ) {
      document.onmousemove = btnMouseMove
      document.onmouseup = btnMouseUp
    }
    if ( task == 'resize' ) {
      changeBtnContainerForResize()
      document.onmousemove = btnMouseResizeMove
      document.onmouseup = removeDocumentEventHandlers
    }
    // отменить перенос и выделение текста при клике на тексте
    document.ondragstart = function() { return false }
    document.body.onselectstart = function() { return false }
  }

  function btnMouseMove(e){ 
    e = fixEvent(e)
    const container = document.querySelector('.col-container')
    const containerPos: object = getPosition(container)
    const containerWidth: number = parseInt(container.offsetWidth)
    const containerHeight: number = parseInt(container.offsetHeight)

    if(
    (e.pageX - currentBtn.offset.left > containerPos.x)                       &&
    (e.pageX + currentBtn.offset.right < (containerPos.x + containerWidth))   &&
    (e.pageY - currentBtn.offset.top > containerPos.y)                        &&
    (e.pageY + currentBtn.offset.bottom < (containerPos.y + containerHeight)) ){
      currentBtn.el.target.style.width = currentBtn.el.target.offsetWidth + 'px'
      currentBtn.el.target.style.height = currentBtn.el.target.offsetHeight + 'px'
      currentBtn.el.target.style.position = 'fixed'
      currentBtn.el.target.style.zIndex = '99999'
      currentBtn.el.target.style.boxShadow = '2px 2px 2px 0px #00000080'
      currentBtn.el.path[1].style.zIndex = buttons.lastZindex + 1
      currentBtn.el.target.style.top = e.pageY - currentBtn.offset.top + 'px'
      currentBtn.el.target.style.left = e.pageX - currentBtn.offset.left + 'px'
    }
  }

  function btnMouseUp(e){ 
    const cellIndex: number = getCellIndex(e)
    
    if ( cellIndex >= 0 ) {
      changeBtnViewByClickedPosition()
      moveButton(currentBtn.id, cellIndex)
      saveData()
    } else {
      setButtonDefaultStyles()
    }

    removeDocumentEventHandlers()
  }

  function showCursor(e, id: number){
    e = fixEvent(e) 
    
    const {index, btn} = getButton(id)
    const offset = getMouseOffset(e.target, e)
    const {xClickedPosition, yClickedPosition} = getClickedPosition(offset, btn)
    
    if ( xClickedPosition == 1 && yClickedPosition == btn.width ||
                xClickedPosition == btn.height && yClickedPosition == 1 ) {
      e.target.style.cursor = 'nesw-resize'
    } else if ( xClickedPosition == 1 && yClickedPosition == 1 ||
                xClickedPosition == btn.height && yClickedPosition == btn.width ) {
      e.target.style.cursor = 'nw-resize'
    } else if ( xClickedPosition == 1 || xClickedPosition == btn.height ) {
      e.target.style.cursor = 'ns-resize'
    } else if ( yClickedPosition == 1 || yClickedPosition == btn.width ) {
      e.target.style.cursor = 'ew-resize'
    }
  }
  function btnMouseResizeMove(e){ 
    e = fixEvent(e)
    for ( let i = 0; i < table.cells.length; i++ ) {
      const targ = table.cells[i]
      const targPos: object = getPosition(targ)
      const targWidth: number = parseInt(targ.offsetWidth)
      const targHeight: number = parseInt(targ.offsetHeight)

      if(
      (e.pageX > targPos.x)                &&
      (e.pageX < (targPos.x + targWidth))  &&
      (e.pageY > targPos.y)                &&
      (e.pageY < (targPos.y + targHeight)) ){
        const {index, btn, containerIndex} = getButton(currentBtn.id)

        let top = 0
        let bottom = 0
        let left = 0
        let right = 0

        if ( btn.xClickedPosition == 1 || btn.xClickedPosition == btn.height ) {
          const hDif: number = (table.model.findIndex(arr => arr.includes(containerIndex)) - top) - table.model.findIndex(arr => arr.includes(i))
          hDif > 0 ? top = hDif : bottom = Math.abs(hDif)
          btn.xClickedPosition = top + bottom + 1
        } else {
          top = btn.directions.top
          bottom = btn.directions.bottom
        }
        if ( btn.yClickedPosition == 1 || btn.yClickedPosition == btn.width ) {
          const wDif: number = table.model[table.model.findIndex(arr => arr.includes(containerIndex))].findIndex(x => x == containerIndex) -
                      table.model[table.model.findIndex(arr => arr.includes(i))].findIndex(x => x == i)
          wDif > 0 ? left = wDif : right = Math.abs(wDif)
          btn.yClickedPosition = left + right + 1
        } else {
          left = btn.directions.left
          right = btn.directions.right
        }

        const height: number = top + bottom + 1
        const width: number = left + right + 1

        setButtonStyles(currentBtn.id, width, height, top, right, bottom, left)
        saveData()
      }
    }
  }




  function generateButtonId() { return buttons.lastId++ }
  function generateButtonZindex() { return buttons.lastZindex++ }
  function getButton(id: number) { 
    const index: number = buttons.list.findIndex((btn) => btn.id == id)
    const btn: object = buttons.list[index]
    const containerIndex: number = buttons.list[index].containerIndex
    return { index, btn, containerIndex }
  }
  // сохранить данные в localStorage
  function saveData() {
    localStorage.removeItem('data')
    localStorage.setItem('data', JSON.stringify({
      buttons: buttons
    }))
  }
  function getCellIndex(e) {
    let index: number = -1
    e = fixEvent(e)
    for( let i = 0; i < table.cells.length; i++ ){
      const targ = table.cells[i]
      const targPos: object = getPosition(targ)
      const targWidth: number = parseInt(targ.offsetWidth)
      const targHeight: number= parseInt(targ.offsetHeight)

      if(
      (e.pageX > targPos.x)                &&
      (e.pageX < (targPos.x + targWidth))  &&
      (e.pageY > targPos.y)                &&
      (e.pageY < (targPos.y + targHeight)) ){
        index = i
      }
    }
    return index
  }
  // изменить центр кнопки, относительно кликнутого места
  function changeBtnViewByClickedPosition() {
    const {index, btn} = getButton(currentBtn.id)
    const {xClickedPosition, yClickedPosition} = getClickedPosition(currentBtn.offset, btn)
    const topCurrentPosition: number = btn.height - btn.directions.bottom
    const topDif: number = xClickedPosition - topCurrentPosition
    const leftCurrentPosition: number = btn.width - btn.directions.right
    const leftDif: number = yClickedPosition - leftCurrentPosition

    if ( xClickedPosition > topCurrentPosition ) {
      setButtonStyles(currentBtn.id, btn.width, btn.height, btn.directions.top + topDif, btn.directions.right, btn.directions.bottom - topDif, btn.directions.left)
    } else {
      setButtonStyles(currentBtn.id, btn.width, btn.height, btn.directions.top - Math.abs(topDif), btn.directions.right, btn.directions.bottom + Math.abs(topDif), btn.directions.left)
    }

    if ( yClickedPosition > leftCurrentPosition ) {
      setButtonStyles(currentBtn.id, btn.width, btn.height, btn.directions.top, btn.directions.right - leftDif, btn.directions.bottom, btn.directions.left + leftDif)
    } else {
      setButtonStyles(currentBtn.id, btn.width, btn.height, btn.directions.top, btn.directions.right + Math.abs(leftDif), btn.directions.bottom, btn.directions.left - Math.abs(leftDif))
    }
  }
  // изменить центр кнопки, относительно кликнутого места при resize
  function changeBtnContainerForResize() {
    const {index, btn} = getButton(currentBtn.id)
    const {xClickedPosition, yClickedPosition} = getClickedPosition(currentBtn.offset, btn)
    
    if ( btn.height > 1 ) {
      if ( xClickedPosition == 1 ) {
        btn.containerIndex += table.cols * btn.directions.bottom
        btn.directions.top += btn.directions.bottom
        btn.directions.bottom = 0
      }
      if ( xClickedPosition == btn.height ) {
        btn.containerIndex -= table.cols * btn.directions.top
        btn.directions.bottom += btn.directions.top
        btn.directions.top = 0
      }
    }
    if ( btn.width > 1 ) {
      if ( yClickedPosition == 1 ) {
        btn.containerIndex += btn.directions.right
        btn.directions.left += btn.directions.right
        btn.directions.right = 0
      }
      if ( yClickedPosition == btn.width ) {
        btn.containerIndex -= btn.directions.left
        btn.directions.right += btn.directions.left
        btn.directions.left = 0
      }
    }
    btn.xClickedPosition = xClickedPosition
    btn.yClickedPosition = yClickedPosition
    setButtonStyles(currentBtn.id, btn.width, btn.height, btn.directions.top, btn.directions.right, btn.directions.bottom, btn.directions.left)
  }
  function setButtonStyles(id: number, width: number, height: number, top: number, right: number, bottom: number, left: number) {
    const {index, btn} = getButton(id)
    const topVal: number = height - 1 - bottom
    const leftVal: number = width - 1 - right
    btn.width = width
    btn.height = height
    btn.directions.top = top
    btn.directions.right = right
    btn.directions.bottom = bottom
    btn.directions.left = left
    btn.style.width = `calc(${width}00% - 2px)`
    btn.style.height = `calc(${height}00% - 2px)`
    btn.style.top = `-${topVal}00%`
    btn.style.left = `-${leftVal}00%`
  }
  function moveButton(id: number, containerIndex: number) {
    const {index, btn} = getButton(id)
    if ( btn.containerIndex == containerIndex ) {
      setButtonDefaultStyles()
      setButtonStyles(currentBtn.id, btn.width, btn.height, btn.directions.top, btn.directions.right, btn.directions.bottom, btn.directions.left)
    }
    btn.containerIndex = containerIndex
    btn.style.zIndex = generateButtonZindex()
    checkButtonOnDirections(currentBtn.id)
  }
  // изменить положение кнопки, если они выходит за границы
  function checkButtonOnDirections(id: number) {
    const {index, btn, containerIndex} = getButton(id)
    const directions: object = { top: 0, bottom: 0, left: 0, right: 0 }
    const commonCells: object = { top: 0, bottom: 0, left: 0, right: 0 }

    commonCells.top = table.model.findIndex(arr => arr.includes(containerIndex))
    commonCells.bottom = table.model.length - commonCells.top - 1
    commonCells.left = table.model[table.model.findIndex(arr => arr.includes(containerIndex))].findIndex(x => x == containerIndex)
    commonCells.right = table.model[table.model.findIndex(arr => arr.includes(containerIndex))].length - commonCells.left - 1

    if ( btn.directions.top > commonCells.top ) directions.top = btn.directions.top - commonCells.top
    if ( btn.directions.bottom > commonCells.bottom ) directions.bottom = btn.directions.bottom - commonCells.bottom
    if ( btn.directions.left > commonCells.left ) directions.left = btn.directions.left - commonCells.left
    if ( btn.directions.right > commonCells.right ) directions.right = btn.directions.right - commonCells.right

    if ( directions.top > 0 ) setButtonStyles(id, btn.width, btn.height, btn.directions.top -= directions.top, btn.directions.right, btn.directions.bottom += directions.top, btn.directions.left)
    if ( directions.bottom > 0 ) setButtonStyles(id, btn.width, btn.height, btn.directions.top += directions.bottom, btn.directions.right, btn.directions.bottom -= directions.bottom, btn.directions.left)
    if ( directions.left > 0 ) setButtonStyles(id, btn.width, btn.height, btn.directions.top, btn.directions.right += directions.left, btn.directions.bottom, btn.directions.left -= directions.left)
    if ( directions.right > 0 ) setButtonStyles(id, btn.width, btn.height, btn.directions.top, btn.directions.right -= directions.right, btn.directions.bottom, btn.directions.left += directions.right)
  }
  function checkButtonsOnChangeTable() {
    buttons.list.forEach(btn => {
      if ( btn.containerIndex < (table.cols * table.rows) ) checkButtonOnDirections(btn.id)
      btn.hidden = btn.width > table.cols || btn.height > table.rows
    })
  }
  function setButtonDefaultStyles() {
    currentBtn.el.target.style.width = '100%'
    currentBtn.el.target.style.height = '100%'
    currentBtn.el.target.style.position = 'relative'
    currentBtn.el.target.style.zIndex = '1'
    currentBtn.el.target.style.boxShadow = 'none'
    currentBtn.el.target.style.top = '0'
    currentBtn.el.target.style.left = '0'
  }
  function removeDocumentEventHandlers() {
    document.onmousemove = null
    document.onmouseup = null
    document.ondragstart = null
    document.body.onselectstart = null
  }
  // получить сдвиг target относительно курсора мыши
  function getMouseOffset(target, e) {
    const docPos: object = getPosition(target)
    const btnWidth: number = parseInt(e.target.offsetWidth)
    const btnHeight: number = parseInt(e.target.offsetHeight)
    return {
      width: btnWidth,
      height: btnHeight,
      top: e.pageY - docPos.y,
      right: btnWidth - (e.pageX - docPos.x), 
      bottom: btnHeight - (e.pageY - docPos.y), 
      left: e.pageX - docPos.x
    }
  }
  function getPosition(e){
    let left: number = 0
    let top: number = 0

    while (e.offsetParent){
      left += e.offsetLeft
      top += e.offsetTop
      e = e.offsetParent
    }

    left += e.offsetLeft
    top += e.offsetTop

    return {x:left, y:top}
  }
  function getClickedPosition(offset: object, btn: object){
    const xClickedPosition: number = parseInt((offset.top - 0.05) / (offset.height / btn.height)) + 1
    const yClickedPosition: number = parseInt((offset.left - 0.05) / (offset.width / btn.width)) + 1

    return { xClickedPosition, yClickedPosition }
  }
  function fixEvent(e) {
    // получить объект событие для IE
    e = e || window.event
    // добавить pageX/pageY для IE
    if ( e.pageX == null && e.clientX != null ) {
      const html = document.documentElement
      const body = document.body
      e.pageX = e.clientX + (html && html.scrollLeft || body && body.scrollLeft || 0) - (html.clientLeft || 0)
      e.pageY = e.clientY + (html && html.scrollTop || body && body.scrollTop || 0) - (html.clientTop || 0)
    }
    // добавить which для IE
    if (!e.which && e.button) {
      e.which = e.button & 1 ? 1 : ( e.button & 2 ? 3 : ( e.button & 4 ? 2 : 0 ) )
    }

    return e
  }

  
  onMounted(() => {
    loadDataFromStorage()
    fillCells()
  })
  function loadDataFromStorage() {
    if ( !localStorage.getItem('data') ) return false

    const data: object = JSON.parse(localStorage.getItem('data'))
    if ( !data.buttons.list.length ) return false
    
    buttons.list = data.buttons.list
    buttons.lastId = data.buttons.lastId
    buttons.lastZindex = data.buttons.lastZindex
  }
  const container = ref(null)
  function fillCells() {
    table.model = createTableModel()
    table.cells = container.value.querySelectorAll('.cell')
    checkButtonsOnChangeTable()
  }
  function createTableModel() {
    const arr: number[] = [];
    let ij: number = 0
    for ( let i = 0; i < table.rows; i++ ) {
      arr[i] = []
      for ( let j = 0; j < table.cols; j++ ) {
        arr[i].push(ij++)
      }
    }
    return arr
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
        height: 50px
        box-shadow: inset 0px 0px 0px 1px #000
        position: relative
      .btn-border
        width: 100%
        height: 100%
        margin: 1px
        border: 4px solid #707070
        box-shadow: 2px 2px 2px 1px #00000080
        cursor: nesw-resize
        position: absolute
        z-index: 1
      .btn
        width: 100%
        height: 100%
  .btn
    height: 50px
    border: none
    &:not(:last-child)
      margin: 0 0 10px 0
    &.blue
      background-color: #4d6bff
    &.orange
      background-color: #ffa34d
    &.green
      background-color: #65c84f
</style>