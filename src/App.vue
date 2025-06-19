<template lang="pug">
  h1 LA/DX Menu Router
  .buttons
    button(@click="showAllInv=!showAllInv") {{showAllInv ? 'Hide Equipped' : 'Show Equipped'}}
    button(@click="editingEnabled=!editingEnabled") {{editingEnabled ? 'Disable Editing' : 'Enable Editing'}}
    button(v-if="editingEnabled" @click="addOne") Add an Action
    button(v-if="editingEnabled" @click="mappingToClipboard") Export Mapping to Clipboard
  br
  div(v-if="!actions.length")
    p This is a tool for quick reference and planning of menu routes for Link's Awakening and Link's Awakening DX speedruns.
    p To get started, click "Add an Action".
    p Once you have built a menu route, click "Export Mapping to Clipboard" to get your mapping as JSON. Host this data somewhere (such as a github gist) and pass it to this site as a query parameter to view it, like this:
    pre {{'<this site>?json=<your json url>'}}
    br
  .action(v-for="(action, i) of actions" @click="select(i)")
    span {{labelFor(action)}}
    span(v-if="action.buttons") {{emojisFor(action.buttons)}}
    .inv-grid(v-if="showAllInv && action.buttons")
      .inv-item(v-for="(item, j) of states[i].inv.slice(0,2)" :class="{selected:states[i].position==j}")
        span(v-if="j==0" style="margin-right:4px") B
        span(v-if="j==1" style="margin-right:4px") A
        span(v-if="j>1" style="margin-right:6px")
        div(:style="spriteFor(item)")
  span(v-if="actions.length") Input count: {{inputCount}}
  .buttons
    button(@click="showAllInv=!showAllInv") {{showAllInv ? 'Hide Equipped' : 'Show Equipped'}}
    button(@click="editingEnabled=!editingEnabled") {{editingEnabled ? 'Disable Editing' : 'Enable Editing'}}
    button(v-if="editingEnabled" @click="addOne") Add an Action
    button(v-if="editingEnabled" @click="mappingToClipboard") Export Mapping to Clipboard
  .modal-overlay(v-if="showModal" @click.self="showModal = false")
    .modal.action
      input(v-if="editingEnabled" v-model="actions[selected].label" :placeholder="labelFor(actions[selected])")
      span(v-else) {{labelFor(actions[selected])}}
      span(v-if="actions[selected].buttons") {{emojisFor(actions[selected].buttons)}}
      .inv-and-buttons
        .inv-grid
          .inv-item(v-for="(item, j) of states[selected].inv.slice(0,12)" :class="{selected:states[selected].position==j, 'top-of-inv':j<=1}")
            span(v-if="j==0" style="margin-right:2px") B
            span(v-if="j==1" style="margin-right:2px") A
            span(v-if="j>1" style="margin-right:6px")
            div(:style="spriteFor(item)")
        .input-btns(v-if="editingEnabled")
          .buttons
            button(@click="actions[selected].buttons = (actions[selected].buttons ?? '') + 'u';actions[selected].sq=false;actions[selected].get=''" style="margin-left: 18px") ⬆️
          .buttons
            button(@click="actions[selected].buttons = (actions[selected].buttons ?? '') + 'l';actions[selected].sq=false;actions[selected].get=''") ⬅️
            button(@click="actions[selected].buttons = (actions[selected].buttons ?? '') + 'r';actions[selected].sq=false;actions[selected].get=''") ➡️
            button(@click="actions[selected].buttons = (actions[selected].buttons ?? '') + 'b';actions[selected].sq=false;actions[selected].get=''") 🅱️
            button(@click="actions[selected].buttons = (actions[selected].buttons ?? '') + 'a';actions[selected].sq=false;actions[selected].get=''") 🅰️
          .buttons
            button(@click="actions[selected].buttons = (actions[selected].buttons ?? '') + 'd';actions[selected].sq=false;actions[selected].get=''" style="margin-left: 18px") ⬇️
            button(@click="actions[selected].buttons = ''" style="margin-left: 38px") Clear
      div(v-if="editingEnabled")
        label(for="sq-toggle") Save &amp; Quit
        input#sq-toggle(type="checkbox" v-model="actions[selected].sq" @click="actions[selected].buttons='';actions[selected].get=''")
      div(v-if="editingEnabled" style="padding-bottom: 16px")
        label(for="get-select") Get item:&nbsp;
        select#get-select(@change="actions[selected].buttons='';actions[selected].sq=false" v-model="actions[selected].get")
          option(value="") {{'<none>'}}
          option(v-for="item of sprites" :value="item") {{item}}
      .buttons
        button(v-if="editingEnabled" @click="addOne") Add One After
        button(v-if="editingEnabled" @click="deleteOne") Delete
        button(@click="showModal = false") Close
</template>

