<template>
  <div>
    <div
      v-if="bgImage"
      class="bg-blur"
      :style="{ backgroundImage: 'url(' + bgImage + ')' }"
    ></div>
    <div class="reader-main">
      <div class="left-abs-group" v-show="!pureMode">
        <input
          type="file"
          accept=".txt"
          @change="handleFileUpload"
          class="file-input-abs write-mode-tb"
        />
        <button
          class="next-btn-abs write-mode-tb"
          @click="nextPage"
          :disabled="currentPage + 2 >= pages.length"
        >
          下一页
        </button>
      </div>
      <div class="right-abs-group" v-show="!pureMode">
        <span class="page-info-abs">
          第{{ currentPage + 1 }}~{{
            Math.min(currentPage + 2, pages.length)
          }}页 / 共{{ pages.length }}页
        </span>
        <div class="jump-panel">
          <input
            type="number"
            v-model="jumpPageInput"
            min="1"
            :max="pages.length"
            placeholder="跳转至"
            @keydown.enter="jumpToPage"
            @keydown.up.prevent
            @keydown.down.prevent
            class="no-spin"
          />
          <button @click="jumpToPage">跳转</button>
        </div>
        <button
          class="prev-btn-abs write-mode-tb"
          @click="prevPage"
          :disabled="currentPage === 0"
        >
          上一页
        </button>
      </div>
      <div
        style="
          text-align: center;
          margin-bottom: 20px;
          display: flex;
          justify-content: center;
          align-items: center;
          gap: 12px;
        "
        v-show="!pureMode"
      >
        <!-- 设置面板右侧悬浮 -->
        <div
          class="settings-float-panel"
          :class="{ open: settingsPanelOpen }"
          @mouseenter="settingsPanelOpen = true"
          @mouseleave="settingsPanelOpen = false"
        >
          <div v-if="!settingsPanelOpen" class="settings-toggle-btn">⚙️</div>
          <div v-else class="settings-panel">
            <label>
              <input type="checkbox" v-model="pureMode" /> 纯净模式
            </label>
            <label>
              每页字数：
              <input
                type="number"
                v-model.number="charsPerPage"
                min="50"
                max="2000"
                @change="splitPages"
              />
            </label>
            <label>
              行高：
              <input
                type="number"
                step="0.1"
                v-model.number="lineHeight"
                min="0.8"
                max="3"
              />
            </label>
            <label>
              字符间距：
              <input
                type="number"
                step="0.1"
                v-model.number="letterSpacing"
                min="0"
                max="10"
              />
            </label>
            <label>
              背景图片：
              <input type="file" accept="image/*" @change="handleBgUpload" />
            </label>
            <button @click="saveSettings">保存设置</button>
            <button
              @click="clearCache"
              style="
                margin-top: 8px;
                color: #c00;
                border-color: #c00;
                background: #fff2;
              "
            >
              清除本地缓存
            </button>
          </div>
        </div>
      </div>
      <div class="book">
        <transition name="fade-page" mode="out-in">
          <div
            class="page right-page glass-effect"
            v-if="pages[currentPage]"
            :key="currentPage"
          >
            <div
              class="vertical-text"
              :style="{
                lineHeight: lineHeight,
                letterSpacing: letterSpacing + 'px',
              }"
            >
              {{ pages[currentPage] }}
            </div>
          </div>
        </transition>
        <div class="book-connector"><div class="spine-line"></div></div>
        <transition name="fade-page" mode="out-in">
          <div
            class="page left-page glass-effect"
            v-if="pages[currentPage + 1]"
            :key="currentPage + 1"
          >
            <div
              class="vertical-text"
              :style="{
                lineHeight: lineHeight,
                letterSpacing: letterSpacing + 'px',
              }"
            >
              {{ pages[currentPage + 1] }}
            </div>
          </div>
        </transition>
      </div>
    </div>
  </div>
</template>

