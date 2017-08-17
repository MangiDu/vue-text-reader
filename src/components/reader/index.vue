<template lang="html">
  <div class="reader__wrapper">
    <div class="reader__settings">
      <input type="number" v-model="fontSize" min="12">
      <span @click="changePage('prev')">上一页</span>
      <span @click="changePage('next')">下一页</span>
      {{ activePage }}
    </div>
    <div class="reader__content" :style="{'font-size': fontSize + 'px', 'height': contentHeight + 'px', 'margin-top': marginGap + 'px', 'line-height': lineHeight}">
      <div class="reader__text" ref="text">
      </div>
    </div>
  </div>
</template>

<script>
const INDENT_PATTERN = /^\s{1,}/
export default {
  props: {
    text: {
      type: String
    }
  },
  data () {
    return {
      // 间距高度
      marginGap: 20,
      contentMinHeight: 350,
      contentHeight: 0,
      resizeDebounce: 500,
      // 文本样式
      fontSize: 16,
      lineHeight: 2,
      // 加载状态
      loading: false,
      // 阅读状态
      page: 1,
      activePage: {}
    }
  },
  computed: {
    paragraphArr () {
      // 将text文本转成html段落的字符串
      let pArr = this.text.split('\n')
      let paragraphArr = []
      for (let paragraph of pArr) {
        let result = paragraph.match(INDENT_PATTERN)
        let indentCount = (result && result[0] && result[0].length) || 0
        paragraphArr.push({
          html: getHtmlString('p', paragraph.substring(indentCount))
        })
      }
      return paragraphArr
    }
  },
  watch: {
    text () {
      if (!this.text || !this.text.length) return
      this.getPageList()
    }
  },
  mounted () {
    this.getContentHeight()
    // 窗口resize时要自适应高度，不产生滚动条
    this._resizeHandler = this.onWindowResize.bind(this)
    window.addEventListener('resize', this._resizeHandler)
  },
  beforeDestroy () {
    window.removeEventListener('resize', this._resizeHandler)
  },
  methods: {
    onWindowResize () {
      // 防抖
      if (this._resizeTimer) {
        clearTimeout(this._resizeTimer)
      }
      this._resizeTimer = setTimeout(this.getContentHeight.bind(this), this.resizeDebounce)
    },
    getContentHeight () {
      let windowHeight = window.innerHeight
      let minHeight = this.contentMinHeight
      let contentHeight = (windowHeight > minHeight ? windowHeight : minHeight) - this.marginGap * 2
      let maxLineCount = Math.floor(contentHeight / (this.fontSize * this.lineHeight))
      this.maxLineCount = maxLineCount
      this.contentHeight = maxLineCount * this.fontSize * this.lineHeight
    },
    getPageList () {
      if (!this.paragraphArr || !this.paragraphArr.length) {
        return []
      }
      // 把段落数组插入文档中
      this.$refs.text.innerHTML = this.paragraphArr.map(item => item.html).join('')
      let paraEls = this.$refs.text.children
      this.paragraphArr.forEach((para, index) => {
        let paraEl = paraEls.item(index)
        para.lineCount = paraEl.clientHeight / (this.fontSize * this.lineHeight)
      })
      let maxLineCount = this.maxLineCount
      let pageList = []
      let paraIndex = 0
      // let hiddenLineCount = 0
      do {
        let pageInfo = {
          startIdx: paraIndex,
          topHiddenLineCount: 0
        }
        if (pageList.length) {
          // 处理遮挡
          let prevPageInfo = pageList[pageList.length - 1]
          if (prevPageInfo.bottomHiddenLineCount) {
            pageInfo.topHiddenLineCount = this.paragraphArr[paraIndex].lineCount - prevPageInfo.bottomHiddenLineCount
          }
        }
        let count = 0
        for (let idx = paraIndex; idx < this.paragraphArr.length; idx++) {
          let paraItem = this.paragraphArr[idx]
          count += paraItem.lineCount
          if (count <= maxLineCount) {
            paraIndex += 1
          } else {
            pageInfo.endIdx = idx
            pageInfo.bottomHiddenLineCount = count - maxLineCount
            break
          }
        }
        pageList.push(pageInfo)
      } while (paraIndex < this.paragraphArr.length)
      this.pageList = pageList
      console.log(pageList)
      this.setPage()
    },
    changePage (direction) {
      switch (direction) {
        case 'prev':
          if (this.page > 1) {
            this.page--
            this.setPage()
          }
          break
        case 'next':
          let length = this.pageList.length
          if (this.page < length) {
            this.page++
            this.setPage()
          }
          break
        default:
      }
    },
    setPage () {
      this.activePage = this.pageList[this.page - 1]
    }
  }
}

function getHtmlString (name, content) {
  return `<${name}>${content}</${name}>`
}
</script>

<style lang="css">
.reader__wrapper {
  position: relative;
}
.reader__settings {
  position: absolute;
}
.reader__content {
  position: relative;
  margin-left: auto;
  margin-right: auto;
  width: 60%;
  overflow: hidden;
}
.reader__text p {
  margin: 0;
  text-indent: 2em;
}
</style>
