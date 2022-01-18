<template>
  <div class="time-select">
    <table ref="table">
      <tr>
        <td v-for="(item, index) in 24" :key="index" colspan="2">{{ index }}</td>
      </tr>
      <tr>
        <td
          v-for="(t, tindex) in 48"
          :key="tindex"
          class="time"
          :class="{ active: selected[tindex] === 1 }"
          @click="handleClick(tindex)"
        ></td>
      </tr>
    </table>
    <div class="options">
      <span class="span-btn" style="color: #999999">{{ isSelected ? "已选择时间段" : "可拖动鼠标选择时间段" }}</span>
      <span
        v-if="isSelected && !disabled"
        class="span-btn"
        style="position: absolute; right: 0"
        @click="removeAll"
      >清空选择</span>
    </div>
    <div v-if="isSelected" class="tips">
      <span>{{ selectTimes.map((o) => o.beginTime + "~" + o.endTime).join("、") }}</span>
    </div>
  </div>
</template>

<script>
export default {
  name: 'TimePeriodPicker',
  model: {
    prop: 'value',
    event: 'change'
  },
  props: {
    value: {
      type: Array,
      required: true,
      default: () => []
    },
    disabled: {
      type: Boolean,
      default: false
    }
  },
  data() {
    return {
      ismoving: false,
      startX: 0,
      startY: 0,
      endX: 0,
      endY: 0,
      tablePosition: {
        top: 0,
        right: 0,
        bottom: 0,
        left: 0
      },
      marker: document.createElement('div'),
      selected: new Array(48).fill(0)
    }
  },
  computed: {
    selectTimes() {
      // 未选中时间段
      if (!this.selected.includes(1)) {
        return []
      }
      const rawTimeList = []
      const resTimeList = []
      // 将selected数组中为1的转为时间对象装入rawTimeList中
      this.selected.forEach((item, index) => {
        item && rawTimeList.push(this.getTimeObj(index))
      })
      rawTimeList.forEach(({ beginTime, endTime }) => {
        if (resTimeList.length === 0) {
          resTimeList.push({ beginTime, endTime })
        } else if (beginTime === resTimeList[resTimeList.length - 1].endTime) {
          resTimeList[resTimeList.length - 1].endTime = endTime
        } else {
          resTimeList.push({ beginTime, endTime })
        }
      })
      return resTimeList
    },
    isSelected() {
      return this.selected.filter((o) => o === 1).length > 0
    }
  },
  watch: {
    value: {
      handler(value) {
        if (value.length === 0) {
          this.selected = new Array(48).fill(0)
          return
        }
        const changeArr = value.map(({ beginTime, endTime }) => {
          const start =
            Number(beginTime.slice(0, 2)) * 2 +
            (beginTime.slice(3) === '30' ? 1 : 0)
          const end =
            Number(endTime.slice(0, 2)) * 2 +
            (endTime.slice(3) === '30' ? 1 : 0) -
            1
          const deleteCount = end - start + 1
          const newArr = new Array(deleteCount).fill(1)
          return {
            start,
            deleteCount,
            newArr
          }
        })
        changeArr.forEach((item) => {
          this.selected.splice(item.start, item.deleteCount, ...item.newArr)
        })
      },
      immediate: true
    }
  },
  mounted() {
    this.initMarker()
  },
  methods: {
    initMarker() {
      this.marker.style.background = 'rgba(47,136,255,0.5)'
      this.marker.style.position = 'fixed'
      this.marker.style.zIndex = '9999'
      this.marker.style.top = '0'
      this.$refs.table.addEventListener('mousedown', (e) => {
        this.handleMouseDown(e)
      })
    },
    handleClick(index) {
      if (this.disabled) return
      this.$set(this.selected, index, this.selected[index] === 1 ? 0 : 1)
      this.handleChange()
    },
    removeAll() {
      this.selected = this.selected.map(() => 0)
      this.handleChange()
    },
    getTimeObj(index) {
      let startHour = Math.floor(index / 2)
      let endHour = startHour
      let startMin = ''
      let endMin = ''
      if (index % 2 === 0) {
        startMin = '00'
        endMin = '30'
      } else {
        endHour = startHour + 1
        startMin = '30'
        endMin = '00'
      }
      startHour = startHour < 10 ? `0${startHour}` : String(startHour)
      endHour = endHour < 10 ? `0${endHour}` : String(endHour)
      return {
        beginTime: `${startHour}:${startMin}`,
        endTime: `${endHour}:${endMin}`
      }
    },
    handleMouseDown(e) {
      if (this.disabled) return
      this.ismoving = true
      this.startX = e.x
      this.startY = e.y
      this.endX = e.x
      this.endY = e.y
      this.marker.style.width = '0'
      this.marker.style.height = '0'
      this.marker.style.top = `${e.x}px`
      this.marker.style.left = `${e.y}px`
      document.body.appendChild(this.marker)
      // 初始化table的位置信息
      const rects = this.$refs.table.getClientRects()[0]
      this.tablePosition.top = rects.top
      this.tablePosition.left = rects.left
      this.tablePosition.right = rects.right
      this.tablePosition.bottom = rects.bottom
      this.$refs.table.__handleMouseUp__ = (e) => {
        this.handleMouseUp(e)
      }
      this.$refs.table.__handleMouseMove__ = (e) => {
        this.handleMouseMove(e)
      }
      document.body.addEventListener(
        'mouseup',
        this.$refs.table.__handleMouseUp__
      )
      document.body.addEventListener(
        'mousemove',
        this.$refs.table.__handleMouseMove__
      )
    },
    handleMouseMove(e) {
      if (!this.ismoving) {
        return false
      }
      this.endX = e.x
      this.endY = e.y
      const isoverflow =
        e.x <= this.tablePosition.left ||
        e.x >= this.tablePosition.right ||
        e.y <= this.tablePosition.top ||
        e.y >= this.tablePosition.bottom
      if (isoverflow) {
        this.handleMouseUp(e)
        return false
      }
      const width = Math.abs(this.startX - this.endX)
      const height = Math.abs(this.startY - this.endY)
      this.marker.style.width = `${width}px`
      this.marker.style.height = `${height}px`
      if (this.startX < this.endX) {
        this.marker.style.left = `${this.startX}px`
      } else {
        this.marker.style.left = `${this.endX}px`
      }
      if (this.startY < this.endY) {
        this.marker.style.top = `${this.startY}px`
      } else {
        this.marker.style.top = `${this.endY}px`
      }
    },
    handleMouseUp() {
      this.ismoving = false
      try {
        this.marker.parentNode.removeChild(this.marker)
        document.body.removeEventListener(
          'mouseup',
          this.$refs.table.__handleMouseUp__
        )
        document.body.removeEventListener(
          'mousemove',
          this.$refs.table.__handleMouseMove__
        )
        this.handleDragSelect()
      } catch (err) {
        console.log(err)
      }
    },
    handleDragSelect() {
      const startX = Math.min(this.startX, this.endX)
      const endX = Math.max(this.startX, this.endX)
      const startY = Math.min(this.startY, this.endY)
      const endY = Math.max(this.startY, this.endY)
      const shouldClickIndex = []
      this.$refs.table.querySelectorAll('td.time').forEach((node, index) => {
        const rects = node.getClientRects()[0]
        // 左上角是否进入
        const ltIsCover =
          rects.right < endX &&
          rects.bottom < endY &&
          rects.right > startX &&
          rects.bottom > startY
        // 右下角是否进入
        const rbpIsCover =
          rects.left < endX &&
          rects.top < endY &&
          rects.left > startX &&
          rects.top > startY
        // 右上角是否进入
        const rtIsCover =
          rects.left < endX &&
          rects.bottom < endY &&
          rects.left > startX &&
          rects.bottom > startY
        // 左下角是否进入
        const lbIsCover =
          rects.right < endX &&
          rects.top > startY &&
          rects.right > startX &&
          rects.top < endY
        // 左边是否交叉
        const leftIsCover =
          rects.left > startX &&
          rects.left < endX &&
          rects.top < startY &&
          rects.bottom > endY
        // 右边是否交叉
        const rightIsCover =
          rects.right > startX &&
          rects.right < endX &&
          rects.top < startY &&
          rects.bottom > endY
        // 上边是否交叉
        const topIsCover =
          rects.top > startY &&
          rects.top < endY &&
          rects.left < startX &&
          rects.right > endX
        // 下边是否交叉
        const bottomIsCover =
          rects.bottom > startY &&
          rects.bottom < endY &&
          rects.left < startX &&
          rects.right > endX
        const isCover =
          ltIsCover ||
          rbpIsCover ||
          rtIsCover ||
          lbIsCover ||
          leftIsCover ||
          rightIsCover ||
          topIsCover ||
          bottomIsCover
        if (isCover) {
          shouldClickIndex.push(index)
        }
      })
      shouldClickIndex.forEach((item) => {
        this.$set(this.selected, item, this.selected[item] === 1 ? 0 : 1)
      })
      this.handleChange()
    },
    handleChange() {
      this.$emit('change', this.selectTimes)
    }
  }
}
</script>
<style lang="less" scoped>
.time-select {
  display: inline-block;
  max-width: 700px;
  table {
    font-family: PingFangSC-Regular, tahoma, arial, 'Hiragino Sans GB',
      'Microsoft yahei', sans-serif;
    text-align: center;
    border-collapse: collapse;
    font-size: 12px;
    color: #333333;
    user-select: none;
    th,
    td {
      border: 1px solid #dee4f5;
      line-height: 1.8em;
    }
    .label {
      padding: 0 5px;
    }
    .time {
      width: 13px;
      height: 25px;
      background: #f5f5f5;
      &.active {
        background: #2f88ff;
      }
    }
  }
  .tips {
    font-size: 12px;
    color: #999;
    line-height: 20px;
    .item {
      display: flex;
      align-items: center;
      line-height: 20px;
      margin-top: 3px;
      label {
        flex: none;
        padding-right: 5px;
      }
      span {
        flex: 1;
        color: #333;
      }
    }
  }
  .options {
    line-height: 20px;
    position: relative;
  }
  .span-btn {
    color: #409eff;
    font-size: 12px;
    cursor: pointer;
    padding: 7px 0;
    display: inline-block;
  }
}
</style>
