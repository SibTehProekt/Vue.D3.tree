<template>
  <div>
    <svg :width="width" :height="height" v-if="render" ref="svgTree">
      <g transform="translate(100,30) scale(1)" ref="main">

        <line :x1="ln.x1" :y1="ln.y1" :x2="ln.x2" :y2="ln.y2" v-for="ln in tmp.lines" :class="ln.className"></line>

        <text :x="ln.tx" :y="ln.ty" v-for="ln in tmp.lines" text-anchor="middle">{{ ln.text }}</text>

        <!--<transition-group tag="g" name="line" >-->
          <path v-for="link in links" class="link" :key="link.id" :d="link.d" :class="link.className"></path>
        <!--</transition-group>-->

        <!--<transition-group tag="g" name="list">-->
          <g v-for="(node, index) in nodes" :key="node.id" :opacity="node.style.opacity"
             :transform="node.style.transform" :class="node.className">

            <circle :r="node.r" @click="toggleNode(index, node)"></circle>

            <text :dx="node.textpos.x" :dy="node.textpos.y" :style="node.textStyle">{{ node.text }}</text>

          </g>
        <!--</transition-group>-->
      </g>
    </svg>
  </div>
</template>

<script>
import euclidean from './euclidean-layout'
import circular from './circular-layout'
import {compareString, toPromise, translate} from './d3-utils'

import * as d3 from 'd3'

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
    default: 3
  },
  sections: {
    type: Array,
    default: []
  },
  curvature: {
    type: Number,
    default: 100
  }
}