<script>
export default {
  name: "JpReader",
  data() {
    return {
      rawText: "",
      pages: [],
      currentPage: 0,
      charsPerPage: 350, // 每页字数
      lineHeight: 1.2, // 行高
      letterSpacing: 1.4, // 字符间距
      settingsPanelOpen: false, // 设置面板开关
      jumpPageInput: "", // 跳转页码输入
      bgImage: "", // 背景图片 base64
      pureMode: false, // 纯净模式
    }
  },
  mounted() {
    // 读取本地设置
    const saved = localStorage.getItem("jpReaderSettings")
    if (saved) {
      const s = JSON.parse(saved)
      this.charsPerPage = s.charsPerPage || this.charsPerPage
      this.lineHeight = s.lineHeight || this.lineHeight
      this.letterSpacing = s.letterSpacing || this.letterSpacing
      this.pureMode = s.pureMode || false
    }
    // 读取本地小说
    const savedText = localStorage.getItem("jpReaderText")
    if (savedText) {
      this.rawText = savedText
      this.splitPages()
      // 读取进度，splitPages后再设置currentPage
      const savedPage = parseInt(
        localStorage.getItem("jpReaderCurrentPage") || "0",
        10
      )
      this.currentPage = isNaN(savedPage) ? 0 : savedPage
    }
    // 读取背景图片
    const savedBg = localStorage.getItem("jpReaderBgImage")
    if (savedBg) {
      this.bgImage = savedBg
    }
    // 监听鼠标滚轮和键盘左右键
    window.addEventListener("wheel", this.handleWheel, { passive: false })
    window.addEventListener("keydown", this.handleKeydown)
  },
  beforeDestroy() {
    window.removeEventListener("wheel", this.handleWheel)
    window.removeEventListener("keydown", this.handleKeydown)
  },
  methods: {
    handleFileUpload(e) {
      const file = e.target.files[0]
      if (!file) return
      const reader = new FileReader()
      reader.onload = (event) => {
        this.rawText = event.target.result
        localStorage.setItem("jpReaderText", this.rawText) // 保存小说内容
        this.splitPages()
        this.currentPage = 0
        localStorage.setItem("jpReaderCurrentPage", 0)
      }
      reader.readAsText(file, "utf-8")
    },
    splitPages() {
      const paragraphs = this.rawText.replace(/\r\n/g, "\n").split("\n")
      let pages = []
      let page = ""
      let emptyCount = 0
      for (let i = 0; i < paragraphs.length; i++) {
        let para = paragraphs[i]
        if (para.trim() === "") {
          emptyCount++
        } else {
          emptyCount = 0
        }
        let testPage = page ? page + "\n" + para : para
        let testCharCount = testPage.replace(/\n/g, "").length // 不计换行符
        // 检查字数或连续空行
        if (testCharCount > this.charsPerPage || emptyCount >= 3) {
          if (page) pages.push(page)
          page = para
          emptyCount = para.trim() === "" ? 1 : 0 // 新页如果是空行，计数重置为1，否则为0
        } else {
          page = testPage
        }
      }
      if (page) pages.push(page)
      this.pages = pages
    },
    prevPage() {
      if (this.currentPage >= 2) {
        this.currentPage -= 2
        localStorage.setItem("jpReaderCurrentPage", this.currentPage)
      }
    },
    nextPage() {
      if (this.currentPage + 2 < this.pages.length) {
        this.currentPage += 2
        localStorage.setItem("jpReaderCurrentPage", this.currentPage)
      }
    },
    jumpToPage() {
      let page = parseInt(this.jumpPageInput, 10) - 1
      if (isNaN(page) || page < 0 || page >= this.pages.length) {
        alert("页码无效")
        return
      }
      // 跳转到偶数页（右页）
      this.currentPage = page % 2 === 0 ? page : page - 1
      localStorage.setItem("jpReaderCurrentPage", this.currentPage)
      this.jumpPageInput = ""
    },
    saveSettings() {
      localStorage.setItem(
        "jpReaderSettings",
        JSON.stringify({
          charsPerPage: this.charsPerPage,
          lineHeight: this.lineHeight,
          letterSpacing: this.letterSpacing,
          pureMode: this.pureMode,
        })
      )
      this.splitPages() // 重新分页
      this.$forceUpdate() // 强制刷新样式
    },
    clearCache() {
      // 清除本地缓存
      localStorage.removeItem("jpReaderText")
      localStorage.removeItem("jpReaderBgImage")
      localStorage.removeItem("jpReaderCurrentPage")
      this.rawText = ""
      this.pages = []
      this.currentPage = 0
      this.bgImage = ""
      this.jumpPageInput = ""
      // 可选：刷新页面
      // location.reload()
      alert("本地缓存已清除！")
    },
    handleBgUpload(e) {
      const file = e.target.files[0]
      if (!file) return
      const reader = new FileReader()
      reader.onload = (event) => {
        this.bgImage = event.target.result
        localStorage.setItem("jpReaderBgImage", this.bgImage)
      }
      reader.readAsDataURL(file)
    },
    handleWheel(e) {
      if (e.ctrlKey || e.altKey || e.metaKey) return // 避免和缩放等冲突
      if (e.deltaY > 0) {
        // 下滚
        this.nextPage()
        e.preventDefault()
      } else if (e.deltaY < 0) {
        // 上滚
        this.prevPage()
        e.preventDefault()
      }
    },
    handleKeydown(e) {
      if (e.ctrlKey || e.altKey || e.metaKey) return
      if (e.key === "ArrowLeft") {
        this.nextPage()
        e.preventDefault()
      } else if (e.key === "ArrowRight") {
        this.prevPage()
        e.preventDefault()
      }
    },
  },
}
</script>

