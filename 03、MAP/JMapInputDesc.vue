<template>
  <div>
    <a-modal
      v-model="mapVisible"
      @ok="handleOk"
      @cancel="handleCancel"
      switchFullscreen
      cancelText="取消"
      title="区域设置"
      id="mapDialog"
      width="80%"
      top="8vh"
    >
      <div class="nav">
        <template v-if="this.polygonShow ? !this.isShow : this.isShow">
          <div class="flexbutton">
            <div>
              <a-button type="primary" @click="toggle('polygonPath')">
                {{ polygonPath.editing ? '停止绘制' : '开始绘制' }}
              </a-button>
              <a-button type="primary" @click="polygonEdit"> 编辑 </a-button>
              <el-link type="info" disabled
                >点击鼠标左键在地图上开始绘制，点击鼠标右键绘制结束，您还可以进行拖动修改哦！</el-link
              >
            </div>
          </div>
        </template>
        <template v-else-if="this.circleShow ? !this.isShow : this.isShow">
          经度：<el-input v-model="circlePath.center.lng" placeholder="请输入经度"></el-input> 纬度：<el-input
            v-model="circlePath.center.lat"
            placeholder="请输入纬度"
          ></el-input>
          半径：<el-input v-model="circlePath.radius" placeholder="请输入半径"></el-input>
          <a-button type="primary" @click="isOK">确定</a-button>
          <a-button type="primary" @click="circleEdit"> 编辑 </a-button>
        </template>
        <baidu-map
          class="map"
          :center="center"
          :zoom="zoom"
          :map-click="false"
          @mousemove="syncPolygon"
          @ready="handler"
          style="height: 600px"
          @click="paintPolygon"
          @rightclick="newPolygon"
        >
          <!-- 地图控件位置 -->
          <bm-navigation anchor="BMAP_ANCHOR_BOTTOM_RIGHT"></bm-navigation>
          <!-- 城市列表 -->
          <bm-city-list anchor="BMAP_ANCHOR_TOP_LEFT"></bm-city-list>
          <!-- 地图类型 -->
          <bm-map-type :map-types="['BMAP_NORMAL_MAP', 'BMAP_HYBRID_MAP']" anchor="BMAP_ANCHOR_TOP_RIGHT"></bm-map-type>
          <!-- 多边形 -->
          <template v-if="this.polygonShow ? !this.isShow : this.isShow">
            <bm-polygon
              :path="path"
              v-for="path of polygonPath.paths"
              :key="path.toString()"
              stroke-color="blue"
              fill-color="red"
              :fill-opacity="0.5"
              :stroke-opacity="0.5"
              :stroke-weight="2"
              :editing="true"
              @lineupdate="updatePolygonPath"
            />
          </template>
          <!-- 圆形 -->
          <template v-else-if="this.circleShow ? !this.isShow : this.isShow">
            <bm-circle
              :center="circlePath.center"
              :radius="circlePath.radius"
              stroke-color="blue"
              fill-color="red"
              :fill-opacity="0.5"
              :stroke-opacity="0.5"
              :stroke-weight="2"
              @lineupdate="updateCirclePath"
              :editing="true"
            ></bm-circle>
          </template>
        </baidu-map>
      </div>
    </a-modal>
  </div>
</template>

<script>

