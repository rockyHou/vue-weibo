<template>
  <div class="home">
    <loading v-show="topIsLoading"></loading>
    <div class="content-tip no-text-select" v-show="noNew">
      <span>这会儿还没有新微博，等会再来刷刷看吧(｡･ω･｡)！</span>
    </div>
    <div class="top-tip" v-if="hasTopTip">
      <a class="to-top-tip">
        <i class="iconfont icon-hot"></i>
        <p class="top-tip-content txt-cut">{{topTip.text}}<i class="iconfont icon-right-arrow"></i></p>
      </a>
    </div>
    <div class="card" v-for="(item,index) in weiboContent.card_group">
      <header class="card-header">
        <!--<div class="header-bg" v-if="typeof item.mblog.cardid!=='undefined'">-->
        <!--&lt;!&ndash;使用 typeof的原因是它不会在一个变量没有被声明的时候抛出一个错误。&ndash;&gt;-->
        <!--</div>-->
        <a class="avatar" :href="item.mblog.user.profile_url">
          <div class="avatar-wrapper border-around-1px">
            <img class="avatar-img" :src="item.mblog.user.profile_image_url">
            <i class="iconfont" :class="calculateVerifiedClass(item.mblog.user.verified_type)"></i>
          </div>
        </a>
        <div class="user-info">
          <a :href="item.mblog.user.profile_url" class="user-name txt-l txt-cut">{{item.mblog.user.screen_name}}</a>
          <div class="publish-data txt-xs">
            <span class="publish-created-at">{{item.mblog.created_at}}</span>
            <span class="publish-source">来自{{item.mblog.source}}</span>
          </div>
        </div>
        <a class="card-operate">
          <i class="iconfont icon-down-arrow"></i>
        </a>
      </header>
      <section class="card-body">
        <p class="default-content" v-html="item.mblog.text"></p>
        <div v-if="item.mblog.pics.length===1" class="single-pic">
          <img :src="item.mblog.pics[0].url">
        </div>
        <ul v-if="item.mblog.pics.length>=2" class="pic-list">
          <li v-for="(eachPic,index) in item.mblog.pics" @click="openPicViewer(eachPic.url)"><img :src=eachPic.url></li>
        </ul>
        <div class="retweet" v-if="item.mblog.retweeted_status!==undefined">
          <p>
            <a :href="item.mblog.retweeted_status.user.profile_url"
               class="retweet-user">@{{item.mblog.retweeted_status.user.screen_name}}</a>：{{item.mblog.retweeted_status.text}}
          </p>
          <div v-if="item.mblog.retweeted_status.pics.length===1" class="single-pic">
            <img :src="item.mblog.retweeted_status.pics[0].url"
                 @click="openPicViewer(item.mblog.retweeted_status.pics[0].url)">
          </div>
          <ul v-if="item.mblog.retweeted_status.pics.length>=2" class="pic-list">
            <li v-for="(eachPic,index) in item.mblog.retweeted_status.pics"><img :src=eachPic.url></li>
          </ul>
        </div>
      </section>
      <footer class="card-footer border-1px border-top-1px txt-s no-text-select">
        <a class="forward">
          <i class="iconfont icon-forward"></i>
          {{item.mblog.reposts_count}}
        </a>
        <i class="separate-line"></i>
        <a class="comment">
          <i class="iconfont icon-comment"></i>
          {{item.mblog.comments_count}}
        </a>
        <i class="separate-line"></i>
        <a class="like" @click.prevent="likeIt($event, item)" :class="{'liked': item.mblog.favorited===true}">
          <i class="iconfont icon-like"></i>
          {{item.mblog.attitudes_count}}
        </a>
      </footer>
    </div>
    <transition name="like"
                v-on:before-enter="beforeLikeEnter"
                v-on:enter="likeEnter"
                v-on:after-enter="afterLikeEnter">
      <div class="like-animation-wrapper" v-show="showLikeAnimationWrapper">
        <i class="iconfont icon-like"></i>
      </div>
    </transition>
    <loading v-show="bottomIsLoading"></loading>
    <div class="content-tip no-text-select" v-show="noMore" @click="updateContent()">
      <span>没有更多微博了QAQ，点我刷新看看吧！</span>
      <a class="iconfont icon-refresh"></a>
    </div>
  </div>
</template>

