<template>
  <div>
    <svg :width="width" :height="height" v-if="!dataIsEmpty && render" ref="svgTree"
         @click="clickSpace"
         @dblclick="onDblClickSpace"
         @contextmenu="contextMenuSpace">
      <g ref="main">

        <line :x1="ln.x1" :y1="ln.y1" :x2="ln.x2" :y2="ln.y2" v-for="ln in tmp.lines" :class="ln.className"></line>
        <text :x="ln.tx" :y="ln.ty" v-for="ln in tmp.lines" text-anchor="middle">{{ ln.text }}</text>

        <!--<transition-group tag="g" name="line" >-->
          <path v-for="link in links" class="link" :key="link.id" :d="link.d" :class="link.className"></path>
        <!--</transition-group>-->

        <!--<transition-group tag="g" name="list">-->
          <g v-for="(node, index) in nodes"
             :key="node.id"
             :opacity="node.style.opacity"
             :transform="node.style.transform"
             :class="node.className"
             @mousemove="mouseMoveNode($event, index, node)"
             @mouseover="mouseOverNode($event, index, node)"
             @mouseout="mouseOutNode($event, index, node)"
             @click="toggleNode($event, index, node)"
             @dblclick="onDblClickNode($event, index, node)"
             @contextmenu="contextMenuNode($event, index, node)">

            <circle :r="node.r"></circle>
            <text dx="0" :dy="radius <= 8 ? -10 : 3" text-anchor="middle" v-if="node.childrenExist">
              {{ node.obj.children ? node.obj.children.length : node.obj._children.length }}
            </text>
            <text :dx="node.textpos.x" :dy="node.textpos.y":style="node.textStyle">{{ node.text }}</text>
            <rect :x="-radius * 1.5" :y="-radius * 1.5" :width="radius * 3" :height="radius * 3" rx="50" fill="rgba(0,0,0,0)"/>

          </g>
        <!--</transition-group>-->
      </g>
    </svg>
    <svg :width="width" :height="height" ref="dataisempty" v-else-if="render">
      <text :x="$el.clientWidth / 2" :y="$el.clientHeight / 2" text-anchor="middle">{{ noDataText }}</text>
    </svg>
  </div>
</template>

<script>
import euclidean from './euclidean-layout'
import circular from './circular-layout'
import {compareString, toPromise, translate} from './d3-utils'
import * as d3 from 'd3'
import {isEmpty, cloneDeep, isNumber, toNumber} from 'lodash'

const layout = {
  euclidean,
  circular
}

let i = 0
const types = ['tree', 'cluster']
const layouts = ['circular', 'euclidean']

const props = {
  data: Object,
  duration: {
    type: Number,
    default: 600
  },
  type: {
    type: String,
    default: 'tree',
    validator (value) {
      return types.indexOf(value) !== -1
    }
  },
  width: {
    type: String,
    default: '100%'
  },
  height: {
    type: String,
    default: '100%'
  },
  layoutType: {
    type: String,
    default: 'euclidean',
    validator (value) {
      return layouts.indexOf(value) !== -1
    }
  },
  marginX: {
    type: Number,
    default: 20
  },
  marginY: {
    type: Number,
    default: 20
  },
  autoMarginX: {
    type: Boolean,
    default: false
  },
  autoMarginY: {
    type: Boolean,
    default: false
  },
  nodeText: {
    type: String,
    required: true
  },
  identifier: {
    type: Function,
    default: () => i++
  },
  zoomable: {
    type: Boolean,
    default: false
  },
  radius: {
    type: Number,
    default: 3
  },
  grid: {
    type: Boolean,
    default: false
  },
  deep: {
    type: Number,
    default: 2
  },
  sections: {
    type: Array,
    default: []
  },
  hideDeepNodes: {
    type: Boolean,
    default: false
  },
  coefficientX: {
    type: Number,
    default: 1
  },
  coefficientY: {
    type: Number,
    default: 1
  },
  zoomMax: {
    type: Number,
    default: 1.5
  },
  zoomMin: {
    type: Number,
    default: 0.1
  },
  gridMarginY: {
    type: Number,
    default: 50
  },
  clickableDefaultNodes: {
    type: Boolean,
    default: true
  },
  noDataText: {
    type: String,
    default: 'Data not available'
  },
  counter: {
    type: Boolean,
    default: false
  }
}

