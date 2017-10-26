<template>
    <div class="reader-container">
        <div class="reader" ref="reader">
        </div>
    </div>
</template>
<style>
    .reader-container {
        width: 100%;
        height: 100%;
        display: flex;
        align-items: center;
        justify-content: center;
        padding: 0 4em;
        box-sizing: border-box;
    }

    @media screen and (max-width: 580px) {
        .reader-container {
            padding: 0 2em;
        }
    }

    /* .buttonlist{
                    flex: 0;
                    bottom: 2em;
                    left: 2em;
                    -webkit-app-region: no-drag;
                } */

    .reader {
        max-width: 100%;
        width: 1400px;
        height: 80%;
        flex-shrink: 0;
        -webkit-app-region: no-drag;
        font-family: "default";
        user-select: none;
        pointer-events: none;
    }

    .reader > * {
        pointer-events: none;
    }
</style>

<script>
    import { updateBook, getBookMeta } from "../BookOps"
    import Rx from '@reactivex/rxjs'
    import _ from 'lodash'    
    // import { do } from '@reactivex/rxjs/operator/do'

    export default {
        components: {

        },
        data() {
            return {
                book: null,
                font: null
            }
        },
        beforeDestroy(){
            document.removeEventListener('keydown', this.keyEvent, false)
            this.book = null
            //  关闭监听事件。。。
            this.$store._subscribers.pop()
        },
        async beforeMount(){
            let fontFace = new FontFace("defaultText", "url(../static/fonts/SourceHanSerifSC-Regular.otf)", {})
            this.font = await fontFace.load()
        },
        async mounted() {
            var _this = this
            
            await this.loadBook()

            //初始化渲染
            let reader = this.$el.querySelector('.reader')
            let rendition = this.book.renderTo(reader,{
                width: "100%",
                height: "100%",
                // manager: "continuous",
                flow: "paginated",//scorlled
                spread: "none"
            })       

            rendition.hooks.content.register((content)=>{
                content.document.fonts.add(_this.font)
            })
            
            //初始化样式
            this.setStyle(this.$store.state.reader.book)         
            rendition.themes.default({
                "*": {
                    'font-family': "defaultText"
                }
            });
             
            //开始渲染
            rendition.start()
            rendition.display( this.loadCFI() ) 

            //初始化按键事件，在对应方法里处理
            //准备把按键改为主进程监听

            this.$el.addEventListener('wheel', this.keyEvent, false)
            this.$el.addEventListener('keydown', this.keyEvent, false)
            this.book.rendition.on("keydown", _this.keyEvent.bind(_this))

            //初始化store监听
            this.$store.subscribe((mutation,state)=>{
                this.syncStore(mutation, state)
            })

            //。。。生成事件作为源
            //加上消抖，手速太快的时候写入DB会有冲突，控制台一篇红色。。。

            // var storeHandler = Rx.Observable.fromEvent(this.$el, 'storeChange')
            // .distinctUntilChanged((pre,cur)=> _.isEqual(pre.detail.state.reader.book, cur.detail.state.reader.book))
            // .throttleTime(100)           
            // .subscribe((event)=>{
            // })   
            // var storeChange = new CustomEvent('storeChange',{detail: {mutation: mutation,state: state}})
            // this.$el.dispatchEvent(storeChange)
          


            // 初始化阅读位置自动保存
            this.book.rendition.on('locationChanged',( location )=>{
                console.log("当前位置", location.start)
                this.$store.commit("SET_LASTREAD",{cfi: location.start})
            })

        },
        methods: {
            syncStore(mutation, state){
                if(this.$route.path == "/reader"){   
                    if(mutation.type == "SET_STYLE"){
                        let book = state.reader.book
                        this.setStyle(book)
                        updateBook(book)
                    }
                    else if(mutation.type == "SET_LASTREAD"){
                        let book = state.reader.book
                        updateBook(book)
                    }
                }
            },
            setStyle(book){
                if(book.config && this.book){            
                    if(book.config["font-size"]){
                        this.book.rendition.themes.fontSize(book.config["font-size"] + "px")
                    }
                    if(book.config["line-height"]){                    
                        this.book.rendition.themes.override("line-height", book.config["line-height"] + "em")
                    }
                }
            },
            keyEvent(event) {
                if (event.key == "ArrowDown" || event.key == "ArrowRight" || event.deltaX > 0 || event.deltaY > 0) {
                    console.log("下")
                    this.pageNext()
                }
                else if (event.key == "ArrowUp" || event.key == "ArrowLeft" || event.deltaX < 0 || event.deltaY < 0) {
                    console.log("上")
                    this.pagePrev()
                }
            },
            pageNext() {
                this.book.rendition.next()

            },
            pagePrev() {
                this.book.rendition.prev()
            },
            async loadBook() {
                let bookData = this.$store.state.reader.book
                if (bookData) {
                    this.book = ePub(bookData.url)   
                    let meta = await getBookMeta(bookData.url)
                    document.title = meta.title || meta.bookTitle || "无题"
                }
                else {
                    this.$router.push("/")
                }
            },
            loadCFI() {                
                let bookData = this.$store.state.reader.book
                let cfi = bookData.lastRead
                return cfi ? cfi : undefined          
            }
        },
        watch: {

        }
    }
</script>