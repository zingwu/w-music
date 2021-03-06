<template>
  <div class="player" v-show="playlist.length > 0">
    <transition name="normal">
      <div class="normal-player" v-show="fullScreen" @touchstart.once="firstPlay">
        <div class="background">
          <div class="filter"></div>
          <img :src="currentSong.image" width="100%" height="100%" />
        </div>
        <div class="top">
          <div class="back" @click="back">
            <i class="iconfont icon-down"></i>
          </div>
          <h1 class="title" v-html="currentSong.name"></h1>
          <h2 class="subtitle" v-html="currentSong.singer"></h2>
        </div>
        <div class="middle" @click="changeMiddle">
          <transition name="middleL">
            <div class="middle-l" v-show="currentShow === 'cd'">
              <div class="cd-wrapper">
                <div class="cd" :class="cdCls">
                  <img :src="currentSong.image" class="image" />
                </div>
              </div>
            </div>
          </transition>
          <transition name="middleR">
            <scroll
              class="middle-r"
              ref="lyricList"
              v-show="currentShow === 'lyric'"
              :data="currentLyric && currentLyric.lines"
            >
              <div class="lyric-wrapper">
                <div class="currentLyric" v-if="currentLyric">
                  <p
                    ref="lyricLine"
                    class="text"
                    :class="{'current': currentLineNum === index}"
                    v-for="(line, index) in currentLyric.lines"
                    :key="line.key"
                  >{{line.txt}}</p>
                </div>
                <p class="no-lyric" v-if="currentLyric === null">{{upDatecurrentLyric}}</p>
              </div>
            </scroll>
          </transition>
        </div>
        <div class="bottom">
          <div class="progress-wrapper">
            <span class="time time-l">{{format(currentTime)}}</span>
            <div class="progress-bar-wrapper">
              <progress-bar
                :percent="percent"
                @percentChangeEnd="percentChangeEnd"
                @percentChange="percentChange"
              ></progress-bar>
            </div>
            <span class="time time-r">{{format(duration)}}</span>
          </div>
          <div class="operators">
            <div class="icon i-left">
              <i class="iconfont mode" :class="iconMode" @click="changeMode"></i>
            </div>
            <div class="icon i-left">
              <i class="iconfont icon-previous" @click="prev"></i>
            </div>
            <div class="icon i-center">
              <i class="iconfont icon-play" @click="togglePlaying" :class="playIcon"></i>
            </div>
            <div class="icon i-right">
              <i class="iconfont icon-next" @click="next"></i>
            </div>
            <div class="icon i-right">
              <i
                class="iconfont"
                @click="toggleFavorite(currentSong)"
                :class="getFavoriteIcon(currentSong)"
              ></i>
            </div>
          </div>
        </div>
      </div>
    </transition>
    <transition name="mini">
      <div class="mini-player" v-show="!fullScreen" @click.stop="open">
        <div class="icon">
          <img :class="cdCls" :src="currentSong.image" width="40" height="40" />
        </div>
        <div class="text">
          <h2 class="name" v-html="currentSong.name"></h2>
          <div class="desc" v-html="currentSong.singer"></div>
        </div>
         <!-- @click.stop阻止togglePlaying这个点击事件冒泡到父元素 -->
        <div class="control" @click.stop="togglePlaying">
          <i class="iconfont icon-play" :class="miniIcon"></i>
        </div>
        <div class="control" @click.stop="showPlaylist">
          <i class="iconfont icon-menu"></i>
        </div>
      </div>
    </transition>
    <playlist @stopMusic="stopMusic" ref="playlist"></playlist>
    <audio
      id="music-audio"
      ref="audio"
      @ended="end"
      autoplay
      @canplay="ready"
      @error="error"
      @timeupdate="updateTime"
    ></audio>
  </div>
</template>

<script>
import Lyric from 'lyric-parser'
import Scroll from 'base/scroll/scroll'
import ProgressBar from 'base/progress-bar/progress-bar'
import Playlist from 'components/playlist/playlist'
import { mapGetters, mapMutations, mapActions } from 'vuex'
import { getSong, getLyric } from 'api/song'
import { playMode } from 'common/js/config'
import { shuffle } from 'common/js/utl'