export default {
  name: 'D3Tree',

  props,

  data () {
    return {
      innerData: this.data,
      depth: 0,
      currentTransform: null,
      maxTextLenght: {
        first: 0,
        last: 0
      },
      dataDeep: 0,
      render: false,
      zoom: {
        min: 0.1,
        max: 10
      },
      tree: null,
      tmp: {
        hide: [],
        lines: []
      }
    }
  },
  beforeMount () {
    this.childrenExist(this.innerData)
  },
  updated: function () {
    this.$nextTick(function () {
      this.tmp.lines = this.lines
    })
  },
  mounted () {
    const size = this.getSize()
    const tree = d3.tree()
    this.layout.size(tree, size, this.margin, this.maxTextLenght)
    this.tree = tree

    this.$nextTick(() => {
      this.render = !this.render
      this.$nextTick(() => {
        const svg = d3.select(this.$refs.svgTree)
        let zoom = d3.zoom().scaleExtent([this.zoom.min, this.zoom.max]).on('zoom', this.zoomed(svg.select('g')))
        svg.call(zoom).on('wheel', () => d3.event.preventDefault())
        svg.call(zoom.transform, d3.zoomIdentity)
        this.tmp.lines = this.lines
      })
    })
  },

  methods: {
    childrenExist (dataNode) {
      dataNode.childrenExist = (typeof dataNode.children !== 'undefined')
      if (typeof dataNode.children !== 'undefined' && Array.isArray(dataNode.children)) {
        for (let child of dataNode.children) {
          this.childrenExist(child)
        }
      }
    },

    getSize () {
      let width = this.$el.clientWidth
      let height = this.$el.clientHeight
      return { width, height }
    },

    updateTransform (g, size) {
      size = size || this.getSize()
      return this.layout.updateTransform(g, this.margin, size, this.maxTextLenght)
    },

    onNodeClick (d) {
      if (d.parent !== null) {
        if (d.children) {
          this.collapse(d)
        } else {
          this.expand(d)
        }
      }
    },

    zoomed (g) {
      return () => {
        const transform = d3.event.transform
        const size = this.getSize()
        const transformToApply = this.updateTransform(transform, size)
        this.currentTransform = transform
        this.$emit('zoom', {transform})
        g.attr('transform', transformToApply)
      }
    },

    resetZoom () {
      if (!this.zoomable) {
        return Promise.resolve(false)
      }
      const {svg, zoom} = this.internaldata
      const transitionPromise = toPromise(svg.transition().duration(this.duration).call(zoom.transform, () => d3.zoomIdentity))
      return transitionPromise.then(() => true)
    },

    toggleNode (index, node) {
      let dataNode = this.searchNode(this.innerData, node.id)
      if (dataNode.children !== null) {
        dataNode._children = dataNode.children
        dataNode.children = null
      } else {
        dataNode.children = dataNode._children
        dataNode._children = null
      }
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
    }
  },

  computed: {
    root () {
      if (this.innerData) {
        return this.tree(d3.hierarchy(this.innerData).sort((a, b) => {
          return compareString(a.data.text, b.data.text)
        }))
      }
      return null
    },
    nodes () {
      if (this.root) {
        this.depth = 0
        return this.root.descendants().map(d => {
          if (this.depth < d.depth) {
            this.depth = d.depth
          }
          return {
            id: d.data.id,
            r: this.radius,
            className: 'nodetree' +
              (d.children && d.children !== null ? ' node--internal-opened' : ' node--internal-closed') +
              (!d.data.childrenExist || d.parent === null ? ' node--notclick' : ''),
            text: d.data.name,
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
      if (this.root) {
        return this.root.descendants().slice(1).map((d) => {
          let y1 = d.y
          let x1 = d.x
          let y2 = d.parent.y + this.curvature
          let x2 = d.x
          let y3 = d.parent.y + this.curvature
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
      if (this.nodes && this.$refs.main) {
        let lines = []
        let w = d3.select(this.$refs.main).select('path').node().getBBox().width
        let x = w / 2
        for (let i = 0; i <= this.depth; i++) {
          lines.push({
            tx: x - (w / 2),
            ty: -25,
            text: this.sections[i],
            x1: x,
            y1: -30,
            x2: x,
            y2: this.$refs.main.getBBox().height + 30,
            className: 'linktree'
          })
          x += w
        }
        return lines
      }
      return []
    },
    margin () {
      return {x: this.marginX, y: this.marginY}
    },

    layout () {
      return layout[this.layoutType]
    }
  },

  watch: {
    data (current, old) {
      this.onData(current)
    },

    rendered () {
      console.log(this.lines)
    },

    grid () {
      this.removeGrid()
      this.redraw()
    },

    type () {
      if (!this.internaldata.tree) {
        return
      }
      this.internaldata.tree = this.tree
      this.redraw()
    },

    marginX (newMarginX, oldMarginX) {
      this.completeRedraw({margin: {x: oldMarginX, y: this.marginY}})
    },

    marginY (newMarginY, oldMarginY) {
      this.completeRedraw({margin: {x: this.marginX, y: oldMarginY}})
    },

    layout (newLayout, oldLayout) {
      this.completeRedraw({layout: oldLayout})
    },

    radius () {
      this.completeRedraw({layout: this.layout})
    }
  }
}
</script>

<style>
svg {
  transform: translate3d(0, 0, 0);
  backface-visibility: hidden;
  perspective: 1000;
}

svg * {
  will-change: transform;
}

.treeclass .nodetree  circle {
  fill: rgb(255, 255, 255);
  stroke: steelblue;
  stroke-width: 1.5px;
}

.treeclass .node--notclick circle {
  cursor: default !important;
}

.treeclass .node--internal-opened  circle {
  fill: rgb(255, 255, 255);
  cursor: pointer;
}

.treeclass .node--internal-closed circle {
  fill:  rgb(176, 196, 222);
  cursor: pointer;
}

.treeclass .nodetree {
  /*transition: all 1s;*/
}

.treeclass .nodetree text {
  font: 10px sans-serif;
  cursor: pointer;
}

.treeclass .nodetree.selected text {
  font-weight: bold;
}

.treeclass .node--internal text {
  text-shadow: 0 1px 0 #fff, 0 -1px 0 #fff, 1px 0 0 #fff, -1px 0 0 #fff;
}

.treeclass .linktree {
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