<script>
  import loading from '../../components/loading/loading.vue'

  export default {
    name: 'home',
    data(){
      return {
        //要使用实例的属性，需要在这初始化声明
        topTip: {},
        weiboContent: {},
        hasTopTip: false,
        showPicViewer: this.$store.state.switchPicViewer,
        pagePos: 0,
        topIsLoading: true,
        bottomIsLoading: false,
        noMore: false,
        noNew: false,
        showLikeAnimationWrapper: false,
        clickedLikeBtnPos: {
          pageX: 0,
          pageY: 0
        }
      }
    },
    components: {
      loading
    },
    /*此处也可以在mounted之中用$nextTick调用methods的方法，来初始化weiboContent。详见 http://dwz.cn/65ocqi
     * 但我个人结合生命周期图认为，created早于mounted，用于初始化视图，应该首选created！*/
    created() {
      /*https://segmentfault.com/q/1010000006915580
       * 放在目录里并不意味能通过http访问。
       * 要么另起web服务serve这个目录，要么放在static目录里，因为dev-server对该目录文件实现了http访问。*/
      // this.$http.get('static/data/weibo-content.json', {id: 0}).then(res => {
      //等价于：
      this.$http.get('apis/weibo-content?targetCursor=1', {id: 0}).then(res => {
        /*res.body.data和res.data.data，哪一个才是最佳实践？*/
        if (res.body.errorNum !== 0) {
          console.log('Get data error!')
          return;
        }
        this.weiboContent = res.data.data  //微博的所有内容
//        console.log('this.weiboContent:', this.weiboContent)
//        console.log('this.weiboContent.previous_cursor:', this.weiboContent.previous_cursor)
//        console.log('this.weiboContent.next_cursor:', this.weiboContent.next_cursor)
        let tempTopTip = res.data.data.card_group[0]
        if (tempTopTip.mod_type === 'mod/clientTopTips') {
          this.topTip = res.data.data.card_group.shift() //公告是card_group[0]，用shift方法弹出
          this.hasTopTip = true
//          console.log('this.topTip:', this.topTip)
        }
        //主要内容加载完成后，才开始显示底部加载动画：
        this.bottomIsLoading = true
        setTimeout(() => {
          //故意推迟，以显示加载动画效果
          this.topIsLoading = false
        }, 1000)
      })
//        console.log('card_group:', this.weiboContent.card_group)
//        console.log('mblog:', this.weiboContent.card_group[2].mblog)
//        console.log('card_group[2].cardid : ', this.weiboContent.card_group[2].mblog.cardid)
//        console.log('pics[0].url:', this.weiboContent.card_group[2].mblog.pics[0].url)

      this.addScrollEvent()
    },
    methods: {
      calculateVerifiedClass: function (verifiedType) {
        let tempOutcome = ''
        switch (verifiedType) {
          case -1:
            tempOutcome = 'no-verified'
            break
          case 0:
            tempOutcome = 'icon-yellow-v'
            break
          case 1:
            tempOutcome = 'icon-blue-v'
            break
        }
        //console.log('verifiedType : ' + tempOutcome)
        return tempOutcome
      },
      openPicViewer(targetPicUrl) {
//        console.log('targetPicUrl = ' + targetPicUrl)
//        console.log('openPicViewer in Home.')
        this.$store.commit('openPicViewer', {targetPicUrl: targetPicUrl})
      },
      addScrollEvent() {
        /*可以通过以下两句分别对比使用事件防抖前后的效果：*/
//      window.addEventListener('scroll', this.handleScroll)
        window.addEventListener('scroll', this.myDebounce(this.handleScroll, 500))
        console.log('Scroll event added!')
      },
      handleScroll() {
        //console.log('scrolling...', window.scrollY)
        // Either scroll[Width/Height] or offset[Width/Height], whichever is greater
        let pageHeight = Math.max(
          document.documentElement.clientHeight,
          document.body.scrollHeight, document.documentElement.scrollHeight,
          document.body.offsetHeight, document.documentElement.offsetHeight)
        let windowScrollHeight = window.scrollY || window.pageYOffset
//        console.log('window.scrollY = ', window.scrollY)
//        console.log('window.pageYOffset  = ', window.pageYOffset)
        if (windowScrollHeight + window.innerHeight > pageHeight - 100) {
          this.hideAppAddTip()
          console.log('To push content')
          this.getContent()
        }
      },
      getContent() {
        /*procedure:
         1.get the next next_cursor from weiboContent.
         2.use as query attach to url.
         3.judge whether or not has new weibo.
         4.if has new weibo , invoke ajax by vue-resource.*/
        let nextCursor = this.weiboContent.next_cursor
        if (nextCursor !== -1) {
          let targetUrl = '/apis/weibo-content' + '?targetCursor=' + nextCursor
          this.$http.get(targetUrl).then((res) => {
//            console.log('targetUrl = ' + targetUrl)
            if (res.body.errorNum === 0) {
//              console.log('res.data.data.card_group = ', res.data.data.card_group)
              //把已有的微博和加载的旧微博合并
              this.weiboContent.card_group = this.weiboContent.card_group.concat(res.data.data.card_group)
              //只更新下一个目标的指向，避免影响前一个目标的加载
              this.weiboContent.next_cursor = res.data.data.next_cursor
            } else {
              console.log('Get data error!')
            }
          })
        } else {
          console.log('targetCursor === -1 ,No new content.')
          this.bottomIsLoading = false
          this.noMore = true
        }
      },
      updateContent() {
        this.hideAppAddTip()
        this.scrollToTop()
        let previousCursor = this.weiboContent.previous_cursor
        this.topIsLoading = true
        if (previousCursor !== -1) {
          let targetUrl = '/apis/weibo-content' + '?targetCursor=' + previousCursor
          this.$http.get(targetUrl).then((res) => {
//          console.log('targetUrl = ' + targetUrl)
            if (res.body.errorNum === 0) {
              console.log('res.data.data.card_group = ', res.data.data.card_group)
              //把新的微博和已有的微博合并
              let _this = this
              this.weiboContent.card_group = res.data.data.card_group.concat(_this.weiboContent.card_group)
              //只更新前一个目标的指向，避免影响下一个目标的加载
              this.weiboContent.previous_cursor = res.data.data.previous_cursor
              //故意延迟消失，以显示效果
              setTimeout(() => {
                this.topIsLoading = false
              }, 500)
            } else {
              console.log('Get data error!')
            }
          })
        } else {
          console.log('targetCursor === -1 ,No new content.')
          setTimeout(() => {
            this.topIsLoading = false
            this.noNew = true
          }, 500)
          //3s后，隐藏没有新微博的提示
          setTimeout(() => {
            this.noNew = false
          }, 3000)
        }
      },
      scrollToTop() {
        window.scrollTo(0, 0)
      },
      myDebounce(func, wait) {
//        console.log('set my debounce')
        let timeout;
        return function () {
          clearTimeout(timeout)
          timeout = setTimeout(func, wait)
          /*一个事件发生wait毫秒后，不再触发该事件，才执行相应的处理函数。*/
        }
      },
      likeIt(e, item) {
        //获取点击位置，用于设置动画的起始位置：
        console.log('clickedLikeBtnPos = ', e)
        this.clickedLikeBtnPos.pageX = e.pageX - parseInt(window.scrollX)
        this.clickedLikeBtnPos.pageY = e.pageY - parseInt(window.scrollY)
        console.log('this.clickedLikeBtnPos.pageY = ', this.clickedLikeBtnPos.pageY)
        if (item.mblog.favorited === false) {
          //显示点赞动画：
          this.showLikeAnimationWrapper = true
          this.$set(item.mblog, 'attitudes_count', item.mblog.attitudes_count + 1)
          this.$set(item.mblog, 'favorited', true)
        } else {
          this.$set(item.mblog, 'attitudes_count', item.mblog.attitudes_count - 1)
          this.$set(item.mblog, 'favorited', false)
        }
      },
      beforeLikeEnter(el) {
        /*在动画块呈现之前，，将其位置设置到点赞的位置上：*/
        el.style.transform = 'scale(0.1)'
        el.style.top = this.clickedLikeBtnPos.pageY + 'px'
        el.style.left = this.clickedLikeBtnPos.pageX + 'px'
        console.log('beforeLikeEnter : this.clickedLikeBtnPos.pageY = ', this.clickedLikeBtnPos.pageY)
      },
      likeEnter(el, done) {
        /* eslint-disable no-unused-vars*/
        /*很神奇！必须要触发一次重绘才能实现位置移动的过渡效果？？？
        * 正在查资料搞清楚这个问题！*/
        let rf = el.offsetHeight
        el.style.transform = 'scale(1)'
        el.style.top = '190px'
        el.style.left = '50%'
        done()
      },
      afterLikeEnter(el) {
        //动画结束后，隐藏该动画块（需要enter钩子中调用done()）：
        /*一个坑：enter钩子中调用了done()的话，动画效果就没有了？？？
        * 这和css中动画声明的class有关。*/
        this.showLikeAnimationWrapper = false
        console.log('动画结束后，隐藏该动画块.')
      },
      hideAppAddTip() {
        this.$emit('hideAppAddTip')
//        alert('emit hideAppAddTip')
      }
    }
  }