<style>
:root {
  --page-bg: #f7f4e8;
  --page-border: #e0d7c3;
  --text-color: #3e2f1c;
  --main-bg: #f3efe2;
  --btn-bg: #e0d7c3;
  --btn-color: #3e2f1c;
  --btn-border: #cbbfa3;
  --panel-bg: #f7f4e8;
  --panel-border: #e0d7c3;
  --input-bg: #f3efe2;
}
@media (prefers-color-scheme: dark) {
  :root {
    --page-bg: #181818;
    --page-border: #ccc;
    --text-color: #e6e6e6;
    --main-bg: #111;
    --btn-bg: #222;
    --btn-color: #eee;
    --btn-border: #888;
    --panel-bg: #222d;
    --panel-border: #888;
    --input-bg: #222;
  }
}
body,
html,
#app {
  background: var(--main-bg) !important;
}
.bg-blur {
  position: fixed;
  z-index: 0;
  top: 0;
  left: 0;
  right: 0;
  bottom: 0;
  width: 100vw;
  height: 100vh;
  background-size: cover;
  background-position: center;
  filter: blur(5px) brightness(0.6);
  pointer-events: none;
}
.book,
.left-abs-group,
.right-abs-group,
.settings-float-panel,
.page,
.vertical-text {
  position: relative;
  z-index: 1;
}
/* 液态玻璃效果 */
.glass-effect {
  background: rgba(255, 255, 255, 0.18) !important;
  border-radius: 5px;
  box-shadow: 0 4px 24px 0 rgba(0, 0, 0, 0.1) !important;
  backdrop-filter: blur(58px) saturate(160%);
  -webkit-backdrop-filter: blur(58px) saturate(160%);
  border: 1.5px solid rgba(255, 255, 255, 0.28) !important;
}
@media (prefers-color-scheme: dark) {
  .glass-effect {
    background: rgba(30, 30, 30, 0.18);
    border: 1.5px solid rgba(255, 255, 255, 0.1);
  }
}
.book {
  display: flex;
  flex-direction: row-reverse;
  justify-content: center;
  align-items: flex-start;
  gap: 0px;
}
.page {
  width: 35vw;
  height: 95vh;
  /* background: var(--page-bg);  交给glass-effect */
  /* border: 1.5px solid var(--page-border);  交给glass-effect */
  /* border-radius: 8px;  交给glass-effect */
  /* box-shadow: 0 4px 24px #d6cdbb88, 0 1px 0 #fff8 inset;  交给glass-effect */
  margin: 0 3px;
  display: flex;
  align-items: center;
  justify-content: center;
  /* 可选纸张纹理 */
  /* background-image: url('paper-texture.png'); */
}
.vertical-text {
  writing-mode: vertical-rl;
  text-orientation: mixed;
  font-size: 20px;
  color: var(--text-color);
  height: 90%;
  overflow-y: auto;
  white-space: pre-wrap;
  padding: 20px;
  background: transparent;
  /* 自定义滚动条样式 */
  scrollbar-width: thin;
  scrollbar-color: rgba(180, 180, 180, 0.3) rgba(255, 255, 255, 0.08);
}
/* Webkit滚动条样式（液态玻璃风格） */
.vertical-text::-webkit-scrollbar {
  width: 10px;
  background: rgba(255, 255, 255, 0.08);
  border-radius: 8px;
}
.vertical-text::-webkit-scrollbar-thumb {
  background: rgba(255, 255, 255, 0.28);
  border-radius: 8px;
  box-shadow: 0 2px 8px 0 rgba(0, 0, 0, 0.08);
  backdrop-filter: blur(8px) saturate(120%);
  -webkit-backdrop-filter: blur(8px) saturate(120%);
  border: 1px solid rgba(255, 255, 255, 0.18);
}
.vertical-text::-webkit-scrollbar-thumb:hover {
  background: rgba(255, 255, 255, 0.38);
}
/* 全局滚动条液态玻璃风格 */
body::-webkit-scrollbar {
  width: 12px;
  background: rgba(255, 255, 255, 0.08);
}
body::-webkit-scrollbar-thumb {
  background: rgba(255, 255, 255, 0.18);
  border-radius: 8px;
  box-shadow: 0 2px 8px 0 rgba(0, 0, 0, 0.08);
  backdrop-filter: blur(8px) saturate(120%);
  -webkit-backdrop-filter: blur(8px) saturate(120%);
  border: 1px solid rgba(255, 255, 255, 0.12);
}
body::-webkit-scrollbar-thumb:hover {
  background: rgba(255, 255, 255, 0.28);
}
@media (prefers-color-scheme: dark) {
  .vertical-text::-webkit-scrollbar {
    background: rgba(30, 30, 30, 0.18);
  }
  .vertical-text::-webkit-scrollbar-thumb {
    background: rgba(80, 80, 80, 0.38);
    border: 1px solid rgba(255, 255, 255, 0.08);
  }
  .vertical-text::-webkit-scrollbar-thumb:hover {
    background: rgba(120, 120, 120, 0.48);
  }
  body::-webkit-scrollbar {
    background: rgba(30, 30, 30, 0.18);
  }
  body::-webkit-scrollbar-thumb {
    background: rgba(80, 80, 80, 0.28);
    border: 1px solid rgba(255, 255, 255, 0.08);
  }
  body::-webkit-scrollbar-thumb:hover {
    background: rgba(120, 120, 120, 0.38);
  }
}
button {
  background: var(--btn-bg);
  color: var(--btn-color);
  border: 1px solid var(--btn-border);
  border-radius: 6px;
  padding: 8px 18px;
  margin: 0 8px;
  font-size: 16px;
  cursor: pointer;
  transition: background 0.2s, color 0.2s;
  box-shadow: 0 1px 4px #e0d7c355;
}
button:disabled {
  background: #eee8;
  color: #aaa;
  cursor: not-allowed;
}
.file-input-abs {
  background: var(--btn-bg);
  color: var(--btn-color);
  border-radius: 6px;
  padding: 8px 12px;
  border: 1px solid var(--btn-border);
}
.left-abs-group {
  position: absolute;
  left: 0;
  top: 50%;
  transform: translateY(-50%);
  z-index: 10;
  display: flex;
  flex-direction: column;
  align-items: flex-start;
  gap: 16px;
}
.next-btn-abs {
  margin-top: 16px;
}
.right-abs-group {
  position: absolute;
  right: 0;
  top: 50%;
  transform: translateY(-50%);
  z-index: 10;
  display: flex;
  flex-direction: column;
  align-items: flex-end;
  gap: 16px;
}
.page-info-abs {
  color: #aaa;
  background: var(--panel-bg);
  border-radius: 6px;
  padding: 8px 18px;
  font-size: 16px;
  user-select: none;
  border: 1px solid var(--panel-border);
}
.prev-btn-abs {
  margin-top: 16px;
}
.write-mode-tb {
  writing-mode: tb;
}
.settings-float-panel {
  position: fixed;
  top: 60px;
  right: 0;
  z-index: 100;
  transition: width 0.3s, background 0.3s;
  width: 36px;
  height: auto;
  background: transparent;
  display: flex;
  align-items: flex-start;
}
.settings-float-panel.open {
  width: 250px;
  background: var(--panel-bg);
  border-radius: 12px 0 0 12px;
  box-shadow: -2px 2px 12px #0008;
  padding: 8px 12px 8px 18px;
  border: 1px solid var(--panel-border);
}
.settings-toggle-btn {
  width: 36px;
  height: 36px;
  background: var(--btn-bg);
  color: var(--btn-color);
  border-radius: 12px 0 0 12px;
  display: flex;
  align-items: center;
  justify-content: center;
  font-size: 22px;
  cursor: pointer;
  box-shadow: -2px 2px 8px #0006;
  margin-left: 0;
}
.settings-float-panel .settings-panel {
  display: flex;
  flex-direction: column;
  gap: 12px;
  color: var(--text-color);
  background: transparent;
  font-size: 16px;
}
.settings-float-panel .settings-panel label {
  margin-right: 0;
}
.settings-float-panel .settings-panel input {
  width: 80px;
  margin-left: 4px;
  background: var(--input-bg);
  color: var(--text-color) !important;
  border: 1px solid var(--btn-border);
  border-radius: 4px;
  padding: 2px 6px;
}
.settings-float-panel .settings-panel input::placeholder {
  color: var(--text-color) !important;
  opacity: 0.7;
}
.settings-float-panel .settings-panel button {
  background: var(--btn-bg);
  color: var(--btn-color);
  border: 1px solid var(--btn-border);
  border-radius: 6px;
  padding: 4px 12px;
  font-size: 15px;
  cursor: pointer;
  transition: background 0.2s;
  align-self: flex-start;
}
.jump-panel {
  display: flex;
  align-items: center;
  gap: 6px;
  margin-left: 12px;
}
.jump-panel input {
  width: 60px;
  background: #f3efe2;
  color: var(--text-color);
  border: 1px solid var(--btn-border);
  border-radius: 4px;
  padding: 2px 6px;
  /* 隐藏type=number的上下箭头 */
  -moz-appearance: textfield;
}
/* Chrome, Safari, Edge 隐藏number上下箭头 */
.jump-panel input::-webkit-outer-spin-button,
.jump-panel input::-webkit-inner-spin-button {
  -webkit-appearance: none;
  margin: 0;
}
.jump-panel button {
  background: var(--btn-bg);
  color: var(--btn-color);
  border: 1px solid var(--btn-border);
  border-radius: 6px;
  padding: 4px 10px;
  font-size: 15px;
  cursor: pointer;
  transition: background 0.2s;
}
.book-connector {
  width: 32px;
  height: 95vh;
  align-self: stretch;
  background: rgba(255, 255, 255, 0.22);
  backdrop-filter: blur(8px) saturate(120%);
  -webkit-backdrop-filter: blur(8px) saturate(120%);
  border-radius: 5px;
  box-shadow: 0 0 12px 0 rgba(0, 0, 0, 0.08);
  margin: 0 2px;
  display: flex;
  justify-content: center;
  align-items: center;
}
@media (prefers-color-scheme: dark) {
  .book-connector {
    background: rgba(30, 30, 30, 0.22);
  }
}
.spine-line {
  width: 2px;
  height: 80%;
  background: linear-gradient(to bottom, #fff8, #8888, #fff8);
  border-radius: 2px;
}
/* 在<style>末尾添加淡入淡出动画样式 */
.fade-page-enter-active,
.fade-page-leave-active {
  transition: opacity 0.2s;
}
.fade-page-enter,
.fade-page-leave-to {
  opacity: 0.1;
}
/* 让包裹.book的容器始终垂直居中 .book */
.reader-main {
  min-height: 100vh;
  padding: 5px 0;
  position: relative;
  display: flex;
  flex-direction: column;
  justify-content: center; /* 垂直居中 */
}
</style>
