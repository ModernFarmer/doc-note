<template>
  <div class="_um-_note-container" :style="containerStyle">
    <div class="_um-_note-headbox _um-_not-chooseable"
      :style="headBoxStyle"
    >
      <div class="_um-_unfold-box"
        v-if="foldable || foldable === ''"
        :style="{ width: showing ? 'calc(100% - 42px)' : 'calc(100% - 27px)' }"
        @click="toFoldOrUnfold"
      >
        <div class="_um-_unfold-text"
          :style="{ left: showing ? '16px' : '0', color: unfoldColor }"
        ><div class="_um-_unfold-scale-text">{{ showing ? 'fold' : 'unfold note' }}</div></div>
        <div class="_um-_unflod-arrow"
          :style="{ left: showing ? '0' : '55px', color: arrowColor }"
        >{{ showing ? '‹‹‹' : '›››' }}</div>
      </div>
      <div
        class="_um-_note-config-edit"
        v-if="editable || editable === ''"
        v-show="showing"
        :style="{background: edit ? '#0fec3f' : 'red'}"
        @click="toEdit"
      ></div>
      <div
        class="_um-_note-config-submit"
        v-if="editable || editable === ''"
        v-show="contentChange"
        @click.stop="toSubmit"
      ></div>
      <div
        class="_um-_submit-confirm"
        :style="{ background: confirmBackground }"
        v-if="submit"
        @click.stop
      >
        <div class="_um-_submit-item-cancel" @click="submit = false"><span class="_um-submit-span">×</span></div>
        <div class="_um-_submit-item-ok" @click="toHandleSubmit">√</div>
      </div>
    </div>
    <pre
      ref="preRef"
      class="_um-_pre-class language-"
      :style="{ height }"
      @scroll="handleScroll"
    ><div
      :id="`${item.key}_codeBox`"
      class="_um-_code-box"
      v-for="(item, index) in codeList" :key="`${item.key}_${index}`"
    ><div
      class="_um-_sign-public"
      :style="lanStyle"
    >{{ item.showingLanguage }}</div><div
      class="_um-_line-dashed"
      :style="dashedStyle"
    ></div><div
      class="_um-_sign-add _um-_not-chooseable"
      :style="addStyle"
      v-show="edit"
      @click.stop="toAdd(index, item.key)"
    >+</div><div
      class="_um-_sign-minus _um-_not-chooseable"
      :style="minusStyle"
      v-show="edit && codeList.length > 1"
      @click.stop="toRemove(index)"
    >-</div><div
      :id="`${item.key}_selectBox`"
      class="_um-_select-container"
      :style="selectStyle"
      v-if="add && index === addIndex"
    ><div
      class="_um-_select-item _um-_not-chooseable"
      v-for="(val, ind) in languageList"
      :key="`${val.fnKey}_${ind}`"
      @click="toHandleAdd(val, index, item.key)"
    >{{ val.value }}</div></div><div
      class="_um-_confirm-container"
      :style="confirmStyle"
      v-if="remove && index === removeIndex"
    ><div
      class="_um-_confirm-message"
      :style="{ color: confirmMessageColor }"
    >{{ deleteObj.explain }}</div><div
      class="_um-_confirm-item _um-_not-chooseable"
      v-for="val in deleteObj.list"
      :key="`fnBtn_${val.key}`"
      @click="toHandleRemove(val.key, item.key, index)"
    >{{ val.value }}</div></div><code
        :id="item.key"
        class="_um-_code-class"
        :contenteditable="edit"
        v-html="item.processedCode"
        @input="handleInput($event, item, '__input')"
        @keydown="handleInput($event, item, '__tabDown')"
        @paste="handleInput($event, item, '__paste')"
        @compositionstart="limitInput($event, item, '__input')"
        @compositionend="limitInput($event, item, '__input')"
      ></code></div></pre>
  </div>
</template>

<script>
import { getLanguage, getShowingLanguage, getKey, selection, setCore, _BD, _unBD, UM_NOTE_CONFIG } from './publicData'
import { themeConfigMap } from './theme'

const themesData = themeConfigMap[UM_NOTE_CONFIG.theme]