export default {
  name: 'D3Tree',
  props,
  data () {
    return {
      innerData: cloneDeep(this.data),
      depth: 0,
      maxTextLenght: {
        first: 0,
        last: 0
      },
      render: false,
      tree: null,
      tmp: {
        lines: [],
        automargin: {
          counter: []
        },
        position: {
          y1: 0,
          y2: 100
        }
      },
      minMargin: 0,
      zoom: {
        x: 0,
        y: 0,
        scale: 1
      },
      history: {
        nodes: [],
        zoom: []
      },
      snapshotOpenNodes: {}
    }
  },

  beforeMount () {
    if (!this.dataIsEmpty && typeof this.innerData.children !== 'undefined') {
      // Добавление нужных полей и т.д.
      // Можно сказать нормализация дерева для дальнейшей работы
      this.addFields(this.innerData)
    }
  },

  updated: function () {
    this.$nextTick(function () {
      this.updateLines()
    })
  },

  /**
   * @package d3
   */
  mounted () {
    let svg = null
    let zoom = null
    this.$nextTick().then(() => {
      this.update()
      this.render = !this.render
    }).then(() => {
      this.updateLines()
      if (this.zoomable) {
        svg = d3.select(this.$refs.svgTree)
        zoom = d3.zoom().scaleExtent([this.zoomMin, this.zoomMax]).on('zoom', this.zoomed(svg.select('g')))
      }
    }).then(() => {
      if (this.zoomable) {
        if (!this.dataIsEmpty && !this.dataNotChildren) {
          this.computeZoom()
        }
        svg.call(zoom).on('wheel', () => d3.event.preventDefault())
        svg.call(zoom.transform, !this.dataIsEmpty && !this.dataNotChildren ? this.getDefaultZoom() : d3.zoomIdentity)
      }
    }).then(() => {
      this.createSnapshotOfOpenNodes()
    })
  },

  methods: {
    /**
     * @package d3
     */
    getDefaultZoom () {
      return d3.zoomIdentity.translate(this.zoom.x, this.zoom.y).scale(this.zoom.scale)
    },

    /**
     * Просчет зума
     *
     * @package d3
     */
    computeZoom () {
      let tree = this.$refs.svgTree
      let main = this.$refs.main
      let pw = d3.select('path').node().getBBox().width

      let scaleByWidth = tree.clientWidth / ((main.getBBox().width + pw / 2))
      let scaleByHeight = tree.clientHeight / (main.getBBox().height + this.gridMarginY * 2)
      let scale = scaleByWidth > scaleByHeight ? scaleByHeight : scaleByWidth
      scale = scale < this.zoomMin ? this.zoomMin : (scale > this.zoomMax ? this.zoomMax : scale)

      let x = tree.clientWidth / 2
      x += (pw / 2 + this.getMaxDepth()) * scale // встаем в начало сетки
      x -= pw * (this.depth + 1) * scale / 2 // отнимаем половину растояния сетки, чтобы встать в центр сетки

      let y = this.getMaxDepth() * this.diameter * scale
      y += (tree.clientHeight / 2 - main.getBBox().height / 2 * scale)
      y -= this.gridMarginY / 2 * scale

      this.zoom.scale = scale
      this.zoom.x = x
      this.zoom.y = y
    },

    update () {
      this.tree = this.initLayout()
    },

    updateLines () {
      this.tmp.lines = this.lines
    },

    upgrade (isZoom = false, durationZoom = null) {
      this.innerData = cloneDeep(this.data)
      this.cleanTmpAutoMarginCounter()
      this.addFields(this.innerData)
      this.update()

      if (isZoom) {
        this.$nextTick().then(() => {
          this.$nextTick(() => {
            if (!this.dataIsEmpty && !this.dataNotChildren) {
              this.computeZoom()
            }
            this.resetZoom(durationZoom)
          })
        })
      }
    },

    /**
     * Инициализация шаблона
     *
     * @package d3
     * @return
     */
    initLayout () {
      const size = this.getSize()
      const tree = this.type === 'cluster' ? d3.cluster() : d3.tree()
      const margin = this.margin(this.autoMarginY, this.autoMarginX)
      this.layout.size(tree, size, margin, this.maxTextLenght)
      return tree
    },

    /**
     * Добавление дополнительных полей для дерева
     * Так же закрытие нод с дочерними элементами относительно указанной глубины deep
     *
     * @param dataNode
     * @param deep
     *
     * @return void
     */
    addFields (dataNode, deep = 0) {
      if (typeof this.tmp.automargin.counter[deep] === 'undefined') {
        this.tmp.automargin.counter[deep] = 0
      }

      // само добавление полей
      dataNode.childrenExist = (typeof dataNode.children !== 'undefined')
      dataNode.clicking = this.hideDeepNodes && this.deep <= deep
      dataNode.deep = deep
      dataNode._children = null

      if (!this.hideDeepNodes || this.deep >= deep) {
        // коунтирвание для просчета маргина
        this.tmp.automargin.counter[deep]++
      }

      if (typeof dataNode.children !== 'undefined' && Array.isArray(dataNode.children)) {
        deep++
        for (let child of dataNode.children) {
          this.addFields(child, deep)
        }
      }

      // закрытие всех нод находящихся ниже по дереву относительно указанной глубины deep
      if (this.hideDeepNodes && this.deep < deep) {
        this.closeNode(dataNode)
      }
    },

    getSize () {
      let width = this.$el.clientWidth
      let height = this.$el.clientHeight
      return { width, height }
    },

    openNode (dataTree) {
      if (typeof dataTree._children !== 'undefined' && !isEmpty(dataTree._children)) {
        dataTree.children = dataTree._children
        dataTree._children = null
      }
    },

    closeNode (dataTree) {
      if (typeof dataTree.children !== 'undefined' && !isEmpty(dataTree.children)) {
        dataTree._children = dataTree.children
        dataTree.children = null
      }
    },

    isOpenNode (dataTree) {
      return !isEmpty(dataTree.children)
    },

    hasChildrenNode (dataTree) {
      return dataTree.childrenExist
    },

    updateTransform (g, size) {
      size = size || this.getSize()
      const margin = this.margin(this.autoMarginY, this.autoMarginX)
      this.$emit('moveSpace', g, size)
      return this.layout.updateTransform(g, margin, size, this.maxTextLenght)
    },

    zoomed (g) {
      return () => {
        const transform = d3.event.transform
        const size = this.getSize()
        const transformToApply = this.updateTransform(transform, size)
        this.$emit('zoom', {transform})
        g.attr('transform', transformToApply)
      }
    },

    resetZoom (duration = null) {
      if (!this.zoomable) {
        return Promise.resolve(false)
      }

      if (!isNumber(duration)) {
        duration = this.duration
      }

      const svg = d3.select(this.$refs.svgTree)
      let zoom = d3.zoom().scaleExtent([this.zoomMin, this.zoomMax]).on('zoom', this.zoomed(svg.select('g')))
      const transitionPromise = toPromise(
        svg.transition().duration(duration).call(zoom.transform, () => {
          return !this.dataIsEmpty && !this.dataNotChildren ? this.getDefaultZoom() : d3.zoomIdentity
        })
      )
      return transitionPromise.then(() => true)
    },

    searchNode (dataNode, id) {
      if (dataNode.id === id) {
        return dataNode
      }
      if (typeof dataNode.children !== 'undefined' && Array.isArray(dataNode.children)) {
        for (let child of dataNode.children) {
          let res = this.searchNode(child, id)
          if (res !== null) {
            return res
          }
        }
      }
      return null
    },

    select: (index, node) => {
      this.selected = index
    },

    margin (autoMarginY = false, autoMarginX = false) {
      return {
        x: autoMarginX ? this.getAutoMarginX() : this.marginX,
        y: autoMarginY ? this.getAutoMarginY() : this.marginY
      }
    },

    getMaxDepth () {
      let max = 0
      for (let item in this.tmp.automargin.counter) {
        if (this.tmp.automargin.counter[item] > max) {
          max = this.tmp.automargin.counter[item]
        }
      }
      return max
    },

    getAutoMarginY () {
      let marginY = this.getMaxDepth() * this.diameter * this.coefficientY
      return this.culcMargin(marginY)
    },

    getAutoMarginX () {
      let marginX = this.getMaxDepth() / 10 * this.radius * this.coefficientY
      return this.culcMargin(marginX)
    },

    culcMargin (margin) {
      return -(margin > this.minMargin ? margin : this.minMargin)
    },

    cleanTmpAutoMarginCounter () {
      for (let i in this.tmp.automargin.counter) {
        this.tmp.automargin.counter[i] = undefined
      }
    },

    /**
     * Создание слепка открытых нод ()
     */
    createSnapshotOfOpenNodes () {
      return this._createSnapshotOfOpenNodes(this.innerData, this.snapshotOpenNodes)
    },

    /**
     * Рекурсивный внутренний метод для создания слепка открытых нод
     */
    _createSnapshotOfOpenNodes (dataTree, activeCast) {
      if (dataTree.childrenExist && dataTree.children !== null) {
        activeCast.id = dataTree.instanceId
        activeCast.children = []
        for (let child of dataTree.children) {
          if (child.childrenExist && child.children !== null) {
            let tmp = {}
            activeCast.children.push(this._createSnapshotOfOpenNodes(child, tmp))
          }
        }
        return activeCast
      }
    },

    printSnapshotOfOpenNodes (zoomPosition, durationZoom = 0) {
      Promise.resolve().then(() => {
        this._printSnapshotOfOpenNodes(this.innerData, this.snapshotOpenNodes)
      }).then(() => {
        this.cleanTmpAutoMarginCounter()
      }).then(() => {
        this.culcAutomarginDeem(this.innerData)
      })
    },

    _printSnapshotOfOpenNodes (dataTree, activeCast, deep = 0) {
      if (dataTree.instanceId === activeCast.id) {
        if (!this.isOpenNode(dataTree)) {
          this.openNode(dataTree)
        }
      }

      if (this.hasChildrenNode(dataTree) && !isEmpty(activeCast.children) && !isEmpty(dataTree.children)) {
        for (let child of dataTree.children) {
          for (let childActiveCast of activeCast.children) {
            this._printSnapshotOfOpenNodes(child, childActiveCast, deep)
          }
        }
      }
    },

    getZoomPosition () {
      let tmp = d3.select(this.$refs.main).attr('transform').match(/[\d.]+/g)
      return {
        x: toNumber(tmp[0]),
        y: toNumber(tmp[1]),
        scale: toNumber(tmp[2])
      }
    },

    culcAutomarginDeem (dataNode, deep = 0) {
      if (typeof this.tmp.automargin.counter[deep] === 'undefined') {
        this.tmp.automargin.counter[deep] = 0
      }

      if (deep === 0) {
        this.tmp.automargin.counter[deep]++
      }

      if (typeof dataNode.children !== 'undefined' && !isEmpty(dataNode.children)) {
        deep++

        if (typeof this.tmp.automargin.counter[deep] === 'undefined') {
          this.tmp.automargin.counter[deep] = 0
        }

        for (let child of dataNode.children) {
          // коунтирвание для просчета маргина
          this.tmp.automargin.counter[deep]++
          this.culcAutomarginDeem(child, deep)
        }
      }
    },

    /**
     * Установка позиции
     */
    setZoomPosition (zoomPosition, duration = 0) {
      console.log(this.isZoomPositioObject(zoomPosition))
      if (!this.zoomable || !this.isZoomPositioObject(zoomPosition)) {
        return Promise.resolve(false)
      }

      this.zoomPositionValidScale(zoomPosition)

      if (!isNumber(duration)) duration = 0

      const svg = d3.select(this.$refs.svgTree)
      let zoom = d3.zoom().scaleExtent([this.zoomMin, this.zoomMax]).on('zoom', this.zoomed(svg.select('g')))
      const transitionPromise = toPromise(
        svg.transition().duration(duration).call(zoom.transform, () => {
          return d3.zoomIdentity.translate(zoomPosition.x, zoomPosition.y).scale(zoomPosition.scale)
        })
      )
      return transitionPromise.then(() => true)
    },

    zoomPositionValidScale (zoomPosition) {
      if (zoomPosition.scale < this.zoomMin) {
        zoomPosition.scale = this.zoomMin
      } else if (zoomPosition.scale > this.zoomMax) {
        zoomPosition.scale = this.zoomMax
      }
    },

    isZoomPositioObject (zoomPosition) {
      return (typeof zoomPosition.x !== 'undefined' && isNumber(zoomPosition.x) &&
        typeof zoomPosition.y !== 'undefined' && isNumber(zoomPosition.y) &&
        typeof zoomPosition.scale !== 'undefined' && isNumber(zoomPosition.scale))
    },

    // методы - обработки событий

    /**
     * @package lodash
     * @param e
     * @param index
     * @param node
     */
    toggleNode (e, index, node) {
      if ((this.clickableDefaultNodes || node.deep >= this.deep) && node.childrenExist && !isEmpty(node.parent)) {
        let dataNode = this.searchNode(this.innerData, node.id)
        if (dataNode.children !== null) {
          this.tmp.automargin.counter[dataNode.deep + 1] -= dataNode.children.length
          this.closeNode(dataNode)
        } else {
          this.tmp.automargin.counter[dataNode.deep + 1] += dataNode._children.length
          this.openNode(dataNode)
        }

        this.update()
        this.$nextTick(() => {
          if (!this.dataIsEmpty && !this.dataNotChildren) {
            this.computeZoom()
          }
        })
      }
      e.stopPropagation()
      this.$emit('onClickNode', e, index, node)
    },

    mouseMoveNode (e, index, node) {
      this.$emit('mouseMoveNode', e, index, node)
    },

    mouseOverNode (e, index, node) {
      this.$emit('mouseOverNode', e, index, node)
    },

    mouseOutNode (e, index, node) {
      this.$emit('mouseOutNode', e, index, node)
    },

    onDblClickNode (e, index, node) {
      e.stopPropagation()
      this.$emit('onDblClickNode', e, index, node)
    },

    contextMenuNode (e, index, node) {
      e.stopPropagation()
      this.$emit('contextMenuNode', e, index, node)
    },

    clickSpace (e) {
      this.$emit('clickSpace', e)
    },

    onDblClickSpace (e) {
      this.$emit('onDblClickSpace', e)
    },

    contextMenuSpace (e) {
      this.$emit('contextMenuSpace', e)
    }
  },

  computed: {
    root () {
      if (!this.dataIsEmpty) {
        return this.tree(d3.hierarchy(this.innerData).sort((a, b) => {
          return compareString(a.data.text, b.data.text)
        }))
      }
      return null
    },

    nodes () {
      if (!this.dataIsEmpty && this.root) {
        this.depth = 0
        return this.root.descendants().map(d => {
          if (this.depth < d.depth) {
            this.depth = d.depth
          }

          let className = 'nodetree' +
            (d.children && d.children !== null ? ' node--internal-opened' : ' node--internal-closed') +
            (!d.data.childrenExist ? ' node--not-children' : '') +
            (!d.data.childrenExist || d.parent === null || (this.hideDeepNodes && !d.data.clicking)
              ? ' node--notclick' : '')

          let parent = Object.assign({}, d.parent)

          if (typeof parent.children !== 'undefined') {
            delete parent.children
          }

          if (typeof parent._children !== 'undefined') {
            delete parent._children
          }

          return {
            id: d.data.id,
            r: this.radius,
            className: className,
            childrenExist: d.data.childrenExist,
            parent: parent,
            obj: d.data,
            text: d.data.name,
            deep: d.data.deep,
            style: {
              transform: translate(d, this.layout),
              opacity: 1
            },
            textpos: {
              x: d.children ? -13 : 13,
              y: 3
            },
            textStyle: {
              textAnchor: d.children ? 'end' : 'start'
            }
          }
        })
      }
    },

    links () {
      if (!this.dataIsEmpty && this.root) {
        return this.root.descendants().slice(1).map((d) => {
          let y1 = d.y
          let x1 = d.x
          let y2 = d.parent.y + (d.y - d.parent.y) / 2
          let x2 = d.x
          let y3 = d.parent.y + (d.y - d.parent.y) / 2
          let x3 = d.parent.x
          let y4 = d.parent.y
          let x4 = d.parent.x
          return {
            id: `link${d.data.id}`,
            d: `M${y1},${x1} C ${y2},${x2} ${y3},${x3} ${y4},${x4}`,
            className: 'linktree'
          }
        })
      }
    },

    lines () {
      if (!this.dataIsEmpty && typeof this.innerData.children !== 'undefined' && this.$refs.main && this.grid) {
        let mainEl = d3.select(this.$refs.main)
        let lines = []
        let path = mainEl.select('path')
        let w = path.node().getBBox().width
        let hfW = w / 2
        let x = hfW
        let y1 = null
        let y2 = 0
        for (let node of mainEl.selectAll('path').nodes()) {
          let box = node.getBBox()
          if (y1 === null || y1 > box.y) y1 = box.y
          let y = box.y + box.height
          if (y2 < y) y2 = y
        }
        lines.push({
          tx: 0,
          ty: 0,
          text: '',
          x1: -hfW,
          y1: y1 - this.gridMarginY,
          x2: -hfW,
          y2: y2 + this.gridMarginY,
          className: 'linktree'
        })
        for (let i = 0; i <= this.depth; i++) {
          lines.push({
            tx: x - hfW,
            ty: y1 - this.gridMarginY,
            text: this.sections[i],
            x1: x,
            y1: y1 - this.gridMarginY,
            x2: x,
            y2: y2 + this.gridMarginY,
            className: 'linktree'
          })
          x += w
        }
        return lines
      }
      return []
    },

    layout () {
      return layout[this.layoutType]
    },

    dataIsEmpty () {
      return isEmpty(this.innerData)
    },

    dataNotChildren () {
      return typeof this.innerData.children === 'undefined'
    },

    diameter () {
      return this.radius * 2
    }
  },

  watch: {
    data (current, old) {
      this.createSnapshotOfOpenNodes()
      this.innerData = cloneDeep(this.data)
      this.cleanTmpAutoMarginCounter()
      this.addFields(this.innerData)
      this.printSnapshotOfOpenNodes()
      // this.printSnapshotOfOpenNodes()
    },

    type () {
      this.update()
    },

    grid () {
      this.updateLines()
    },

    autoMarginX () {
      this.update()
    },

    autoMarginY () {
      this.update()
    },

    marginX () {
      this.updateLines()
      this.update()
    },

    marginY () {
      this.updateLines()
      this.update()
    },

    layout (newLayout, oldLayout) {
      // this.completeRedraw({layout: oldLayout})
      this.update()
    },

    radius () {
      this.update()
    },

    deep () {
      this.upgrade()
    },

    hideDeepNodes () {
      this.upgrade()
    }
  }
}
</script>

