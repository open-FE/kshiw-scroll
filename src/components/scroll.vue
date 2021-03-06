<template>
    <div ref="wrapper" class="scroll-wrap" @mouseenter="_enter($event)" @mouseleave="_leave($event)">
        <div ref="list" style="display: inline-block">
            <slot></slot>
        </div>
        <div class="scroll-bar" :class="{on: verticalCur}" v-show="verticalShow" ref="barV" @click.stop="_clickY($event)">
            <div class="scroll-bar-main" ref="barVMain" @mousedown.stop="_downY($event)"></div>
        </div>
        <div class="scroll-bar-h" :class="{on: horizontalCur}" v-show="horizontalShow" ref="barH" @click.stop="_click($event)">
            <div class="scroll-bar-h-main" ref="barHMain" @mousedown.stop="_down($event)"></div>
        </div>
    </div>
</template>

<script>
    const TEST_FLAG_COL = process.env.NODE_ENV !== 'production'
    export default {
        name: "scroll",
        props: {
            horizontal: {
                type: Boolean,
                default: false
            },
            vertical: {
                type: Boolean,
                default: false
            },
            // 终端 false为PC 待扩展
            terminal: {
                type: Boolean,
                default: false
            },
            // 速度
            speed: {
                type: Number,
                default: 300
            },
            wrapW: {
                type: Number,
                default: undefined
            },
            eqid: {
                type: String
            },
            verticalBarShow: {
                type: Boolean,
                default: false
            },
            clTop: {
                type: Number,
                default: 0
            }
        },
        data() {
            return {
                verticalShow: this.vertical,
                horizontalShow: this.horizontal,
                barMinX: 0,
                barMaxX: 0,
                barMinY: 0,
                barMaxY: 0,
                timer: null,
                verticalCur: false,
                horizontalCur: false,
            }
        },
        methods: {
            // 初始化
            _initScroll() {
                this._set()
                this._refresh()
            },
            // 设置参数
            _set() {
                this.mouse = {}  // mouse对象
                this.refWrap = this.$refs.wrapper
                this.refList = this.$refs.list
                console.log(this.refWrap.offsetWidth, this.refList.offsetWidth)
                if (this.horizontal) {
                    this.listW = this.wrapW || this.refList.offsetWidth
                    this.barH = this.$refs.barH
                    this.barHMain = this.$refs.barHMain
                }
                if (this.vertical) {
                    this.barV = this.$refs.barV
                    this.barVMain = this.$refs.barVMain
                }
                this.refList.style.position = 'relative'
            },
            // 计算刷新
            _refresh() {
                TEST_FLAG_COL && console.log('scroll_refresh')
                /**
                 * 横向
                 */
                if (this.horizontal) {
                    this.horizontalShow = true
                    if (this.listW <= this.refWrap.offsetWidth) {
                        this.horizontalShow = false
                        this._scrollX_To(0, 'set')
                        return
                    }
                    else {
                        this.barHMain.style.width = this.refWrap.offsetWidth / this.listW * this.barH.offsetWidth + 'px'
                        this.$nextTick(() => {
                            let offX = this.refWrap.offsetWidth * this.barH.offsetWidth / this.listW
                            this.barHMain.style.width = offX + 'px'
                            this.barMinX = this.barH.getBoundingClientRect().left
                            this.barMaxX = Math.floor(this.barH.offsetWidth - offX)
                            this.mouse.proportionX = (this.refList.offsetWidth - this.barH.offsetWidth) / this.barMaxX
                        })
                    }
                }
                /**
                 * 纵向
                 */
                if (this.vertical) {
                    this.verticalShow = true
                    this.listH = this.refList.offsetHeight
                    console.log(this.listH, this.refWrap.offsetHeight)
                    // 小于母体高度 隐藏
                    if (this.listH <= this.refWrap.offsetHeight) {
                        this.verticalBarShow = false
                        this.verticalShow = false
                        return
                    } else {
                        this.$nextTick(() => {
                            this.barVMain.style.height = this.refWrap.offsetHeight / this.listH * this.barV.offsetHeight + 'px'
                            let offY = this.refWrap.offsetHeight * this.barV.offsetHeight / this.listH
                            this.barMinY = this.barV.getBoundingClientRect().top
                            this.barMaxY = Math.floor(this.barV.offsetHeight - offY)
                            this.mouse.proportionY = (this.refList.offsetHeight - this.barV.offsetHeight) / this.barMaxY
                        })
                    }
                }
            },
            _down(even) {
                this.mouse.startX = even.clientX
                this.mouse.switch = true
                this.mouse.x = even.target.offsetLeft
                this.barHMain.className += ' cur'
            },
            _downY(even) {
                this.mouse.startY = even.clientY
                this.mouse.switchY = true
                this.mouse.y = even.target.offsetTop
            },
            _enter() {
                // TEST_FLAG_COL && console.log('_enter')
                if (this.verticalShow) {
                    this.verticalCur = true
                }
                if (this.horizontalShow) {
                    this.horizontalCur = true
                }
            },
            _leave() {
                this.horizontalCur = false
                this.verticalCur = false
            },
            _click(even) {
                if (this.mouse.switch) {
                    return
                }
                this.mouse.x = 0
                this.mouse.startX = this.barMinX
                this._animated(this.barHMain, 'left')
                this._scrollX_To(even.clientX - this.barHMain.offsetWidth * 0.5)
            },
            _clickY(even) {
                if (this.mouse.switchY) {
                    return
                }
                this.mouse.startY = this.barMinY
                this.mouse.y = 0
                this._animated(this.barVMain, 'top')
                this._scrollY_To(even.clientY - this.barVMain.offsetHeight * 0.5)
            },
            _scrollX_To(vx, type) {
                let x = 0
                this.horizontalCur = true
                if (type == 'set') {
                    x = vx
                } else {
                    x = vx - (this.mouse.startX || 0) + (this.mouse.x || 0)
                }
                this.mouse.nowX = x <= 0 ? 0 : (x >= this.barMaxX ? this.barMaxX : x)
                this.barHMain.style.left = this.mouse.nowX + 'px'
                this.refList.style.left = -1 * this.mouse.nowX * this.mouse.proportionX + 'px'
            },
            _scrollY_To(dy, type) {
                let y = 0
                this.verticalCur = true
                if (type == 'wheel' || type == 'set') {
                    y = dy
                } else {
                    y = dy - (this.mouse.startY || 0) + (this.mouse.y || 0)
                }
                this.mouse.nowY = y <= 0 ? 0 : (y >= this.barMaxY ? this.barMaxY : y)
                this.barVMain.style.top = this.mouse.nowY + 'px'
                this.refList.style.top = -1 * this.mouse.nowY * this.mouse.proportionY + 'px'
            },
            _animated(el, direction) {
                TEST_FLAG_COL && console.log('_animated')
                clearTimeout(this.timer)
                this.refList.style.transition = direction + ' ' + this.speed + 'ms'
                el.style.transition = direction + ' ' + this.speed + 'ms'
                this.timer = setTimeout(() => {
                    el.style.transition = direction + ' ' + '0ms'
                    this.refList.style.transition = direction + ' ' + '0ms'
                }, this.speed)
            },
            _mouseup() {
                TEST_FLAG_COL && console.log('_mouseup')
                if (this.mouse) {
                    this.mouse.switch = false
                    this.mouse.switchY = false
                    this.verticalCur = false
                    this.horizontalCur = false
                }
            },
            _wheel(event) {
                let sp = this.speed / 20
                let cy = event.deltaY > 0 ? (this.mouse.nowY || 0) + sp : (this.mouse.nowY || 0) - sp
                this.verticalShow ? this._scrollY_To(cy, 'wheel') : ''
                event.preventDefault();
            },
            _mousemove(even) {
                // TEST_FLAG_COL && console.log('_mousemove')
                if (this.mouse && this.mouse.switch) {
                    window.getSelection ? window.getSelection().removeAllRanges() : document.selection.empty()  // 禁止拖动时选中文本
                    this._scrollX_To(even.clientX)
                }
                if (this.mouse && this.mouse.switchY) {
                    window.getSelection ? window.getSelection().removeAllRanges() : document.selection.empty()  // 禁止拖动时选中文本
                    this._scrollY_To(even.clientY)
                }
                return false;
            },
        },
        mounted() {
            this._initScroll()
            window.addEventListener('mousemove', this._mousemove)
            window.addEventListener('mouseup', this._mouseup)
            this.vertical ? this.$refs.wrapper.addEventListener('wheel', this._wheel) : ''
            window.onresize = () => {
                TEST_FLAG_COL && console.log('onsize')
                this._refresh()
                this.horizontal ? this._scrollX_To(this.mouse.nowX, 'set') : ''
                this.vertical ? this._scrollY_To(this.mouse.nowY, 'set') : ''
            }
        },
        destroyed() {
            window.removeEventListener('mouseup', this._mouseup)
            window.removeEventListener('mousemove', this._mousemove)
            window.onresize = null
        }
    }
