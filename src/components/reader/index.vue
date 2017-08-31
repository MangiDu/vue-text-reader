<template lang="html">
  <div class="reader__wrapper" :style="{'padding-top': marginGap + 'px'}">
    <div class="reader__settings" :style="{'height': marginGap + 'px', 'line-height': marginGap + 'px'}">
      <!-- <input type="number" v-model="fontSize" min="12"> -->
      <span v-if="page > 1" @click="changePage('prev')">上一页</span>
      <span v-if="page < pageList.length" @click="changePage('next')">下一页</span>
    </div>
    <div class="reader__content" :style="{'width': contentWidth + 'px', 'height': contentHeight + 'px', 'font-size': fontSize + 'px', 'line-height': lineHeight}">
      <div class="reader__text" ref="text"></div>
      <div class="reader__text" v-html="currentContent" :style="{'margin-top': -1 * fontSize * lineHeight * activePage.topHiddenLineCount + 'px'}"></div>
    </div>
  </div>
</template>

<script>
export default {
  props: {
    text: {
      type: String
    }
  },
  data () {
    return {
      // 间距高度宽度等
      marginGap: 20,
      contentMinHeight: 350,
      contentHeight: 0,
      contentWidthBoundary: 800,
      contentMinWidth: 400,
      contentWidth: 0,
      resizeDebounce: 500,
      // 文本样式
      fontSize: 16,
      lineHeight: 2,
      // 加载状态
      loading: false,
      // 文本数据
      maxLineCount: -1,
      pageList: [],
      // 阅读状态
      page: 1, // 当前阅读页，永远比index大1
      activePage: {},
      anchor: {}
    }
  },
  computed: {
    paragraphArr () {
      // 将text文本转成html段落的字符串
      let pArr = this.text.split('\n')
      let paragraphArr = []
      let wordsCount = 0
      pArr.forEach(function (paragraph, idx) {
        let content = paragraph.trim()
        let contentLength = content.length
        // 为每个字添加index定位信息
        content = content.split('').map((word, index) => getHtmlString('span', word, {
          'word-index': wordsCount + index
        })).join('')
        paragraphArr.push({
          html: getHtmlString('p', content, {
            'para-index': idx
          }),
          wordStartIdx: wordsCount,
          wordEndIdx: wordsCount + (contentLength - 1)
        })
        wordsCount += contentLength
      })

      return paragraphArr
    },
    currentContent () {
      // 当前阅读页面
      if (!this.pageList || !this.pageList.length) return ''
      let htmlString = ''
      let currentPageInfo = this.pageList[this.page - 1]
      let startIdx = currentPageInfo.paraStartIdx || 0
      let endIdx = currentPageInfo.paraEndIdx || (this.paragraphArr.length - 1)
      for (let i = startIdx; i <= endIdx; i++) {
        htmlString += this.paragraphArr[i].html
      }
      return htmlString
    }
  },
  watch: {
    text () {
      if (!this.text || !this.text.length) return
      this.getPageList()
    }
  },
  mounted () {
    this.getContentSize()
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
      this._resizeTimer = setTimeout(this.getContentSize.bind(this), this.resizeDebounce)
    },
    getContentSize () {
      // 计算内容高度
      let windowHeight = window.innerHeight
      let minHeight = this.contentMinHeight
      let contentHeight = Math.max(windowHeight, minHeight) - this.marginGap * 2
      let maxLineCount = Math.floor(contentHeight / (this.fontSize * this.lineHeight))
      this.maxLineCount = maxLineCount
      this.contentHeight = maxLineCount * this.fontSize * this.lineHeight
      // 计算内容宽度
      let windowWidth = window.innerWidth

      let widthBoundary = this.contentWidthBoundary
      let widthRadio
      if (windowWidth > widthBoundary) widthRadio = 0.6
      else if (windowWidth > this.contentMinWidth) {
        widthRadio = 0.6 + 0.4 * (Math.abs(widthBoundary - windowWidth) / (widthBoundary - this.contentMinWidth))
      } else widthRadio = 1

      let wantedWidth = windowWidth * widthRadio

      this.contentWidth = windowWidth > this.contentMinWidth ? wantedWidth : windowWidth
      this.$nextTick(() => {
        this.getPageList({
          sizeChanged: true
        })
      })
    },
    getPageList (option) {
      if (!this.paragraphArr || !this.paragraphArr.length) {
        return []
      }
      // 重置页码为1
      this.page = 1
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
          paraStartIdx: paraIndex,
          topHiddenLineCount: 0
        }
        if (pageList.length) {
          // 处理遮挡
          let prevPageInfo = pageList[pageList.length - 1]
          if (prevPageInfo.bottomHiddenLineCount) {
            pageInfo.topHiddenLineCount = this.paragraphArr[paraIndex].lineCount - prevPageInfo.bottomHiddenLineCount
          }
        }
        // 需要把隐藏的顶部行数计算进去
        let count = -1 * pageInfo.topHiddenLineCount
        for (let idx = paraIndex; idx < this.paragraphArr.length; idx++) {
          let paraItem = this.paragraphArr[idx]
          count += paraItem.lineCount
          if (count <= maxLineCount) {
            paraIndex += 1
          } else {
            pageInfo.paraEndIdx = idx
            pageInfo.bottomHiddenLineCount = count - maxLineCount
            break
          }
        }
        pageList.push(pageInfo)
      } while (paraIndex < this.paragraphArr.length)

      // TODO：页面载入时惰性求值并保存
      // TODO：写成工具方法
      // 如果在初始化时计算所有页面的边界，在超长段落的情况下，可能会花费太久时间
      // 计算并设置每页的边界字idx
      pageList.forEach((page) => {
        // 计算起始文字定位边界
        let paraIdx = page.paraStartIdx
        let paraInfo = this.paragraphArr[paraIdx]
        let paraEl = document.querySelector(`p[para-index="${paraIdx}"]`)

        let paraOffsetTop = paraEl.offsetTop
        let topLineCount = page.topHiddenLineCount

        for (let i = paraInfo.wordStartIdx, count = paraInfo.wordEndIdx; i <= count; i++) {
          let spanEl = document.querySelector(`span[word-index="${i}"]`)
          // 因为此时全部的段落和文字都在文档中，所以要减去段落的偏移
          let offsetTop = spanEl.offsetTop - paraOffsetTop
          if (offsetTop > topLineCount * (this.fontSize * this.lineHeight)) {
            // 如果高出了段落顶部隐藏的高度就说明已经到显示的正文区域了
            page.wordStartIdx = i
            break
          }
        }

        // 计算结束文字定位边界
        // 如果没有末尾定位信息，可能是两种情况：1. 还没有内容时； 2. 末尾页； 默认使用最后一段
        paraIdx = page.paraEndIdx || this.paragraphArr.length - 1
        paraInfo = this.paragraphArr[paraIdx]
        paraEl = document.querySelector(`p[para-index="${paraIdx}"]`)

        paraOffsetTop = paraEl.offsetTop
        let bottomHiddenLineCount = page.bottomHiddenLineCount
        topLineCount = paraInfo.lineCount - bottomHiddenLineCount

        for (let i = paraInfo.wordStartIdx, count = paraInfo.wordEndIdx; i <= count; i++) {
          let spanEl = document.querySelector(`span[word-index="${i}"]`)
          // 因为此时全部的段落和文字都在文档中，所以要减去段落的偏移
          let offsetTop = spanEl.offsetTop - paraOffsetTop
          if (offsetTop > topLineCount * (this.fontSize * this.lineHeight)) {
            // 如果高出了段落顶部隐藏的高度就说明已经离开正文区域到隐藏区域了
            page.wordEndIdx = i - 1
            break
          }
        }
      })

      this.pageList = pageList

      this.$refs.text.innerHTML = ''
      this.setPage(option)
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
    setPage (option = {}) {
      let pageObj
      if (option.sizeChanged) {
        // 如果是resize引发的页面重排，则要重置当前页码和设置包含之前阅读位置的页面对象
        let paraIdx = this.anchor.paraIdx
        let startLineCount = this.anchor.startLineCount
        let paraLineCount = this.paragraphArr[paraIdx] && this.paragraphArr[paraIdx].lineCount
        for (let i = 0; i < this.pageList.length; i++) {
          let page = this.pageList[i]
          if ((paraIdx > page.paraStartIdx && paraIdx < page.paraEndIdx) || (paraIdx === page.paraStartIdx && page.topHiddenLineCount < startLineCount) || (paraIdx === page.paraEndIdx && page.bottomHiddenLineCount < (paraLineCount - startLineCount + 1))) {
            this.page = i + 1
            pageObj = page
            break
          }
        }
      } else {
        pageObj = this.pageList[this.page - 1]
      }
      if (pageObj && pageObj.paraStartIdx !== undefined) {
        this.activePage = pageObj
        this.anchor = {
          paraIdx: pageObj.paraStartIdx,
          startLineCount: pageObj.topHiddenLineCount + 1
        }
      }
    }
  }
}

function getHtmlString (name, content, attributes = {}) {
  let attrArr = []
  for (let key in attributes) {
    attrArr.push(` ${key}="${attributes[key]}"`)
  }
  return `<${name}${attrArr.join('')}>${content}</${name}>`
}
</script>

<style lang="less">
.reader__wrapper {
  position: relative;
  height: 100%;
  box-sizing: border-box;
  overflow: hidden;
}
.reader__settings {
  position: absolute;
  bottom: 0;
  left: 0;
  width: 100%;
  text-align: center;
}
.reader__content {
  position: relative;
  margin-left: auto;
  margin-right: auto;
  overflow: hidden;
}
.reader__text p {
  margin: 0;
  text-indent: 2em;
}
</style>
