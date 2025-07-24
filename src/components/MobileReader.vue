<template>
  <div>
    <div
      v-if="bgImage"
      class="bg-blur"
      :style="{ backgroundImage: 'url(' + bgImage + ')' }"
    ></div>
    <div class="reader-main">
      <div class="top-group" v-show="!pureMode">
        <input
          v-if="!rawText"
          type="file"
          accept=".txt"
          @change="handleFileUpload"
          class="file-input-mobile"
        />
        <button
          class="next-btn-mobile"
          @click="nextPage"
          :disabled="currentPage + 1 >= pages.length"
          style="display: none"
        >
          <
        </button>

        <span class="page-info-mobile">
          第{{ currentPage + 1 }}页 / 共{{ pages.length }}页
          <input
            type="number"
            v-model="jumpPageInput"
            min="1"
            :max="pages.length"
            placeholder="跳转至"
            @keydown.enter="jumpToPage"
            class="no-spin jump-inline-input"
            style="
              width: 36px;
              margin-left: 6px;
              font-size: 13px;
              padding: 1px 4px;
            "
          />
          <button
            @click="jumpToPage"
            class="jump-inline-btn"
            style="
              margin-left: 2px;
              font-size: 13px;
              padding: 1px 6px;
              height: 24px;
            "
          >
            跳转
          </button>
        </span>
        <button
          class="prev-btn-mobile"
          @click="prevPage"
          :disabled="currentPage === 0"
          style="display: none"
        >
          >
        </button>
      </div>
      <!-- 移除.jump-panel-mobile -->
      <div
        class="settings-float-panel-mobile"
        :class="{ open: settingsPanelOpen }"
        @click="settingsPanelOpen = true"
        @dblclick="settingsPanelOpen = false"
        v-show="!pureMode"
      >
        <div v-if="!settingsPanelOpen" class="settings-toggle-btn-mobile">
          ⚙️
        </div>
        <div v-else class="settings-panel-mobile">
          <label><input type="checkbox" v-model="pureMode" /> 纯净模式</label>
          <label
            >每页字数：<input
              type="number"
              v-model.number="charsPerPage"
              min="50"
              max="2000"
              @change="splitPages"
          /></label>
          <label
            >行高：<input
              type="number"
              step="0.1"
              v-model.number="lineHeight"
              min="0.8"
              max="3"
          /></label>
          <label
            >字符间距：<input
              type="number"
              step="0.1"
              v-model.number="letterSpacing"
              min="0"
              max="10"
          /></label>
          <label
            >背景图片：<input
              type="file"
              accept="image/*"
              @change="handleBgUpload"
          /></label>
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
      <div class="book-mobile">
        <transition name="fade-page" mode="out-in">
          <div
            class="page-mobile glass-effect"
            v-if="pages[currentPage]"
            :key="currentPage"
            @click="handlePageClick"
          >
            <div
              class="vertical-text-mobile"
              :style="{
                lineHeight: lineHeight,
                letterSpacing: letterSpacing + 'px',
              }"
            >
              {{ pages[currentPage] }}
            </div>
          </div>
        </transition>
      </div>
    </div>
  </div>
</template>

