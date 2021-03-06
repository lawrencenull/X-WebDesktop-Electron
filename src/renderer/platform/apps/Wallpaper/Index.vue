/**
* Created by OXOYO on 2017/11/17.
*
* 桌面壁纸
*/

<style scoped lang="less" rel="stylesheet/less">
  .wallpaper {
    position: absolute;
    top: 0;
    bottom: 0;
    left: 0;
    right: 0;

    .wallpaper-images {
      width: 100%;
      height: 100%;
      transition: background-image 1s ease-in-out 0s;

      img {
        width: 100%;
        height: 100%;
      }
    }
    .wallpaper-components {
      width: 100%;
      height: 100%;
    }
  }
</style>

<template>
  <div class="wallpaper">
    <transition enter-active-class="animated fadeIn" leave-active-class="animated fadeOut">
      <!-- 图片背景 -->
      <div class="wallpaper-images" v-if="currentWallpaper.type === 'images'" :style="currentWallpaper.style">
      </div>
      <!-- 组件背景 -->
      <div class="wallpaper-components" v-if="currentWallpaper.type === 'components'">
        <component :is="currentWallpaper.src"></component>
      </div>
    </transition>
  </div>
</template>

<script>
  import { mapState } from 'vuex'

  export default {
    name: 'Wallpaper',
    data () {
      return {
        wallpaperList: {
          // 图片壁纸路径列表
          // FIXME 列表应该通过用户配置获得
          images: [
            // './static/wallpaper/bg_0001.jpg',
            // './static/wallpaper/bg_0002.jpg',
            // './static/wallpaper/bg_0003.jpg'
          ],
          // 组件壁纸 组件名称列表
          components: [
            'RainDay'
          ]
        },
        currentWallpaper: {
          // 类别
          type: '',
          // 资源
          src: '',
          // 样式
          style: {}
        },
        // 切换方式: default || bing || all
        switchType: 'bing'
      }
    },
    computed: {
      ...mapState('Platform/Wallpaper', {
        wallpaperInfo: state => state.currentWallpaper
      })
    },
    watch: {
      'switch': function (val) {
        let _t = this
        if (val) {
          // 执行切换
          _t.doSwitch()
        }
      }
    },
    methods: {
      init: function () {
        let _t = this
        // 切换壁纸
        _t.doSwitch()
      },
      doSwitch: function () {
        let _t = this
        switch (_t.switchType) {
          case 'default':
            _t.doSwitchByDefault()
            break
          case 'bing':
            _t.doSwitchByBing()
            break
          case 'all':
            let typeIndex = Math.round(Math.random())
            if (typeIndex) {
              _t.doSwitchByBing()
            } else {
              _t.doSwitchByDefault()
            }
            break
        }
      },
      // 执行默认壁纸
      doSwitchByDefault: function () {
        let _t = this
        // 1.随机壁纸类型
        let typeIndex = Math.round(Math.random())
        // FIXME 由于RainDay非常耗CPU，将typeIndex改为0
        typeIndex = 0
        let type = Object.keys(_t.wallpaperList)[typeIndex]
        if (_t.wallpaperList[type].length) {
          // 1.防止该类型下数据为为空的清空
          // 2.随机壁纸资源序号
          let srcIndex = Math.floor(Math.random() * _t.wallpaperList[type].length)
          let src = _t.wallpaperList[type][srcIndex]
          // 4.处理样式
          if (type === 'images') {
            _t.lazyLoadImg(src)
          } else {
            // 3.更新当前壁纸数据
            _t.currentWallpaper = {
              ..._t.currentWallpaper,
              type: type,
              src: _t.wallpaperList[type][srcIndex],
              style: {}
            }
            // 分发mutations，更新当前的壁纸信息
            _t.$store.commit(_t.$utils.store.getType('Wallpaper/currentWallpaper/update', 'Platform'), {
              ..._t.currentWallpaper
            })
          }
        } else {
          _t.doSwitchByBing()
        }
      },
      // 切换bing壁纸
      doSwitchByBing: async function () {
        let _t = this
        // 1.获取bing壁纸
        let src = await _t.getBingWallpaper()
        if (src) {
          // 渐进加载
          _t.lazyLoadImg(src)
        }
      },
      getBingWallpaper: async function () {
        let _t = this
        let dayIndex = Math.floor(Math.random() * 17)
        // 分发action，执行保存
        let res = await _t.$store.dispatch(_t.$utils.store.getType('Wallpaper/bing', 'Platform'), {
          format: 'js',
          // idx 可选值 -1 || 1 ~ 16
          idx: dayIndex,
          // n 最大为8
          n: 8
        })
        if (!res) {
          _t.$Message.error('获取壁纸失败！')
          return null
        } else if (res.status !== 200) {
          return null
        }
        // 处理返回数据
        if (res) {
          _t.$Message.success('获取壁纸成功！')
          try {
            // 解析壁纸数据
            let wallpaperData = res.data.images
            // 随机一条数据
            let srcIndex = Math.floor(Math.random() * wallpaperData.length)
            // 拼装地址
            let baseUrl = 'http://cn.bing.com'
            let wallpaperUrl = wallpaperData[srcIndex].url
            return baseUrl + wallpaperUrl
          } catch (err) {
            _t.doSwitchByDefault()
            return null
          }
        }
      },
      lazyLoadImg: function (src) {
        let _t = this
        // 渐进加载
        let img = new Image()
        img.src = src
        img.onload = function () {
          _t.currentWallpaper['type'] = 'images'
          _t.currentWallpaper['src'] = src
          // 处理样式
          _t.currentWallpaper['style'] = {
            background: 'url(' + src + ') center center / cover no-repeat fixed'
          }
          // 分发mutations，更新当前的壁纸信息
          _t.$store.commit(_t.$utils.store.getType('Wallpaper/currentWallpaper/update', 'Platform'), {
            ..._t.currentWallpaper
          })
        }
      }
    },
    created: function () {
      let _t = this
      // 判断是否组件创建时是否已存在壁纸信息，有则覆盖，无则初始化
      if (_t.wallpaperInfo && _t.wallpaperInfo.type && _t.wallpaperInfo.src) {
        _t.currentWallpaper = {
          ..._t.wallpaperInfo
        }
      } else {
        // 初始化
        _t.init()
      }
    },
    mounted: function () {
      let _t = this
      // 监听事件
      _t.$utils.bus.$on('platform/wallpaper/switch', function () {
        _t.doSwitch()
      })
    },
    beforeDestroy: function () {
      let _t = this
      _t.$utils.bus.$off([
        'platform/wallpaper/switch'
      ])
    }
  }
</script>