</script>

<style scoped lang="stylus">
.top-tip
  width 100%
  background-color #fed
  .to-top-tip
    height 2.75em
    display: flex
    align-items: center
    padding-left .75rem
    .icon-hot
      color: #ff8200
      margin-top -5px
      font-size: 22px
    .top-tip-content
      font-size: .875rem
      color: #FF8200;
      padding 0 .6875rem
      line-height: 1.5rem
      .iconfont
        margin-left: .5rem
        font-size: 0.775rem

//iconfont没选好，文字对不齐

// BUG!!会和footer的active覆盖！
/*.card:active*/
  /*background-color #ebebeb*/

.card-header
  display: flex
  .header-bg
    background-image url("../../../static/img/card-header-bg-headline.png")
    width: 100%
    height: 60px
    background-repeat: no-repeat
    background-position: top right
    background-size: 100% auto
    position: absolute
    top -4px
    left 0
  .avatar
    margin .75rem 0 .5rem .75rem
    display flex
    position relative
    .avatar-img
      width 2.125rem
      height 2.125rem
      border-radius 50%
      vertical-align top
      display block
    .no-verified
      display none
    .icon-yellow-v,.icon-blue-v
      position: absolute
      right: -.125rem
      bottom: -.125rem

.user-info
  max-width 16rem /*避免名字过长*/
  display: flex
  justify-content center
  flex-direction column
  padding: .6875rem .6875rem 0
  line-height: 1rem
  .user-name
    color #333
  .publish-data
    color #929292
    .publish-source
      padding-left .5rem