</script>

<style scoped>
    .scroll-wrap {
        padding-bottom: 0px;
        position: relative;
        height: 100%
    }

    .scroll-bar {
        position: absolute;
        top: 0;
        bottom: 0;
        right: 0px;
        width: 7px;
        border-radius: 5px;
        overflow: hidden;
        background-color: rgba(0, 0, 0, 0.05);
        opacity: 0;
        transition: opacity 250ms;
        z-index: 100;
    }

    .scroll-bar.on {
        opacity: 1;
    }

    .scroll-bar:hover {
        cursor: pointer
    }

    .scroll-bar .scroll-bar-main {
        position: absolute;
        top: 0;
        right: 0;
        left: 0;
        height: 30%;
        border-radius: 5px;
        background-color: rgba(0, 0, 0, 0.5);
    }

    .scroll-bar-h {
        position: absolute;
        bottom: 0;
        right: 0;
        left: 0;
        height: 8px;
        border-radius: 5px;
        overflow: hidden;
        background-color: rgba(0, 0, 0, 0.05);
        cursor: pointer;
        z-index: 100;
        opacity: 0;
        transition: opacity 250ms;
    }

    .scroll-bar-h.on {
        opacity: 1;
    }

    .scroll-bar-h .scroll-bar-h-main {
        position: absolute;
        left: 0;
        height: 100%;
        border-radius: 5px;
        background-color: rgba(0, 0, 0, 0.5);
        cursor: pointer;
        z-index: 100;
    }

    .scroll-bar-h .scroll-bar-h-main.cur {
        background-color: #555;
        transition: background-color 0.3s;
        -webkit-transition: background-color 0.3s;
        -moz-transition: background-color 0.3s;
    }
</style>
