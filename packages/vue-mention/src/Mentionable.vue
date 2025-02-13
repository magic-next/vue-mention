<script>
import getCaretPosition from 'textarea-caret'
import { VPopover } from 'v-tooltip'

const userAgent = typeof window !== 'undefined' ? window.navigator.userAgent : ''
const isIe = userAgent.indexOf('MSIE ') !== -1 || userAgent.indexOf('Trident/') !== -1

export default {
  components: {
    VPopover,
  },

  inheritAttrs: false,

  props: {
    keys: {
      type: Array,
      required: true,
    },

    placement: {
      type: String,
      default: 'top-start',
    },

    items: {
      type: Array,
      default: () => [],
    },

    omitKey: {
      type: Boolean,
      default: false,
    },

    enableSpace: {
      type: Boolean,
      default: false,
    },

    filteringDisabled: {
      type: Boolean,
      default: false,
    },

    insertSpace: {
      type: Boolean,
      default: false,
    },

    mapInsert: {
      type: Function,
      default: null,
    },

    limit: {
      type: Number,
      default: 8,
    },
  },

  data () {
    return {
      key: null,
      oldKey: null,
      searchText: null,
      caretPosition: null,
      selectedIndex: 0,
    }
  },

  computed: {
    filteredItems () {
      if (!this.searchText || this.filteringDisabled) {
        return this.items
      }

      const searchText = this.searchText.toLowerCase()

      return this.items.filter(item => {
        /** @type {string} */
        let text
        if (item.searchText) {
          text = item.searchText
        } else if (item.label) {
          text = item.label
        } else {
          text = ''
          for (const key in item) {
            text += item[key]
          }
        }
        return text.toLowerCase().includes(searchText)
      })
    },

    displayedItems () {
      return this.filteredItems.slice(0, this.limit)
    },
  },

  watch: {
    displayedItems () {
      this.selectedIndex = 0
    },

    searchText (value, oldValue) {
      if (value) {
        this.$emit('search', value, oldValue)
      }
    },
  },

  mounted () {
    this.input = this.getInput()
    this.attach()
  },

  updated () {
    const input = this.getInput()
    if (input !== this.input) {
      this.detach()
      this.input = input
      this.attach()
    }
  },

  beforeDestroy () {
    this.detach()
  },

  methods: {
    getInput () {
      const [vnode] = this.$scopedSlots.default()
      if (vnode) {
        if (vnode.elm.tagName === 'INPUT' || vnode.elm.tagName === 'TEXTAREA' || vnode.elm.isContentEditable) {
          return vnode.elm
        } else {
          return vnode.elm.querySelector('input') || vnode.elm.querySelector('textarea') || vnode.elm.querySelector('[contenteditable="true"]')
        }
      }
      return null
    },

    attach () {
      if (this.input) {
        this.input.addEventListener('input', this.onInput)
        this.input.addEventListener('keydown', this.onKeyDown)
        this.input.addEventListener('keyup', this.onKeyUp)
        this.input.addEventListener('scroll', this.onScroll)
        this.input.addEventListener('blur', this.onBlur)
      }
    },

    detach () {
      if (this.input) {
        this.input.removeEventListener('input', this.onInput)
        this.input.removeEventListener('keydown', this.onKeyDown)
        this.input.removeEventListener('keyup', this.onKeyUp)
        this.input.removeEventListener('scroll', this.onScroll)
        this.input.removeEventListener('blur', this.onBlur)
      }
    },

    onInput () {
      this.checkKey()
    },

    onBlur () {
      this.closeMenu()
    },

    onKeyDown (e) {
      if (this.key) {
        if (e.key === 'ArrowDown' || e.keyCode === 40) {
          this.selectedIndex++
          if (this.selectedIndex >= this.displayedItems.length) {
            this.selectedIndex = 0
          }
          this.cancelEvent(e)
        }
        if (e.key === 'ArrowUp' || e.keyCode === 38) {
          this.selectedIndex--
          if (this.selectedIndex < 0) {
            this.selectedIndex = this.displayedItems.length - 1
          }
          this.cancelEvent(e)
        }
        if ((e.key === 'Enter' || e.key === 'Tab' || e.keyCode === 13 || e.keyCode === 9) &&
          this.displayedItems.length > 0) {
          this.applyMention(this.selectedIndex)
          this.cancelEvent(e)
        }
        if (e.key === 'Escape' || e.keyCode === 27) {
          this.closeMenu()
          this.cancelEvent(e)
        }
      }
    },

    onKeyUp (e) {
      if (this.cancelKeyUp && (e.key === this.cancelKeyUp || e.keyCode === this.cancelKeyCode)) {
        this.cancelEvent(e)
      }
      this.cancelKeyUp = null
      // IE
      this.cancelKeyCode = null
    },

    cancelEvent (e) {
      e.preventDefault()
      e.stopPropagation()
      this.cancelKeyUp = e.key
      // IE
      this.cancelKeyCode = e.keyCode
    },

    onScroll () {
      this.updateCaretPosition()
    },

    getSelectionStart () {
      return this.input.isContentEditable ? window.getSelection().anchorOffset : this.input.selectionStart
    },

    setCaretPosition (index) {
      this.$nextTick(() => {
        this.input.selectionEnd = index
      })
    },

    getValue () {
      return this.input.isContentEditable ? window.getSelection().anchorNode.textContent : this.input.value
    },

    setValue (value) {
      this.input.value = value
      this.emitInputEvent('input')
    },

    emitInputEvent (type) {
      let event
      if (isIe) {
        event = document.createEvent('Event')
        event.initEvent(type, true, true)
      } else {
        event = new Event(type)
      }
      this.input.dispatchEvent(event)
    },

    checkKey () {
      const index = this.getSelectionStart()
      if (index >= 0) {
        const { key, keyIndex } = this.getLastKeyBeforeCaret(index)
        const searchText = this.lastSearchText = this.getLastSearchText(index, keyIndex)
        if (!(keyIndex < 1 || /\s/.test(this.getValue()[keyIndex - 1]))) {
          return false
        }
        if (searchText != null) {
          this.openMenu(key, keyIndex)
          this.searchText = searchText
          return true
        }
      }
      this.closeMenu()
      return false
    },

    getLastKeyBeforeCaret (caretIndex) {
      const [keyData] = this.keys.map(key => ({
        key,
        keyIndex: this.getValue().lastIndexOf(key, caretIndex - 1),
      })).sort((a, b) => b.keyIndex - a.keyIndex)
      return keyData
    },

    getLastSearchText (caretIndex, keyIndex) {
      if (keyIndex !== -1) {
        const searchText = this.getValue().substring(keyIndex + 1, caretIndex)
        if (this.enableSpace || !/\s/.test(searchText)) {
          return searchText
        }
      }
      return null
    },

    openMenu (key, keyIndex) {
      if (this.key !== key) {
        this.key = key
        this.keyIndex = keyIndex
        this.updateCaretPosition()
        this.selectedIndex = 0
        this.$emit('open', key)
      }
    },

    closeMenu () {
      if (this.key != null) {
        this.oldKey = this.key
        this.key = null
        this.$emit('close', this.oldKey)
      }
    },

    updateCaretPosition () {
      if (this.key) {
        if (this.input.isContentEditable) {
          const rect = window.getSelection().getRangeAt(0).getBoundingClientRect()
          const inputRect = this.input.getBoundingClientRect()
          this.caretPosition = {
            left: rect.left - inputRect.left,
            top: rect.top - inputRect.top,
            height: rect.height,
          }
        } else {
          this.caretPosition = getCaretPosition(this.input, this.keyIndex)
        }
        this.caretPosition.top -= this.input.scrollTop
        if (this.$refs.popper && this.$refs.popper.popperInstance) {
          this.$refs.popper.popperInstance.scheduleUpdate()
        }
      }
    },

    applyMention (itemIndex) {
      const item = this.displayedItems[itemIndex]
      const value = (this.omitKey ? '' : this.key) + String(this.mapInsert ? this.mapInsert(item, this.key) : item.value) + (this.insertSpace ? ' ' : '')
      if (this.input.isContentEditable) {
        const range = window.getSelection().getRangeAt(0)
        range.setStart(range.startContainer, range.startOffset - this.key.length - (this.lastSearchText ? this.lastSearchText.length : 0))
        range.deleteContents()
        range.insertNode(document.createTextNode(value))
        range.setStart(range.endContainer, range.endOffset)
        this.emitInputEvent('input')
      } else {
        this.setValue(this.replaceText(this.getValue(), this.searchText, value, this.keyIndex))
        this.setCaretPosition(this.keyIndex + value.length)
      }
      this.$emit('apply', item, this.key)
      this.closeMenu()
    },

    replaceText (text, searchText, newText, index) {
      return text.slice(0, index) + newText + text.slice(index + searchText.length + 1, text.length)
    },
  },
}
</script>

<template>
  <div
    class="mentionable"
    style="position:relative;"
  >
    <slot />

    <VPopover
      ref="popper"
      v-bind="$attrs"
      :placement="placement"
      :open="!!key"
      trigger="manual"
      :auto-hide="false"
      class="popper"
      style="position:absolute;"
      :style="caretPosition ? {
        top: `${caretPosition.top}px`,
        left: `${caretPosition.left}px`,
      } : {}"
    >
      <div
        :style="caretPosition ? {
          height: `${caretPosition.height}px`,
        } : {}"
      />

      <template #popover>
        <div v-if="!displayedItems.length">
          <slot name="no-result">
            No result
          </slot>
        </div>

        <template v-else>
          <div
            v-for="(item, index) of displayedItems"
            :key="index"
            class="mention-item"
            :class="{
              'mention-selected': selectedIndex === index,
            }"
            @mouseover="selectedIndex = index"
            @mousedown="applyMention(index)"
          >
            <slot
              :name="`item-${key || oldKey}`"
              :item="item"
              :index="index"
            >
              <slot
                name="item"
                :item="item"
                :index="index"
              >
                {{ item.label || item.value }}
              </slot>
            </slot>
          </div>
        </template>
      </template>
    </VPopover>
  </div>
</template>