export default {
  name: "JMapInputDesc",
  props: {
    circleShow: {
      type: Boolean
    },
    polygonShow: {
      type: Boolean
    },
    polygon: {
      type: Array
    },
    circle:{
      type:Object
    }
  },
  data () {
    return {
      mapVisible: false,
      isShow: false,
      editing: false,
      center: { lng: 103.063035, lat: 31.672767 },
      zoom: 10,
      map: null,
      polygonPath: {
        editing: false,
        paths: []
      },
      circlePath: {
        editing: false,
        center: {
          lng: 103.404,
          lat: 31.915
        },
        radius: 5000
      }
    }
  },
  methods: {
    polygonEdit () {
      this.polygonPath.paths = []
      var newpolygon = this.polygon ? this.polygon : []
      if (newpolygon === []) {
        return false
      } else {
        var paths = []
        newpolygon.forEach(item => {
          paths.push({
            lng: item[0],
            lat: item[1]
          })
        });
        this.polygonPath.paths.push(paths)
      }
    },
    circleEdit () {
      const newcircle = this.circle ? this.circle : {}
      if (newcircle === {}) {
        return false
      } else {
        this.circlePath.radius = newcircle.radius
        this.circlePath.center = newcircle.center
      }
    },
    // 开启多边形绘制
    toggle (name) {
      this.$message({
        showClose: true,
        message: '点击鼠标左键在地图上开始绘制，点击鼠标右键绘制结束',
        type: 'success',
        duration: 5000
      });
      this[name].editing = !this[name].editing
      // 在这里做一步判断，如果有路径且开启绘制就把原来的路径清空
      if (this.polygonPath.paths && this.polygonPath.editing) {
        this.polygonPath.paths = []
      }
    },
    // 鼠标移动时
    syncPolygon (e) {
      if (!this.polygonPath.editing) {
        return
      }
      const { paths } = this.polygonPath
      if (!paths.length) {
        return
      }
      const path = paths[paths.length - 1]
      if (!path.length) {
        return
      }
      if (path.length === 1) {
        path.push(e.point)
      }
      this.$set(path, path.length - 1, e.point)
    },
    // 鼠标右键点击时往路径里push一个面
    newPolygon (e) {
      if (!this.polygonPath.editing) {
        return
      }
      // 当开始绘制后把按钮调回开始绘制状态，防止绘制多个图形
      this['polygonPath'].editing = !this['polygonPath'].editing
      const { paths } = this.polygonPath
      if (!paths.length) {
        paths.push([])
      }
      const path = paths[paths.length - 1]
      path.pop()
      if (path.length) {
        paths.push([])
      }
    },
    // 鼠标左键多边形绘制
    paintPolygon (e) {
      if (!this.polygonPath.editing) {
        return
      }
      const { paths } = this.polygonPath
      !paths.length && paths.push([])
      paths[paths.length - 1].push(e.point)
    },
    updatePolygonPath (e) {
      //编辑覆盖物时触发，获取坐标点集合	
      this.polygonPath.paths[0] = e.target.getPath()
    },
    updateCirclePath (e) {
      this.circlePath.center = e.target.getCenter()
      this.circlePath.radius = e.target.getRadius()
    },
    handler ({ BMap, map }) {
      map.enableScrollWheelZoom(true)
    },
    isOK () {
      this.$emit("mapLocationSave", this.circlePath)
      this.mapVisible = false;
    },
    handleOk () {
      var polygonpath = this.polygonPath.paths[0]
      var newarr = []
      for (let i = 0; i < polygonpath.length; i++) {
        newarr.push([polygonpath[i].lng, polygonpath[i].lat])
      }
      this.$emit("mapLocationSave", newarr)
      this.mapVisible = false;
    },
    show () {
      this.mapVisible = true;
    },
    handleCancel () {
      this.mapVisible = false;
    }
  },
}
</script>

<style scoped>
.el-input {
  width: 200px;
  margin-right: 20px;
}
.ant-btn {
  margin-bottom: 10px;
  margin-right: 10px;
}
.flexbutton {
  display: flex;
  justify-content: space-between;
}

.allmap {
  width: 100%;
  height: 100vh;
  position: relative;
  z-index: 1;
}
button {
  border: none;
}
.change-map-type {
  position: absolute;
  right: 50px;
  bottom: 10px;
  z-index: 2;
}

ul li {
  list-style: none;
}
.nav {
  position: relative;
}
/* .info {
  z-index: 999;
  width: auto;
  min-width: 22rem;
  padding: 0.75rem 1.25rem;
  margin-left: 1.25rem;
  position: fixed;
  top: 1rem;
  background-color: #fff;
  border-radius: 0.25rem;
  font-size: 14px;
  color: #666;
  box-shadow: 0 2px 6px 0 rgba(27, 142, 236, 0.5);
} */

.drawing-panel {
  z-index: 999;
  position: absolute;
  bottom: 40px;
  left: 20px;
  border-radius: 0.25rem;
  height: 47px;
  box-shadow: 0 2px 6px 0 rgba(27, 142, 236, 0.5);
}

.bmap-btn {
  border-right: 1px solid #d2d2d2;
  float: left;
  width: 64px;
  height: 100%;
  background-image: url(http://api.map.baidu.com/library/DrawingManager/1.4/src/bg_drawing_tool.png);
  cursor: pointer;
}

.drawing-panel .bmap-rectangle {
  background-position: -325px 0;
}

.drawing-panel .bmap-polygon {
  background-position: -260px 0;
}

.drawing-panel .bmap-circle {
  background-position: -130px 0;
}
</style>