.card-operate
  position: absolute
  top: 0
  right: 0
  width: 3.75rem
  height: 3.375rem
  &:active
    background-color #ebebeb
  .icon-down-arrow
    position: absolute
    top .5rem
    right .5rem


.card-body
  padding: .25rem .75rem .625rem
  line-height 1.35rem
.pic-list
  margin-top .4375rem
  width: 15.5rem
  overflow hidden
  & li
    float: left
  & img
    width 4.75rem
    height 4.75rem
    margin-right .25rem

.single-pic
  margin-top .3125rem
  max-height 12.5rem
  overflow hidden
  text-align left
  & img
    width: 11.25rem
    vertical-align top

.retweet
  padding: .625rem
  margin .4375rem -.75rem -.75rem
  background-color #f7f7f7

.card-footer
  width: 100%
  display: flex
  align-items center
  background-image: -webkit-gradient(linear,left top,left bottom,color-stop(0%,#dadada),color-stop(50%,#dadada),color-stop(50%,rgba(0, 0, 0, 0)))
  background-image: -webkit-linear-gradient(top,#dadada 0,#dadada 50%,rgba(0, 0, 0, 0) 50%)
  background-image: linear-gradient(to bottom, #dadada 0, #dadada 50%, rgba(0, 0, 0, 0) 50%)
  /*从上到下，在1px的高度之中，从#dadada开始，直到0.5px时，将剩余的下半部分0.5px设置为透明，
  从而实现视觉上的0.5px。*/
  -webkit-background-size: 100% 1px
  background-size: 100% 1px
  background-repeat: no-repeat
  background-position: top
  & > a
    color #929292
    flex:1
    text-align center
    padding: .375rem 0
    display: inline-block
    height: 1.5rem
    line-height: 1.5rem
    &:active
      background-color #ebebeb


.liked
  color red !important

.like-animation-wrapper
  width: 50px
  height 50px
  line-height 50px
  text-align center
  color: #ff8200
  background-color: #f5f5f5
  border: 1px solid #eee
  border-radius 50%
  position: fixed
  /*top 90px*/
  /*left 50%*/
  /*水平居中成问题啊！*/
  transition all .3s linear
  margin-left -25px
  .icon-like
    font-size 32px

.content-tip
  width 100%
  text-align center
  min-height 50px
  margin-bottom .5625rem
  &:active
    background-color #ebebeb
  span
    display inline-block
    font-size .75rem
    line-height 19px
    color #7c7c7c
    margin 14px 0

.like-enter-to
  animation: like .8s .4s
/*动画性能有待优化，看起来帧数有些低？！*/

/*.like-enter-active*/
/*animation: like .4s .4s*/

@keyframes like {
  0% {
    transform: rotate(30deg)
  }
  20% {
    transform: rotate(-30deg)
  }
  40% {
    transform: rotate(30deg)
  }
  60% {
    transform: rotate(0deg)
  }
}
</style>
