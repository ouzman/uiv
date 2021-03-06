<template>
  <div tabindex="-1" role="dialog" class="modal" :class="{fade:transitionDuration>0}" @click.self="backdropClicked">
    <div ref="dialog" class="modal-dialog" :class="modalSizeClass" role="document">
      <div class="modal-content">
        <div class="modal-header" v-if="header">
          <slot name="header">
            <button
              type="button"
              class="close"
              aria-label="Close"
              @click="toggle(false)"
              style="position: relative;z-index: 1060"
              v-if="dismissBtn">
              <!-- 1060 is bigger than dialog z-index 1050 because it got cover by title sometimes -->
              <span aria-hidden="true">×</span>
            </button>
            <h4 class="modal-title">
              <slot name="title">{{title}}</slot>
            </h4>
          </slot>
        </div>
        <div class="modal-body">
          <slot></slot>
        </div>
        <div class="modal-footer" v-if="footer">
          <slot name="footer">
            <btn @click="toggle(false,'cancel')">
              <span>{{cancelText || t('uiv.modal.cancel')}}</span>
            </btn>
            <btn type="primary" @click="toggle(false,'ok')" data-action="auto-focus">
              <span>{{okText || t('uiv.modal.ok')}}</span>
            </btn>
          </slot>
        </div>
      </div>
    </div>
    <div ref="backdrop" class="modal-backdrop" :class="{fade:transitionDuration>0}"></div>
  </div>
</template>

<script>
  import Local from '../../mixins/localeMixin'
  import Btn from './../button/Btn'
  import {
    EVENTS,
    on,
    off,
    removeFromDom,
    toggleBodyOverflow,
    addClass,
    removeClass,
    getComputedStyle
  } from '../../utils/domUtils'
  import {isFunction} from '../../utils/objectUtils'

  const MODAL_BACKDROP = 'modal-backdrop'
  const IN = 'in'
  const getOpenModalNum = () => document.querySelectorAll(`.${MODAL_BACKDROP}`).length

  export default {
    mixins: [Local],
    components: {Btn},
    props: {
      value: {
        type: Boolean,
        default: false
      },
      title: String,
      size: String,
      backdrop: {
        type: Boolean,
        default: true
      },
      footer: {
        type: Boolean,
        default: true
      },
      header: {
        type: Boolean,
        default: true
      },
      cancelText: String,
      okText: String,
      dismissBtn: {
        type: Boolean,
        default: true
      },
      transitionDuration: {
        type: Number,
        default: 150
      },
      autoFocus: {
        type: Boolean,
        default: false
      },
      keyboard: {
        type: Boolean,
        default: true
      },
      beforeClose: Function,
      zOffset: {
        type: Number,
        default: 20
      }
    },
    data () {
      return {
        msg: '',
        timeoutId: 0
      }
    },
    computed: {
      modalSizeClass () {
        return {
          [`modal-${this.size}`]: Boolean(this.size)
        }
      }
    },
    watch: {
      value (v) {
        this.$toggle(v)
      }
    },
    mounted () {
      removeFromDom(this.$refs.backdrop)
      on(window, EVENTS.KEY_UP, this.onKeyPress)
      if (this.value) {
        this.$toggle(true)
      }
    },
    beforeDestroy () {
      clearTimeout(this.timeoutId)
      removeFromDom(this.$refs.backdrop)
      off(window, EVENTS.KEY_UP, this.onKeyPress)
    },
    methods: {
      onKeyPress (event) {
        if (this.keyboard && this.value && event.keyCode === 27) {
          this.toggle(false)
        }
      },
      toggle (show, msg) {
        // skip the hiding while show===false & beforeClose returning falsely value
        if (!show && isFunction(this.beforeClose) && !this.beforeClose()) {
          return
        }
        this.msg = msg
        this.$emit('input', show)
      },
      $toggle (show) {
        const modal = this.$el
        const backdrop = this.$refs.backdrop
        clearTimeout(this.timeoutId)
        if (show) {
          const alreadyOpenModalNum = getOpenModalNum()
          document.body.appendChild(backdrop)
          modal.style.display = 'block'
          modal.scrollTop = 0
          backdrop.offsetHeight // force repaint
          toggleBodyOverflow(false)
          addClass(backdrop, IN)
          addClass(modal, IN)
          // fix z-index for nested modals
          // no need to calculate if no modal is already open
          if (alreadyOpenModalNum > 0) {
            const modalBaseZ = parseInt(getComputedStyle(modal).zIndex) || 1050 // 1050 is default modal z-Index
            const backdropBaseZ = parseInt(getComputedStyle(backdrop).zIndex) || 1040 // 1040 is default backdrop z-Index
            const offset = alreadyOpenModalNum * this.zOffset
            modal.style.zIndex = `${modalBaseZ + offset}`
            backdrop.style.zIndex = `${backdropBaseZ + offset}`
          }
          // z-index fix end
          this.timeoutId = setTimeout(() => {
            if (this.autoFocus) {
              let btn = this.$el.querySelector('[data-action="auto-focus"]')
              if (btn) {
                btn.focus()
              }
            }
            this.$emit('show')
            this.timeoutId = 0
          }, this.transitionDuration)
        } else {
          removeClass(backdrop, IN)
          removeClass(modal, IN)
          this.timeoutId = setTimeout(() => {
            modal.style.display = 'none'
            removeFromDom(backdrop)
            if (getOpenModalNum() === 0) {
              toggleBodyOverflow(true)
            }
            this.$emit('hide', this.msg || 'dismiss')
            this.msg = ''
            this.timeoutId = 0
            // restore z-index for nested modals
            modal.style.zIndex = ''
            backdrop.style.zIndex = ''
            // z-index fix end
          }, this.transitionDuration)
        }
      },
      backdropClicked (event) {
        if (this.backdrop) {
          this.toggle(false)
        }
      }
    }
  }
</script>
