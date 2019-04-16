<template>
  <div class="ebook-reader">
    <div class="ebook-reader-mask"
         @touchmove="move"
         @touchend="moveEnd"
         @mousedown.left="onMouseEnter"
         @mousemove.left="onMouseMove"
         @mouseup.left="onMouseEnd" @click="onMaskClick"></div>
    <div class="read-wrapper">
      <div id="read"></div>
    </div>
  </div>
</template>

<script type="text/ecmascript-6">
  import Epub from 'epubjs'
  import { ebookMixin } from '@/utils/mixin'
  import { getLocalForage } from '../../utils/localForage'
  import {
    getTheme,
    getFontFamily,
    saveFontFamily,
    saveTheme,
    getFontSize,
    saveFontSize,
    saveMetadata,
    getLocation
  } from '../../utils/localStorage'

  global.ePub = Epub
  export default {
    mixins: [ebookMixin],
    data() {
      return {
        havePaginate: false,
        isOnline: false
      }
    },
    methods: {
      move(e) {
        let offsetY = 0
        if (this.firstOffsetY) {
          offsetY = e.changedTouches[0].clientY - this.firstOffsetY
          this.$store.commit('SET_OFFSETY', offsetY)
        } else {
          this.firstOffsetY = e.changedTouches[0].clientY
        }
        e.preventDefault()
        e.stopPropagation()
      },
      onMouseEnter(e) {
        this.mouseMove = 1
        this.mouseStartTime = e.timeStamp
        e.preventDefault()
        e.stopPropagation()
      },
      onMouseMove(e) {
        if (this.mouseMove === 1) {
          this.mouseMove = 2
        } else if (this.mouseMove === 2) {
          let offsetY = 0
          if (this.firstOffsetY) {
            offsetY = e.clientY - this.firstOffsetY
            this.$store.commit('SET_OFFSETY', offsetY)
          } else {
            this.firstOffsetY = e.clientY
          }
        }
        e.preventDefault()
        e.stopPropagation()
      },
      onMouseEnd(e) {
        if (this.mouseMove === 2) {
          this.$store.dispatch('setOffsetY', 0)
          this.firstOffsetY = 0
          this.mouseMove = 3
        }
        this.mouseEndTime = e.timeStamp
        const time = this.mouseEndTime - this.mouseStartTime
        if (time < 200) {
          this.mouseMove = 1
        }
        e.preventDefault()
        e.stopPropagation()
      },
      moveEnd(e) {
        this.$store.dispatch('setOffsetY', 0)
        this.firstOffsetY = 0
      },
      onMaskClick(e) {
        if (this.mouseMove === 2) {
        } else if (this.mouseMove === 1 || this.mouseMove === 4) {
          const offsetX = e.offsetX
          const width = window.innerWidth
          if (offsetX > 0 && offsetX < width * 0.3) {
            this.prevPage()
          } else if (offsetX > 0 && offsetX > width * 0.7) {
            this.nextPage()
          } else {
            this.toggleMenuVisible()
          }
        }
        this.mouseMove = 4
      },
      prevPage() {
        if (this.rendition) {
          this.rendition.prev()
          this.refreshLocation()
        }
        this.hideMenuVisible()
      },
      nextPage() {
        if (this.rendition) {
          this.rendition.next()
          this.refreshLocation()
        }
        this.hideMenuVisible()
      },
      initGuest() {
        this.rendition.on('touchstart', event => {
          this.touchStartX = event.changedTouches[0].clientX
          this.touchStartTime = event.timeStamp
        })
        this.rendition.on('touchend', event => {
          const offsetX = event.changedTouches[0].clientX - this.touchStartX
          const time = event.timeStamp - this.touchStartTime
          if (time < 500 && offsetX > 40) {
            this.prevPage()
          } else if (time < 500 && offsetX < -40) {
            this.nextPage()
          } else {
            this.toggleMenuVisible()
          }
          event.preventDefault()
          event.stopPropagation()
        })
      },
      initTheme() {
        let defaultTheme = getTheme(this.fileName)
        if (!defaultTheme) {
          defaultTheme = this.themeList[0].name
          saveTheme(this.fileName, defaultTheme)
        }
        return defaultTheme
      },
      initFontSize() {
        let fontSize = getFontSize(this.fileName)
        if (!fontSize) {
          fontSize = 16
          saveFontSize(this.fileName, fontSize)
        }
        return fontSize
      },
      initFontFamily() {
        let font = getFontFamily(this.fileName)
        if (!font) {
          font = 'Default'
          saveFontFamily(this.fileName, font)
        }
        return font
      },
      initRendition() {
        this.rendition = this.book.renderTo('read', {
          width: window.innerWidth,
          height: window.innerHeight,
          method: 'default'
        })
        Promise.all([
          this.setDefaultTheme(this.initTheme()),
          this.setDefaultFontSize(this.initFontSize()),
          this.setDefaultFontFamily(this.initFontFamily())
        ]).then(() => {
          this.switchTheme()
          if (this.$route.query.navigation) {
            this.display(this.$route.query.navigation)
          } else {
            const location = getLocation(this.fileName)
            if (location) {
              this.display(location)
            } else {
              this.display()
            }
          }
        })
        this.rendition.hooks.content.register(contents => {
          Promise.all([
            contents.addStylesheet(`${process.env.VUE_APP_RES_URL}/fonts/daysOne.css`),
            contents.addStylesheet(`${process.env.VUE_APP_RES_URL}/fonts/tangerine.css`),
            contents.addStylesheet(`${process.env.VUE_APP_RES_URL}/fonts/montserrat.css`),
            contents.addStylesheet(`${process.env.VUE_APP_RES_URL}/fonts/cabin.css`)
          ]).then(() => {})
        })
      },
      parseBook() {
        this.book.loaded.metadata.then(metadata => {
          this.setMetadata(metadata)
          saveMetadata(this.fileName, metadata)
        })
        if (this.isOnline) {
          this.book.coverUrl().then(url => {
            this.setCover(url)
          })
        } else {
          this.book.loaded.cover.then(cover => {
            this.book.archive.createUrl(cover).then(url => {
              this.setCover(url)
            })
          })
        }
        this.book.loaded.navigation.then(nav => {
          console.log(nav)
          const navItem = (function flatten(arr) {
            return [].concat(...arr.map(v => [v, ...flatten(v.subitems)]))
          })(nav.toc)

          function find(item, v = 0) {
            const parent = navItem.filter(it => it.id === item.parent)[0]
            return !item.parent ? v : (parent ? find(parent, ++v) : v)
          }

          navItem.forEach(item => {
            item.level = find(item)
            item.total = 0
            item.pagelist = []
            if (item.href.match(/^(.*)\.html$/)) {
              item.idhref = item.href.match(/^(.*)\.html$/)[1]
            } else if (item.href.match(/^(.*)\.xhtml$/)) {
              item.idhref = item.href.match(/^(.*)\.xhtml$/)[1]
            }
          })
          this.setNavigation(navItem)
        })
        this.book.ready.then(() => {
          this.setCurrentBook(this.book)
          return this.book.locations.generate(750 * (window.innerWidth / 375) * (getFontSize(this.fileName) / 16))
        }).then(locations => {
          locations.forEach(location => {
            const loc = location.match(/\[(.*)\]!/)[1]
            console.log(loc)
            this.navigation.forEach(item => {
              if (item.idhref && item.idhref.indexOf(loc) >= 0) {
                item.pagelist.push(location)
              }
            })
            let currentPage = 1
            this.navigation.forEach((item, index) => {
              if (index === 0) {
                item.page = 1
              } else {
                item.page = currentPage
              }
              currentPage += item.pagelist.length + 1
            })
          })
          // saveNavigation(this.fileName, this.navigation)
          this.setPagelist(locations)
          this.setBookAvailable(true)
          this.setIsPaginating(false)
          this.refreshLocation()
        })
      },
      initEpub(target) {
        this.book = new Epub(target)
        this.setCurrentBook(this.book)
        this.setIsPaginating(true)
        this.setPaginate(this.$t('book.paginating'))
        this.initRendition()
        this.initGuest()
        this.parseBook()
      }
    },
    mounted() {
      if (this.$route.params.fileName.indexOf('|') > 0) {
        this.setFileName(
          this.$route.params.fileName.split('|').join('/'))
          .then(() => {
            // 实时下载电子书
            this.initEpub(`${process.env.VUE_APP_EPUB_URL}/${this.fileName}.epub`)
            this.isOnline = false
          })
      } else {
        this.setFileName(this.$route.params.fileName)
          .then(() => {
            getLocalForage(this.fileName, (err, blob) => {
              if (!err) {
                if (blob) {
                  // 离线阅读模式
                  this.isOnline = false
                  this.initEpub(blob)
                } else {
                  // 在线阅读模式
                  this.isOnline = true
                  const opf = this.$route.query.opf
                  if (opf) {
                    this.initEpub(opf)
                  }
                }
              }
            })
          })
      }
    }
  }
</script>

<style lang="scss" rel="stylesheet/scss" scoped>
  @import "../../assets/styles/global";

  .ebook-reader {
    width: 100%;
    height: 100%;
    overflow: hidden;
    .ebook-reader-mask {
      position: absolute;
      z-index: 150;
      top: 0;
      left: 0;
      width: 100%;
      height: 100%;
    }
  }
</style>