<style scoped>
svg {
  transform: translate3d(0, 0, 0);
  backface-visibility: hidden;
  perspective: 1000;
}

svg * {
  will-change: transform;
}

.nodetree  circle {
  fill: rgb(255, 255, 255);
  stroke: steelblue;
  stroke-width: 1.5px;
}

.node--notclick circle {
  cursor: default !important;
}

.node--not-children circle {
  fill: rgb(255, 255, 255) !important;
}

.node--internal-opened  circle {
  fill: rgb(255, 255, 255);
  cursor: pointer;
}

.node--internal-closed circle {
  fill:  rgb(176, 196, 222);
  cursor: pointer;
}

.nodetree {
  /*transition: all 1s;*/
}

.nodetree text {
  font: 10px sans-serif;
  cursor: pointer;
}

.nodetree.selected text {
  font-weight: bold;
}

.node--internal text {
  text-shadow: 0 1px 0 #fff, 0 -1px 0 #fff, 1px 0 0 #fff, -1px 0 0 #fff;
}

.linktree {
  fill: none;
  stroke: #555;
  stroke-opacity: 0.4;
  stroke-width: 1.5px;
  /*transition: all 1s;*/
}

.node circle {
  fill: rgb(255, 255, 255);
  stroke: steelblue;
  stroke-width: 1.5px;
}

svg line {
  stroke-dasharray: 5, 10, 5;
  stroke: #DDDDDD;
}

svg text.section-name {
  stroke: #666666;
}
</style>