export default {
  data() {
    return {
      url: '',
      songReady: false,
      currentTime: 0,
      duration: 0,
      percent: 0,
      radius: 32,
      currentLyric: null,
      currentLineNum: 0,
      currentShow: 'cd',
      playingLyric: '',
      noLyric: false
    }
  },
  created() {
    this.move = false
  },
  computed: {
    // 切换播放模式图标
    iconMode() {
      if (this.mode === playMode.sequence) {
        return 'icon-listloop'
      } else if (this.mode === playMode.loop) {
        return 'icon-singlecycle'
      } else {
        return 'icon-random'
      }
    },
    cdCls() {
      return this.playing ? 'play' : 'play pause'
    },
    miniIcon() {
      return this.playing ? 'icon-pause' : 'icon-play'
    },
    // 切换播放暂停图标
    playIcon() {
      return this.playing ? 'icon-pause' : 'icon-play'
    },
    upDatecurrentLyric() {
      if (this.noLyric) {
        return '暂无歌词'
      }
      if (!this.noLyric) {
        return '歌词加载中'
      }
    },
    ...mapGetters([
      'playlist',
      'fullScreen',
      'currentSong',
      'playing',
      'currentIndex',
      'mode',
      'sequenceList',
      'favoriteList'
    ])
  },
  watch: {
    currentSong(newVal, oldVal) {
      // 歌曲id不变，就不切换暂停与播放
      if (!newVal.id) {
        return
      }
      if (newVal.id === oldVal.id) {
        return
      }
      this.$refs.audio.pause()
      this.$refs.audio.currentTime = 0
      this._getSong(newVal.id)
    },
    url(newUrl) {
      this._getLyric(this.currentSong.id)
      this.$refs.audio.src = newUrl
      let stop = setInterval(() => {
        this.duration = this.$refs.audio.duration
        if (this.duration) {
          clearInterval(stop)
        }
      }, 150)
      this.setPlayingState(true)
    },
    currentTime() {
      this.percent = this.currentTime / this.duration
    }
  },
  methods: {
    firstPlay() {
      this.$refs.audio.play()
    },
    stopMusic() {
      // 删除最后一首的时候暂停音乐
      this.$refs.audio.pause()
    },
    showPlaylist() {
      this.$refs.playlist.show()
    },
    changeMiddle() {
      if (this.currentShow === 'cd') {
        this.currentShow = 'lyric'
      } else {
        this.currentShow = 'cd'
      }
    },
    getFavoriteIcon(song) {
      if (this.isFavorite(song)) {
        return 'icon-like'
      }
      return 'icon-dislike'
    },
    toggleFavorite(song) {
      if (this.isFavorite(song)) {
        this.deleteFavoriteList(song)
      } else {
        this.saveFavoriteList(song)
      }
    },
    isFavorite(song) {
      const index = this.favoriteList.findIndex(item => {
        return item.id === song.id
      })
      return index > -1
    },
    changeMode() {
      const mode = (this.mode + 1) % 3 // 改变播放模式
      this.setPlayMode(mode)
      let list = null
      if (mode === playMode.random) {
        list = shuffle(this.sequenceList)
      } else {
        list = this.sequenceList
      }
      this._resetCurrentIndex(list)
      this.setPlaylist(list)
    },
    // 保证切换播放模式当前播放歌曲不变
    _resetCurrentIndex(list) {
      let index = list.findIndex(item => {
        return item.id === this.currentSong.id
      })
      this.setCurrentIndex(index)
    },
    percentChange(percent) {
      this.move = true
      const currentTime = this.duration * percent
      this.currentTime = currentTime
      if (this.currentLyric) {
        this.currentLyric.seek(currentTime * 1000)
      }
    },
    percentChangeEnd(percent) {
      this.move = false
      const currentTime = this.duration * percent
      this.$refs.audio.currentTime = currentTime
      if (!this.playing) {
        this.$refs.audio.play()
        this.setPlayingState(true)
      }
      if (this.currentLyric) {
        this.currentLyric.seek(currentTime * 1000)
      }
    },
    // 实时监测播放进度
    updateTime(e) {
      if (this.move) {
        return
      }
      this.currentTime = e.target.currentTime
    },
    // 更改歌曲播放时间格式
    format(interval) {
      interval = interval | 0
      let minute = (interval / 60) | 0 // 获得分钟数
      let second = interval % 60 // 获得秒数
      if (second < 10) {
        second = '0' + second
      }
      return minute + ':' + second
    },
    end() {
      if (this.mode === playMode.loop) {
        this.loop()
      } else {
        this.next()
      }
    },
    loop() {
      this.$refs.audio.currentTime = 0
      this.$refs.audio.play()
      if (this.currentLyric) {
        this.currentLyric.seek()
      }
    },
    error() {
      this.songReady = true
    },
    ready() {
      this.songReady = true
      this.savePlayHistory(this.currentSong)
    },
    next() {
      if (!this.songReady) {
        return
      }
      if (this.playlist.length === 1) {
        this.loop()
        return
      } else {
        let index = this.currentIndex + 1
        if (index === this.playlist.length) {
          index = 0
        }
        this.setCurrentIndex(index)
        if (!this.playing) {
          this.togglePlaying()
        }
      }
      this.songReady = false
    },
    prev() {
      if (!this.songReady) {
        return
      }
      this.songReady = false
      let index = this.currentIndex - 1
      if (index === -1) {
        index = this.playlist.length - 1
      }
      this.setCurrentIndex(index)
      if (!this.playing) {
        this.togglePlaying()
      }
      this.songReady = false
    },
    // 切换到mini播放器
    back() {
      this.setFullScreen(false)
      this.currentShow = 'cd'
    },
    // 切换到normal播放器
    open() {
      this.setFullScreen(true)
    },
    togglePlaying() {
      const audio = this.$refs.audio
      this.setPlayingState(!this.playing)
      this.playing ? audio.play() : audio.pause()
      if (this.currentLyric) {
        this.currentLyric.togglePlay()
      }
    },
    _getSong(id) {
      getSong(id).then(res => {
        this.url = res.data.data[0].url
      })
    },
    _getLyric(id) {
      if (this.currentLyric) {
        this.currentLyric.stop()
        this.currentLyric = null
      }
      this.noLyric = false
      getLyric(id)
        .then(res => {
          this.currentLyric = new Lyric(res.data.lrc.lyric, this.handleLyric)
          if (this.playing) {
            this.currentLyric.play()
            // 歌词重载以后 高亮行设置为 0
            this.currentLineNum = 0
            this.$refs.lyricList.scrollTo(0, 0, 1000)
          }
        })
        .catch(() => {
          this.currentLyric = null
          this.noLyric = true
          this.currentLineNum = 0
        })
    },
    handleLyric({ lineNum, txt }) {
      this.currentLineNum = lineNum
      if (lineNum > 5) {
        let lineEl = this.$refs.lyricLine[lineNum - 5]
        this.$refs.lyricList.scrollToElement(lineEl, 1000)
      } else {
        this.$refs.lyricList.scrollTo(0, 0, 1000)
      }
    },
    ...mapMutations({
      setFullScreen: 'SET_FULL_SCREEN',
      setPlayingState: 'SET_PLAYING_STATE',
      setCurrentIndex: 'SET_CURRENT_INDEX',
      setPlayMode: 'SET_PLAY_MODE',
      setPlaylist: 'SET_PLAYLIST'
    }),
    ...mapActions(['saveFavoriteList', 'deleteFavoriteList', 'savePlayHistory'])
  },
  components: {
    Scroll,
    Playlist,
    ProgressBar
  }
}
</script>