export default {
  props: {
    language: {
      type: String,
      default: 'javascript',
    },
    codes: {
      type: [Array, Object, String],
      default () {
        return []
      },
    },
    width: {
      type: String,
      default: '100%',
    },
    height: {
      type: String,
      default: 'auto',
    },
    editable: {
      type: Boolean,
      default: false,
    },
    unfold: {
      type: Boolean,
      default: false,
    },
    foldable: {
      type: Boolean,
      default: true,
    },
  },
  coreObj: {},
  codeTagEl: null, // 当添加弹框内的语言数量过多, 可能会导致弹框高度超过代码块高度, codeTagEl用于储存最近一次打开添加弹框的代码块div, 便于关闭弹框时初始化它的高度
  data() {
    let showing = this.foldable === false ? true : (this.unfold || this.unfold === '' ? true : false)
    setTimeout(() => {
      showing = null
    })
    return {
      coreObj: {}, // 核心数据对象, 包含codeList数组内代表的每个dom的核心方法(getFrontOffset, getRealDomAndOffset)、根元素(root)、光标所在元素(container)、需要插入的内容(inset)
      canInput: true, // 输入中文时, 在输入过程中且没有确定中文字符时, input也会触发, 这里限制一下绑定在input上面的监听事件 -> 用于this.handleInput()
      codeTagHeightOrigin: null, // 未打开弹框时代码块div的原始高度, 用于初始化codeTagEl的高度
      last_lang: 0, // 上一次组件内容横向滚动的距离, 用于固定定位添加删除功能等功能按钮
      codeList: [],
      languageList: getLanguage.list,
      contentChange: false,
      deleteObj: {
        explain: UM_NOTE_CONFIG.contentNames?.explain || '确定删除?',
        list: [
          { key: 0, value: UM_NOTE_CONFIG.contentNames?.cancel || '取消' },
          { key: 1, value: UM_NOTE_CONFIG.contentNames?.confirm || '确定' },
        ]
      },
      showing,
      containerStyle: {
        width: showing ? (this.width === 'auto' ? 'auto' : this.width) : '260px',
        height: showing ? 'auto' : '16px',
        background: themesData.container_background,
      },
      headBoxStyle: { background: showing ? 'transparent' : themesData.head_background_hidden },
      lanStyle: { color: themesData.language_color, left: 0, top: 0 },
      addStyle: { left: `calc(100% - 22px)`, bottom: 0 },
      minusStyle: { left: `calc(100% - 38px)`, bottom: 0 },
      dashedStyle: { width: 'calc(100% - 1em)', borderBottom: `1px dashed ${themesData.block_hr_background}`, left: 0, bottom: '8px' },
      selectStyle: {
        border: `1px solid ${themesData.languageSelect_border}`,
        background: themesData.languageSelect_container,
        outline: `2px solid ${themesData.languageSelect_container}`,
        bottom: '-5px',
        right: '27px',
      },
      confirmStyle: {
        border: `1px solid ${themesData.deleteSelect_border}`,
        background: themesData.languageSelect_container,
        outline: `2px solid ${themesData.languageSelect_container}`,
        bottom: '-5px',
        right: '47px',
      },
      confirmBackground: themesData.edit_container,
      confirmMessageColor: themesData.confirm_message_color,
      unfoldColor: themesData.unfold_text_color,
      arrowColor: themesData.unfold_arrow_color,
      add: false,
      edit: false,
      remove: false,
      submit: false,
      addIndex: null,
      removeIndex: null,
    }
  },
  watch: {
    showing(bl) {
      this.headBoxStyle = { background: bl ? 'transparent' : themesData.head_background_hidden }
    },
    codes: {
      handler(codes) {
        const coreObjJson = {}
        if (Array.isArray(codes)) { // 如果codes是一个数组
          if (codes.length) {
            this.codeList = codes.map(item => {
              const _language = getLanguage(item.language)
              const language = _language === 'javascript' ? getLanguage(this.language) : _language
              const showingLanguage = getShowingLanguage(language)
              const key = getKey()
              setCore(coreObjJson, key)
              return {
                key,
                language,
                showingLanguage,
                code: item.code || '',
                processedCode: window.Prism.highlight(item.code || '', window.Prism.languages[language], language),
              }
            })
          } else {
            const key = getKey()
            setCore(coreObjJson, key)
            this.codeList = [{
              key,
              language: 'javascript',
              showingLanguage: 'JavaScript',
              code: '',
              processedCode: '',
            }]
          }
        } else if (typeof codes === 'object') { // 如果codes是一个json
          const _language = getLanguage(codes.language)
          const language = _language === 'javascript' ? getLanguage(this.language) : _language
          const showingLanguage = getShowingLanguage(language)
          const key = getKey()
          setCore(coreObjJson, key)
          this.codeList = [
            {
              key,
              language,
              showingLanguage,
              code: codes.code || '',
              processedCode: window.Prism.highlight(codes.code || '', window.Prism.languages[language], language),
            }
          ]
        } else if (typeof codes === 'string') { // 如果codes是一个字符串
          const language = getLanguage(this.language)
          const showingLanguage = getShowingLanguage(language)
          const key = getKey()
          setCore(coreObjJson, key)
          this.codeList = [
            {
              key,
              language,
              showingLanguage,
              code: codes || '',
              processedCode: window.Prism.highlight(codes || '', window.Prism.languages[language], language),
            }
          ]
        } else {
          throw `The prop 'codes' must be an array or object or a string!`
        }
        this.$options.coreObj = coreObjJson
        this.contentChange = false

        if (this.showing) {
          this.$nextTick(() => {
            this.add = false
            this.remove = false
            this.submit = false
            this.setContainerHeight()
          })
        }
      },
      deep: true,
      immediate: true,
    },
    edit(bl) {
      if (bl) {
        this.dashedStyle.width = 'calc(100% - 47px)'
      } else {
        this.dashedStyle.width = 'calc(100% - 1em)'
      }
    },
  },
  methods: {
    setContainerHeight() {
      this.containerStyle.height = `${this.$refs.preRef.offsetHeight + 19}px`
    },
    toFoldOrUnfold() {
      this.showing = !this.showing
      if (this.showing) {
        this.containerStyle.width = this.width
        this.setContainerHeight()
      } else {
        this.edit = false
        this.containerStyle.width = '260px'
        this.containerStyle.height = '16px'
      }
    },
    handleScroll() {
      const offset = this.$refs.preRef.scrollLeft
      if (this.last_lang !== offset) {
        this.lanStyle.left = this.dashedStyle.left = `${offset}px`
        this.addStyle.left = `calc(100% + ${offset - 22}px)`
        this.minusStyle.left = `calc(100% + ${offset - 38}px)`
        this.selectStyle.right = `calc(27px - ${offset}px)`
        this.confirmStyle.right = `calc(47px - ${offset}px)`
        this.last_lang = offset
      }
    },
    limitInput(e, item, handleType) {
      this.canInput = !e.isTrusted
      // 中文输入确定的那一刻不会触发input事件, 所以在compositionend事件触发且event.data有内容时(在中文输入过程中删除所有内容时也会触发compositionend事件)须触发一次handleInput事件
      if (this.canInput && e.data) {
        this.handleInput(e, item, handleType)
      }
    },
    handleInput(e, item, handleType) {
      if (!selection.haveRange() || !this.canInput || !this.edit) return // 如果窗口中没有Range对象 或 中文输入过程中 或 组件无法编辑状态, 拦截
      const targetObj = this.$options.coreObj[item.key]
      targetObj.container = selection.getEndContainer()
      if (!targetObj.root) targetObj.root = document.querySelector(`#${item.key}`)
      targetObj.inset = ''
      if (handleType === '__tabDown' && e.keyCode !== 9) { // 这里是监听按键按下事件, 如果handleType === '__tabDown'且按下的不是tab键, 则阻断执行
        return
      } else if (handleType === '__tabDown') { // 如果按下的是tab键, 则取消默认
        // 由于默认按下tab键会使容器失去焦点, 所以这里要tab按下事件的监听重写
        e.preventDefault()
        targetObj.inset = '  '
      } else if (handleType === '__paste') { // 监听粘贴事件
        // 由于默认粘贴会将粘贴板上所有的样式都会粘贴到目标容器, 可能会导致样式错乱, 所以这里要做粘贴事件的监听重写
        e.preventDefault()
        const clipboard = e.clipboardData || window.clipboardData
        if(clipboard) {
          targetObj.container = selection.getStartContainer()
          selection.deleteContents()
          targetObj.inset = clipboard.getData("text/plain").toString().replace(/\r\n/g, '\n')
        } else {
          alert('Paste is not supported, please enter it manually!')
          return
        }
      }

      targetObj.getFrontOffset(targetObj.root, targetObj.container, targetObj.inset, (totalOffset, textContent) => {
        const realContent = window.Prism.highlight(textContent, window.Prism.languages[item.language], item.language)
        // 当realContent === item.processedCode时, 不会触发页面更新, 须手动更新
        // 适用于全选内容并粘贴的情况
        if (realContent === item.processedCode) {
          targetObj.root.innerHTML = realContent
        } else {
          // 这里的item对象就是this.codeList[当前索引], 利用引用型对象浅拷贝的特性, 直接操作item
          item.processedCode = realContent
          this.contentChange = true
          this.$nextTick(() => {
            if (this.height === 'auto') this.setContainerHeight()
          })
        }
        item.code = textContent
        this.$nextTick(() => {
          targetObj.getRealDomAndOffset(targetObj.root, totalOffset, (el, i) => {
            selection.setCursorOffset(el, i)
          })
        })
      })
    },
    toAdd(index, elementKey) {
      this.remove = false
      this.submit = false
      if (UM_NOTE_CONFIG.addConfigure) {
        if (this.addIndex === index) {
          if (this.add) {
            this.add = false
            this._u_resetStyle()
          } else {
            UM_NOTE_CONFIG.addConfigure(this.next_add)
          }
        } else {
          UM_NOTE_CONFIG.addConfigure(this.next_add)
        }
      } else {
        if (this.addIndex === index) {
          this.add = !this.add
          if (!this.add) this._u_resetStyle()
        } else {
          this.add = true
        }
      }
      this.addIndex = index
      if (this.add) {
        this.$nextTick(() => {
          const selectEl = document.querySelector(`#${elementKey}_selectBox`)
          const _codeTagEl = document.querySelector(`#${elementKey}_codeBox`)
          if (selectEl.offsetHeight > _codeTagEl.offsetHeight + _codeTagEl.offsetTop - 11) {
            this.$options.codeTagEl = _codeTagEl
            this.codeTagHeightOrigin = this.$options.codeTagEl.offsetHeight
            this.$options.codeTagEl.style.height = `${selectEl.offsetHeight - 3}px`
          }
          if (this.height === 'auto') this.setContainerHeight()
        })
      }
    },
    toHandleAdd(val, index) {
      const key = getKey()
      this.codeList.splice(index + 1, 0, {
        key,
        language: val.fnKey,
        showingLanguage: val.value,
        code: '',
        processedCode: '',
      })
      for (let key in this.$options.coreObj) {
        this.$options.coreObj[key].root = null
      }
      setCore(this.$options.coreObj, key)
      this.contentChange = true
      this.$nextTick(() => {
        if (this.height === 'auto') this.setContainerHeight()
      })
    },
    toEdit() {
      if (UM_NOTE_CONFIG.editConfigure) {
        if (this.edit) {
          this.edit = false
        } else {
          UM_NOTE_CONFIG.editConfigure(this.next_edit)
        }
      } else {
        this.edit = !this.edit
      }
    },
    toSubmit() {
      this.add = false
      this.remove = false
      if (UM_NOTE_CONFIG.submitConfigure) {
        if (this.submit) {
          this.submit = false
        } else {
          UM_NOTE_CONFIG.submitConfigure(this.next_submit)
        }
      } else {
        this.submit = !this.submit
      }
    },
    toHandleSubmit() {
      const data = this.codeList.map(item => {
        const json = { ...item }
        Reflect.deleteProperty(json, 'key')
        return json
      })
      this.$emit('submit', { data, close: this.close })
    },
    toRemove(index) {
      this.add = false
      this.submit = false
      this._u_resetStyle()
      if (UM_NOTE_CONFIG.removeConfigure) {
        if (this.removeIndex === index) {
          if (this.remove) {
            this.remove = false
          } else {
            UM_NOTE_CONFIG.removeConfigure(this.next_remove)
          }
        } else {
          UM_NOTE_CONFIG.removeConfigure(this.next_remove)
        }
      } else {
        if (this.removeIndex === index) {
          this.remove = !this.remove
        } else {
          this.remove = true
        }
      }
      this.removeIndex = index
    },
    toHandleRemove(val, key, index) {
      if (val === 1) {
        this.codeList.splice(index, 1)
        Reflect.deleteProperty(this.$options.coreObj, key)
        for (let key in this.$options.coreObj) {
          this.$options.coreObj[key].root = null
        }
        this.contentChange = true
        this.$nextTick(() => {
          this.setContainerHeight()
        })
      }
    },
    domClick() {
      this.add = false
      this._u_resetStyle()
      this.remove = false
      this.submit = false
    },

    _u_resetStyle() {
      if (this.$options.codeTagEl) {
        this.$options.codeTagEl.style.height = `${this.codeTagHeightOrigin}px`
        if (this.height === 'auto') {
          this.setContainerHeight()
        }
        this.$options.codeTagEl = null
        this.codeTagHeightOrigin = null
      }
    },
    next_add() {
      this.add = true
    },
    next_edit() {
      this.edit = true
    },
    next_remove() {
      this.remove = true
    },
    next_submit() {
      this.submit = true
    },
    close() {
      this.add = false
      this.edit = false
      this.remove = false
      this.submit = false
    },
  },
  mounted() {
    _BD(document, 'click', this.domClick)
  },
  beforeDestroy() {
    _unBD(document, 'click', this.domClick)
  },
}
</script>
