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
          <g v-for="(node, index) in nodes" :key="node.id" :opacity="node.style.opacity"
             :transform="node.style.transform" :class="node.className"
             @click="toggleNode($event, index, node)"
             @dblclick="onDblClickNode($event, index, node)"
             @contextmenu="contextMenuNode($event, index, node)">

            <circle :r="node.r"></circle>
            <text :dx="node.textpos.x" :dy="node.textpos.y" :style="node.textStyle">{{ node.text }}</text>

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
import {isEmpty, cloneDeep} from 'lodash'

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
      minMargin: 100,
      isMouseDown: false
    }
  },
  beforeMount () {
    if (!this.dataIsEmpty && typeof this.innerData.children !== 'undefined') {
      this.addFields(this.innerData)
    }
  },
  updated: function () {
    this.$nextTick(function () {
      this.updateLines()
    })
  },
  mounted () {
    this.update()
    this.$nextTick(() => {
      this.render = !this.render
      this.$nextTick(() => {
        if (this.zoomable) {
          const svg = d3.select(this.$refs.svgTree)
          let zoom = d3.zoom().scaleExtent([this.zoomMin, this.zoomMax]).on('zoom', this.zoomed(svg.select('g')))
          svg.call(zoom).on('wheel', () => d3.event.preventDefault())
          svg.call(zoom.transform, d3.zoomIdentity)
        }
        this.updateLines()
      })
    })
  },

  methods: {
    update () {
      this.tree = this.initLayout()
    },
    updateLines () {
      this.tmp.lines = this.lines
    },
    upgrade () {
      this.innerData = cloneDeep(this.data)
      this.cleanTmpAutoMarginCounter()
      this.addFields(this.innerData)
      this.update()
    },
    initLayout () {
      const size = this.getSize()
      const tree = this.type === 'cluster' ? d3.cluster() : d3.tree()
      const margin = this.margin(this.autoMarginY, this.autoMarginX)
      this.layout.size(tree, size, margin, this.maxTextLenght)
      return tree
    },
    addFields (dataNode, deep = 0) {
      if (typeof this.tmp.automargin.counter[deep] === 'undefined') {
        this.tmp.automargin.counter[deep] = 0
      }
      dataNode.childrenExist = (typeof dataNode.children !== 'undefined')
      dataNode.clicking = this.hideDeepNodes && this.deep <= deep
      dataNode.deep = deep
      if (!this.hideDeepNodes || this.deep >= deep) this.tmp.automargin.counter[deep]++
      if (typeof dataNode.children !== 'undefined' && Array.isArray(dataNode.children)) {
        deep++
        for (let child of dataNode.children) {
          this.addFields(child, deep)
        }
      }
      if (this.hideDeepNodes && this.deep < deep) {
        dataNode._children = dataNode.children
        dataNode.children = null
      }
    },
    getSize () {
      let width = this.$el.clientWidth
      let height = this.$el.clientHeight
      return { width, height }
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
    resetZoom () {
      if (!this.zoomable) {
        return Promise.resolve(false)
      }
      const svg = d3.select(this.$refs.svgTree)
      let zoom = d3.zoom().scaleExtent([this.zoomMin, this.zoomMax]).on('zoom', this.zoomed(svg.select('g')))
      const transitionPromise = toPromise(
        svg.transition().duration(this.duration).call(zoom.transform, () => d3.zoomIdentity)
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
    getAutoMarginY () {
      let max = 0
      for (let item in this.tmp.automargin.counter) {
        if (this.tmp.automargin.counter[item] > max) {
          max = this.tmp.automargin.counter[item]
        }
      }
      let marginY = max * (this.radius * 2) * this.coefficientY
      return this.culcMargin(marginY)
    },
    getAutoMarginX () {
      let max = 0
      for (let item in this.tmp.automargin.counter) {
        if (this.tmp.automargin.counter[item] > max) {
          max = this.tmp.automargin.counter[item]
        }
      }
      let marginX = max / 10 * this.radius * this.coefficientY
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

    // методы - обработки событий
    toggleNode (e, index, node) {
      if ((this.clickableDefaultNodes || node.deep >= this.deep) && node.childrenExist && node.parentExist) {
        let dataNode = this.searchNode(this.innerData, node.id)
        if (dataNode.children !== null) {
          this.tmp.automargin.counter[dataNode.deep + 1] -= dataNode.children.length
          dataNode._children = dataNode.children
          dataNode.children = null
        } else {
          this.tmp.automargin.counter[dataNode.deep + 1] += dataNode._children.length
          dataNode.children = dataNode._children
          dataNode._children = null
        }
        this.update()
      }
      e.stopPropagation()
      this.$emit('onClickNode', e, index, node)
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
            (!d.data.childrenExist || d.parent === null || (this.hideDeepNodes && !d.data.clicking)
              ? ' node--notclick' : '')

          return {
            id: d.data.id,
            r: this.radius,
            className: className,
            childrenExist: d.data.childrenExist,
            parentExist: d.parent !== null,
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
        let x = w / 2
        let y1 = null
        let y2 = 0
        for (let node of mainEl.selectAll('path').nodes()) {
          let box = node.getBBox()
          if (y1 === null || y1 > box.y) y1 = box.y
          let y = box.y + box.height
          if (y2 < y) y2 = y
        }
        for (let i = 0; i <= this.depth; i++) {
          lines.push({
            tx: x - (w / 2),
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
    }
  },

  watch: {
    data (current, old) {
      this.onData(current)
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