<style lang="scss" scoped>
@import "common/scss/variable.scss";
@import "common/scss/mixin.scss";
.player {
  .normal-player {
    position: fixed;
    left: 0;
    right: 0;
    top: 0;
    bottom: 0;
    z-index: 150;
    background: $color-background;
    .background {
      position: absolute;
      left: -50%;
      top: -50%;
      width: 300%;
      height: 300%;
      z-index: -1;
      opacity: 0.6;
      filter: blur(30px);
      .filter {
        position: absolute;
        width: 100%;
        height: 100%;
        background: black;
        opacity: 0.6;
      }
      .filterR {
        position: absolute;
        width: 100%;
        height: 100%;
        background: black;
        opacity: 0.4;
        &.filterR-enter-active,
        &.filterR-leave-active {
          transition: all 0.3s;
        }
        &.filterR-leave-to,
        &.filterR-enter {
          opacity: 0;
        }
        &.filterR-leave {
          opacity: 0.4;
        }
      }
    }
    .top {
      position: relative;
      margin-bottom: 25px;
      .back {
        position: absolute;
        top: 15px;
        left: 20px;
        z-index: 50;
        color: white;
        .fa-angle-down {
          display: block;
          padding: 5px 9px;
          font-size: 35px;
          color: $color-theme-l;
        }
      }
      .title {
        width: 70%;
        margin: 0 auto;
        padding-top: 10px;
        line-height: 20px;
        text-align: center;
        @include no-wrap();
        font-size: $font-size-large;
        font-weight: bold;
        color: $color-text-l;
      }
      .subtitle {
        width: 70%;
        margin: 0 auto;
        line-height: 20px;
        text-align: center;
        @include no-wrap();
        font-size: $font-size-small-x;
        color: $color-text-l;
      }
    }
    .middle {
      display: flex;
      align-items: center;
      position: fixed;
      width: 100%;
      top: 80px;
      bottom: 170px;
      white-space: nowrap;
      font-size: 0;
      .middle-l {
        display: inline-block;
        vertical-align: top;
        position: relative;
        width: 100%;
        height: 0;
        padding-top: 80%;
        &.middleL-enter-active,
        &.middleL-leave-active {
          transition: all 0.3s;
        }
        &.middleL-enter,
        &.middleL-leave-to {
          opacity: 0;
        }
        .cd-wrapper {
          position: absolute;
          left: 10%;
          top: 0;
          width: 80%;
          height: 100%;
          .cd {
            width: 100%;
            height: 100%;
            box-sizing: border-box;
            border: 15px solid rgba(255, 255, 255, 0.1);
            border-radius: 50%;
            &.play {
              animation: rotate 20s linear infinite;
            }
            &.pause {
              animation-play-state: paused;
            }
            .image {
              position: absolute;
              left: 0;
              top: 0;
              width: 100%;
              height: 100%;
              border-radius: 50%;
            }
          }
        }
      }
      .middle-r {
        display: inline-block;
        position: absolute;
        top: 0;
        vertical-align: top;
        width: 100%;
        height: 100%;
        overflow: hidden;
        &.middleR-enter-active,
        &.middleR-leave-active {
          transition: all 0.2s;
        }
        &.middleR-enter,
        &.middleR-leave-to {
          opacity: 0;
        }
        .lyric-wrapper {
          width: 80%;
          margin: 0 auto;
          overflow: hidden;
          text-align: center;
          .text {
            line-height: 40px;
            color: $color-text-ggg;
            font-size: $font-size-medium;
            &.current {
              color: #fff;
            }
          }
          .no-lyric {
            line-height: 40px;
            margin-top: 60%;
            color: $color-text-ggg;
            font-size: $font-size-medium;
          }
        }
      }
    }
    .bottom {
      position: absolute;
      bottom: 50px;
      width: 100%;
      .progress-wrapper {
        display: flex;
        align-items: center;
        width: 80%;
        margin: 0px auto;
        padding: 10px 0;
        .time {
          color: $color-text-l;
          font-size: $font-size-small;
          flex: 0 0 30px;
          line-height: 30px;
          width: 30px;
          &.time-l {
            text-align: left;
          }
          &.time-r {
            text-align: right;
            color: $color-text-gg;
          }
        }
        .progress-bar-wrapper {
          flex: 1;
        }
      }
      .operators {
        display: flex;
        align-items: center;
        .icon {
          flex: 1;
          color: $color-theme-l;
          &.disable {
            color: $color-theme;
          }
          i {
            font-size: 30px;
          }
          .mode {
            font-size: 30px;
          }
          &.i-left {
            text-align: right;
          }
          &.i-center {
            padding: 0 20px;
            text-align: center;
            i {
              font-size: 40px;
            }
          }
          &.i-right {
            text-align: left;
          }
          .icon-like {
            color: $color-sub-theme;
          }
        }
      }
    }
    &.normal-enter-active,
    &.normal-leave-active {
      transition: all 0.4s;
      .top,
      .bottom {
        transition: all 0.4s cubic-bezier(0.86, 0.18, 0.82, 1.32);
      }
    }
    &.normal-enter,
    &.normal-leave-to {
      opacity: 0;
    }
  }
  .mini-player {
    display: flex;
    align-items: center;
    position: fixed;
    left: 0;
    bottom: 0;
    z-index: 180;
    width: 100%;
    height: 60px;
    background: rgba(255, 255, 255, 0.85);
    &.mini-enter-active,
    &.mini-leave-active {
      transition: all 0.4s;
    }
    &.mini-enter,
    &.mini-leave-to {
      opacity: 0;
    }
    .icon {
      flex: 0 0 40px;
      width: 40px;
      padding: 0 10px 0 20px;
      img {
        border-radius: 50%;
        &.play {
          animation: rotate 10s linear infinite;
        }
        &.pause {
          animation-play-state: paused;
        }
      }
    }
    .text {
      display: flex;
      flex-direction: column;
      justify-content: center;
      flex: 1;
      overflow: hidden;
      .name {
        margin-bottom: 2px;
        line-height: 14px;
        @include no-wrap();
        font-size: $font-size-medium;
        color: $color-text;
      }
      .desc {
        @include no-wrap();
        font-size: $font-size-small;
        color: $color-text;
      }
    }
    .control {
      flex: 0 0 30px;
      width: 30px;
      padding: 0 10px;
      .icon-play-mini,
      .icon-pause-mini,
      .icon-playlist,
      .iconfont {
        font-size: 30px;
        color: $color-theme-d;
      }
      .iconfont {
        position: relative;
        left: -5px;
        top: -3px;
      }
      .fa-play {
        color: $color-theme-d;
        font-size: 35px;
        position: absolute;
        left: 227px;
        top: 8.5px;
      }
      .fa-stop {
        color: $color-theme-d;
        font-size: 35px;
        position: absolute;
        left: 227px;
        top: 8.5px;
      }
    }
  }
}
@keyframes rotate {
  0% {
    transform: rotate(0);
  }
  100% {
    transform: rotate(360deg);
  }
}
</style>