<script setup>
  import { ref, computed, onMounted } from 'vue'
  const actions = ref([])

  const states = computed(() => {
    let states = [{ position:2, inv:[...new Array(12), 'boomerang'] }]
    actions.value.forEach((action, index) => {
      let state = JSON.parse(JSON.stringify(states[index]))
      if (action.sq) {
        state.position = 2
      } else if (action.get && !state.inv.includes(action.get)){
        const i = state.inv.findIndex(it => !it)
        if (i >= 0)
          state.inv[i] = action.get
      } else if (action.buttons) {
        action.buttons.split('').forEach(button => {
          if (button === "u")
            state.position -= 2
          else if (button === "l")
            state.position -= 1
          else if (button === "r")
            state.position += 1
          else if (button === "d")
            state.position += 2
          else if (button === "b")
            [state.inv[state.position], state.inv[0]] = [state.inv[0], state.inv[state.position]]
          else if (button === "a")
            [state.inv[state.position], state.inv[1]] = [state.inv[1], state.inv[state.position]]

          if (state.position < 2)
            state.position = 11
          else if (state.position > 11)
            state.position = 2
        })
      }
      states.push(state)
    })
    states.shift()
    return states
  })

  const inputCount = computed(() => actions.value.reduce((acc, action) => {
    if (action.buttons)
      acc += action.buttons.length
    return acc
  }, 0))

  const selected = ref(null)
  const showModal = ref(false)
  const select = i => {
    selected.value = i
    showModal.value = true
  }

  const showAllInv = ref(true)
  const editingEnabled = ref(false)
  const addOne = () => {
    let newIndex = showModal.value ? selected.value + 1 : actions.value.length
    actions.value.splice(newIndex, 0, {})
    selected.value = newIndex
    showModal.value = true
  }
  const deleteOne = () => {
    actions.value.splice(selected.value, 1)
    selected.value = null
    showModal.value = false
  }
  const mappingToClipboard = async () => {
    actions.value = actions.value.map(action => Object.fromEntries(Object.entries(action).filter(([_,v]) => v)))
    await navigator.clipboard.writeText(JSON.stringify(actions.value,null,2))
  }

  const labelFor = action => {
    if (action.label)
      return action.label
    if (action.get)
      return `Get ${action.get}`
    if (action.sq)
      return "Save and quit"
    return "<empty>"
  }

  const emojisFor = string => {
    const map = {
      r: '➡️', // right arrow
      l: '⬅️', // left arrow
      u: '⬆️', // up arrow
      d: '⬇️', // down arrow
      b: '🅱️', // B button
      a: '🅰️', // A button
    }

    return [...string.toLowerCase()]
      .map(ch => map[ch] || ch)
      .join('')
  }

  const sprites = [
    'sword',
    'shield',
    'bracelet',
    'powder',
    'bow',
    'bomb',
    'ocarina',
    'toadstool',
    'feather',
    'boots',
    'hookshot',
    'rod',
    'shovel',
    'boomerang'
  ]
  const spriteFor = string => {
    let index = sprites.indexOf(string) + 1
    return {
      width: "8px",
      height: "16px",
      backgroundImage: 'url(/la-equipment.png)',
      backgroundRepeat: 'no-repeat',
      backgroundSize: 'auto',
      backgroundPosition: `-${index * 8}px 0`,
      "image-rendering": "pixelated"
    }
  }

  onMounted(async () => {
    const params = new URLSearchParams(window.location.search)

    let json = params.get('json')
    if (json)
      await fetch(json)
        .then(res => res.json())
        .then(data => {actions.value = data})

    if (params.get('edit') || !actions.value.length)
      editingEnabled.value = true
  })
</script>

<style lang="sass">
  body
    font-family: sans-serif
    color: white
    background-color: black
  .action
    margin-bottom: 1rem
    display: flex
    flex-direction: column
  .inv-grid
    display: grid
    grid-template-columns: repeat(2, auto)
    width: 50px
    background-color: white
    color: black
    margin-top: 1rem
  .inv-item
    display: flex
    font-size: 8px
  .top-of-inv
    border-bottom: 2px dotted black
    padding-bottom: 2px
    margin-bottom: 2px
  .selected
    background-color: #BBB
  .modal-overlay
    position: fixed
    inset: 0
    background: rgba(0, 0, 0, 0.6)
    display: flex
    align-items: center
    justify-content: center
    z-index: 1000
  .modal
    background: white
    color: black
    padding: 1.5rem
    border-radius: 8px
    box-shadow: 0 0 10px #0008
  .buttons
    display: flex
    justify-content: flex-start
    button
      flex-grow: 0
      margin: 2px
  .inv-and-buttons
    display: flex
    button
      flex-grow: 0
  .input-btns
    margin-top: 16px
    margin-left: 16px
</style>