<script>
export default {
  name: "MobileReader",
  data() {
    return {
      rawText: "",
      pages: [],
      currentPage: 0,
      charsPerPage: 350,
      lineHeight: 1.2,
      letterSpacing: 1.4,
      settingsPanelOpen: false,
      jumpPageInput: "",
      bgImage: "",
      pureMode: false,
    }
  },
  mounted() {
    const saved = localStorage.getItem("jpReaderSettings")
    if (saved) {
      const s = JSON.parse(saved)
      this.charsPerPage = s.charsPerPage || this.charsPerPage
      this.lineHeight = s.lineHeight || this.lineHeight
      this.letterSpacing = s.letterSpacing || this.letterSpacing
      this.pureMode = s.pureMode || false
    }
    const savedText = localStorage.getItem("jpReaderText")
    if (savedText) {
      this.rawText = savedText
      this.splitPages()
      const savedPage = parseInt(
        localStorage.getItem("jpReaderCurrentPage") || "0",
        10
      )
      this.currentPage = isNaN(savedPage) ? 0 : savedPage
    }
    const savedBg = localStorage.getItem("jpReaderBgImage")
    if (savedBg) {
      this.bgImage = savedBg
    }
    // 移除 touchstart/touchend 事件监听
  },
  beforeDestroy() {
    // 移除 touchstart/touchend 事件注销
  },
  methods: {
    handleFileUpload(e) {
      const file = e.target.files[0]
      if (!file) return
      const reader = new FileReader()
      reader.onload = (event) => {
        this.rawText = event.target.result
        localStorage.setItem("jpReaderText", this.rawText)
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
      for (let i = 0; i < paragraphs.length; i++) {
        let para = paragraphs[i]
        let testPage = page ? page + "\n" + para : para
        let testCharCount = testPage.replace(/\n/g, "").length
        if (testCharCount > this.charsPerPage) {
          if (page) pages.push(page)
          page = para
        } else {
          page = testPage
        }
      }
      if (page) pages.push(page)
      this.pages = pages
    },
    prevPage() {
      if (this.currentPage > 0) {
        this.currentPage -= 1
        localStorage.setItem("jpReaderCurrentPage", this.currentPage)
      }
    },
    nextPage() {
      if (this.currentPage + 1 < this.pages.length) {
        this.currentPage += 1
        localStorage.setItem("jpReaderCurrentPage", this.currentPage)
      }
    },
    jumpToPage() {
      let page = parseInt(this.jumpPageInput, 10) - 1
      if (isNaN(page) || page < 0 || page >= this.pages.length) {
        alert("页码无效")
        return
      }
      this.currentPage = page
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
      this.splitPages()
      this.$forceUpdate()
    },
    clearCache() {
      localStorage.removeItem("jpReaderText")
      localStorage.removeItem("jpReaderBgImage")
      localStorage.removeItem("jpReaderCurrentPage")
      this.rawText = ""
      this.pages = []
      this.currentPage = 0
      this.bgImage = ""
      this.jumpPageInput = ""
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
    handlePageClick(e) {
      const rect = e.currentTarget.getBoundingClientRect()
      const x = e.clientX - rect.left
      if (x < rect.width / 2) {
        this.nextPage()
      } else {
        this.prevPage()
      }
    },
  },
}
</script>

<style>
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
.reader-main {
  min-height: 100vh;
  padding: 5px 0;
  position: relative;
  display: flex;
  flex-direction: column;
  justify-content: center;
}
.top-group {
  display: flex;
  justify-content: space-between;
  align-items: center;
  padding: 8px 4vw;
  gap: 8px;
}
.file-input-mobile {
  background: #e0d7c3;
  color: #3e2f1c;
  border-radius: 6px;
  padding: 6px 10px;
  border: 1px solid #cbbfa3;
}
.prev-btn-mobile,
.next-btn-mobile {
  background: #e0d7c3;
  color: #3e2f1c;
  border: 1px solid #cbbfa3;
  border-radius: 6px;
  padding: 6px 14px;
  font-size: 15px;
  cursor: pointer;
}
.page-info-mobile {
  color: #aaa;
  background: #f7f4e8;
  border-radius: 6px;
  padding: 6px 14px;
  font-size: 15px;
  user-select: none;
  border: 1px solid #e0d7c3;
}
.jump-panel-mobile {
  display: none !important;
}
.page-info-mobile,
.vertical-text-mobile,
.settings-panel-mobile,
.jump-inline-input,
.jump-inline-btn {
  color: #222 !important;
  background: transparent;
}
@media (prefers-color-scheme: dark) {
  .page-info-mobile,
  .vertical-text-mobile,
  .settings-panel-mobile,
  .jump-inline-input,
  .jump-inline-btn {
    color: #eee !important;
    background: transparent;
  }
}
.settings-float-panel-mobile {
  position: fixed;
  top: 0;
  left: 0;
  width: 100vw;
  z-index: 100;
  transition: background 0.3s;
  background: transparent;
  display: flex;
  align-items: flex-start;
  justify-content: center;
}
.settings-float-panel-mobile.open {
  background: #f7f4e8;
  border-radius: 12px 12px 0 0;
  box-shadow: 0 -2px 12px #0008;
  padding: 8px 12px 8px 18px;
}
.settings-toggle-btn-mobile {
  width: 36px;
  height: 36px;
  background: #e0d7c3;
  color: #3e2f1c;
  border-radius: 12px 12px 0 0;
  display: flex;
  align-items: center;
  justify-content: center;
  font-size: 22px;
  cursor: pointer;
  box-shadow: 0 -2px 8px #0006;
  margin-bottom: 0;
}
.settings-panel-mobile {
  display: flex;
  flex-direction: column;
  gap: 12px;
  color: #3e2f1c;
  background: transparent;
  font-size: 15px;
}
.settings-panel-mobile label {
  margin-right: 0;
}
.settings-panel-mobile input {
  width: 80px;
  margin-left: 4px;
  background: #f3efe2;
  color: #3e2f1c !important;
  border: 1px solid #cbbfa3;
  border-radius: 4px;
  padding: 2px 6px;
}
.settings-panel-mobile input::placeholder {
  color: #3e2f1c !important;
  opacity: 0.7;
}
.settings-panel-mobile button {
  background: #e0d7c3;
  color: #3e2f1c;
  border: 1px solid #cbbfa3;
  border-radius: 6px;
  padding: 4px 12px;
  font-size: 15px;
  cursor: pointer;
  align-self: flex-start;
}
.book-mobile {
  display: flex;
  justify-content: center;
  align-items: flex-start;
  margin-top: 12px;
}
.page-mobile {
  width: 96vw;
  height: 85vh;
  margin: 0;
  display: flex;
  align-items: center;
  justify-content: center;
  position: relative;
}
.page-tap-area {
  position: absolute;
  top: 0;
  width: 50%;
  height: 100%;
  z-index: 10;
}
.page-tap-area.left {
  left: 0;
}
.page-tap-area.right {
  right: 0;
}
.vertical-text-mobile {
  writing-mode: vertical-rl;
  text-orientation: mixed;
  font-size: 18px;
  color: #3e2f1c;
  height: 90%;
  overflow-y: auto;
  white-space: pre-wrap;
  padding: 12px;
  background: transparent;
  scrollbar-width: thin;
  scrollbar-color: rgba(180, 180, 180, 0.3) rgba(255, 255, 255, 0.08);
}
.vertical-text-mobile::-webkit-scrollbar {
  width: 8px;
  background: rgba(255, 255, 255, 0.08);
  border-radius: 8px;
}
.vertical-text-mobile::-webkit-scrollbar-thumb {
  background: rgba(255, 255, 255, 0.28);
  border-radius: 8px;
  box-shadow: 0 2px 8px 0 rgba(0, 0, 0, 0.08);
  backdrop-filter: blur(8px) saturate(120%);
  -webkit-backdrop-filter: blur(8px) saturate(120%);
  border: 1px solid rgba(255, 255, 255, 0.18);
}
.vertical-text-mobile::-webkit-scrollbar-thumb:hover {
  background: rgba(255, 255, 255, 0.38);
}
.fade-page-enter-active,
.fade-page-leave-active {
  transition: opacity 0.2s;
}
.fade-page-enter,
.fade-page-leave-to {
  opacity: 0.1;
}
.jump-inline-input {
  display: inline-block;
  vertical-align: middle;
  height: 22px;
  font-size: 13px;
  padding: 1px 4px;
}
.jump-inline-btn {
  display: inline-block;
  vertical-align: middle;
  height: 24px;
  font-size: 13px;
  padding: 1px 6px;
  background: #e0d7c3;
  color: #3e2f1c;
  border: 1px solid #cbbfa3;
  border-radius: 6px;
  cursor: pointer;
}
</style>
