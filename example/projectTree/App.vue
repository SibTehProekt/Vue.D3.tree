<template>
  <div id="app" class="container-fluid">
    <div class="col-md-3">

      <div class="panel panel-default">
        <div class="panel-heading">Props</div>

        <div class="panel-body">
            <div class="form-horizontal">

            <div class="form-group">
              <label for="clickable" class="control-label">Clickable deep nodes</label>
              <input type="checkbox" id="clickable" v-model="clickableDefaultNodes"/>
            </div>

            <div class="form-group">
              <label for="hideDeepNodes" class="control-label">Hide deep nodes</label>
              <input type="checkbox" id="hideDeepNodes" v-model="hideDeepNodes"/>
            </div>

            <div class="form-group">
              <label for="grid" class="control-label">Grid</label>
              <input type="checkbox" id="grid" v-model="grid"/>
            </div>

            <div class="form-group">
              <label for="type" class="control-label col-sm-3">type</label>
              <div  class="col-sm-9">
                <select id="type" class="form-control" v-model="type">
                  <option>tree</option>
                  <option>cluster</option>
                </select>
              </div>
            </div>

            <div class="form-group">
              <label for="layout-type" class="control-label col-sm-3">layoutType</label>
              <div  class="col-sm-9">
                <select id="layout-type" class="form-control" v-model="layoutType">
                  <option>euclidean</option>
                  <option>circular</option>
                </select>
              </div>
            </div>

            <div class="form-group">
              <label for="deep" class="control-label col-sm-3">Deep</label>
              <div class="col-sm-7">
                <input id="deep" class="form-control" type="number" min="2" max="5" v-model.number="deep">
              </div>
            </div>

            <div class="form-group">
              <label for="automarginx" class="control-label">Automargin X</label>
              <input type="checkbox" id="automarginx" v-model="autoMarginX"/>
            </div>

            <div class="form-group" v-show="!autoMarginX">
              <label for="marginx" class="control-label col-sm-3">marginx</label>
              <div class="col-sm-7">
                <input id="marginx" class="form-control" type="range" min="-2000" max="2000" v-model.number="Marginx">
              </div>
                <div class="col-sm-2">
                  <p>{{Marginx}}px</p>
              </div>
            </div>

            <div class="form-group">
              <label for="automarginy" class="control-label">Automargin Y</label>
              <input type="checkbox" id="automarginy" v-model="autoMarginY"/>
            </div>

            <div class="form-group" v-show="!autoMarginY">
              <label for="marginy" class="control-label col-sm-3">marginy</label>
              <div class="col-sm-7">
                <input id="marginy" class="form-control" type="range" min="-5000" max="5000" v-model.number="Marginy">
              </div>
              <div class="col-sm-2">
                <p>{{Marginy}}px</p>
              </div>
            </div>

             <div class="form-group">
              <label for="radius" class="control-label col-sm-3">radius</label>
              <div class="col-sm-7">
                <input id="radius" class="form-control" type="range" min="1" max="10" v-model.number="radius">
              </div>
              <div class="col-sm-2">
                <p>{{radius}}px</p>
              </div>
            </div>

            <div class="form-group">
              <label for="velocity" class="control-label col-sm-3">Duration</label>
              <div class="col-sm-7">
                <input id="velocity" class="form-control" type="range" min="0" max="3000" v-model.number="duration">
              </div>
              <div class="col-sm-2">
                <p>{{duration}}ms</p>
              </div>
            </div>

            <div class="form-group">
              <span v-if="currentNode">Current Node: {{currentNode.data.text}}</span>
              <span v-else>No Node selected.</span>
               <i v-if="isLoading" class="fa fa-spinner fa-spin fa-2x fa-fw"></i>
            </div>

            <button type="button" :disabled="!currentNode" class="btn btn-primary" @click="expandAll" data-toggle="tooltip" data-placement="top" title="Expand All from current">
            <i class="fa fa-expand" aria-hidden="true"></i>
            </button>

            <button type="button" :disabled="!currentNode" class="btn btn-secondary" @click="collapseAll" data-toggle="tooltip" data-placement="top" title="Collapse All from current">
            <i class="fa fa-compress" aria-hidden="true"></i>
            </button>

            <button type="button" :disabled="!currentNode" class="btn btn-success" @click="showOnly" data-toggle="tooltip" data-placement="top" title="Show Only from current">
            <i class="fa fa-search-plus" aria-hidden="true"></i>
            </button>

            <button type="button" :disabled="!currentNode" class="btn btn-warning" @click="show" data-toggle="tooltip" data-placement="top" title="Show current">
            <i class="fa fa-binoculars" aria-hidden="true"></i>
            </button>

            <button v-if="zoomable" type="button" class="btn btn-warning" @click="resetZoom" data-toggle="tooltip" data-placement="top" title="Reset Zoom">
            <i class="fa fa-arrows-alt" aria-hidden="true"></i>
            </button>

        </div>

        <div class="panel panel-default">
          <div class="panel-heading">Events</div>

          <div class="panel-body log">
            <div v-for="(event,index) in events" :key="index">
              <p><b>Name:</b> {{event.eventName}} <b>Data:</b>{{event.data.text}}</p>
            </div>
          </div>
        </div>
      </div>
    </div>
  </div>

  <div class="col-md-9 panel panel-default">
    <projectTree ref="tree"
                 :identifier="getId"
                 :zoomable="zoomable"
                 :data="data"
                 :node-text="nodeText"
                 :margin-x="Marginx"
                 :margin-y="Marginy"
                 :autoMarginY="autoMarginY"
                 :autoMarginX="autoMarginX"
                 :radius="radius"
                 :type="type"
                 :layout-type="layoutType"
                 :duration="duration"
                 :grid="grid"
                 :deep="(deep < 1 ? 1 : deep)"
                 :hideDeepNodes="hideDeepNodes"
                 :gridMarginY="gridMarginY"
                 :clickableDefaultNodes="clickableDefaultNodes"
                 :sections="['Проект', 'Сборки', 'Стадии', 'Разделы', 'Блоки', 'Задачи']"
                 class="viewport treeclass tree"
                 @clicked="onClick"
                 @expand="onExpand"
                 @onClickNode="onClickNode"
                 @onDblClickNode="onDblClickNode"
                 @contextMenuNode="contextMenuNode"
                 @clickSpace="clickSpace"
                 @onDblClickSpace="onDblClickSpace"
                 @contextMenuSpace="contextMenuSpace"
                 @retract="onRetract"/>
  </div>

  </div>
</template>

<script>
import {projectTree} from '../../src/'
// import data from '../../data/data_project'
let data = null

export default {
  name: 'app',
  data () {
    return {
      type: 'tree',
      layoutType: 'euclidean',
      duration: 500,
      Marginx: -100,
      Marginy: -100,
      autoMarginX: true,
      autoMarginY: true,
      radius: 10,
      nodeText: 'name',
      currentNode: null,
      hideDeepNodes: true,
      zoomable: true,
      isLoading: false,
      events: [],
      grid: true,
      deep: 2,
      gridMarginY: 50,
      clickableDefaultNodes: false,
      data
    }
  },
  components: {
    projectTree
  },
  methods: {
    do (action) {
      if (this.currentNode) {
        this.isLoading = true
        this.$refs['tree'][action](this.currentNode).then(() => { this.isLoading = false })
      }
    },
    getId (node) {
      return node.id
    },
    expandAll () {
      this.do('expandAll')
    },
    collapseAll () {
      this.do('collapseAll')
    },
    showOnly () {
      this.do('showOnly')
    },
    show () {
      this.do('show')
    },
    onClick (evt) {
      this.currentNode = evt.element
      this.onEvent('onClick', evt)
    },
    onExpand (evt) {
      this.onEvent('onExpand', evt)
    },
    onRetract (evt) {
      this.onEvent('onRetract', evt)
    },
    onEvent (eventName, data) {
      this.events.push({eventName, data: data.data})
      // console.log({eventName, data: data})
    },
    resetZoom () {
      this.isLoading = true
      this.$refs['tree'].resetZoom().then(() => { this.isLoading = false })
    },
    onClickNode (e, index, node) {
      console.log(e)
      console.log(index)
      console.log(node)
    },
    onDblClickNode (e, index, node) {
      console.log(e)
      console.log(index)
      console.log(node)
    },
    contextMenuNode (e, index, node) {
      console.log(e)
      console.log(index)
      console.log(node)
    },
    clickSpace (e) {
      console.log(e)
    },
    onDblClickSpace (e) {
      console.log(e)
    },
    contextMenuSpace (e) {
      console.log(e)
    }
  }
}
</script>

<style>
#app {
  font-family: 'Avenir', Helvetica, Arial, sans-serif;
  -webkit-font-smoothing: antialiased;
  -moz-osx-font-smoothing: grayscale;
  text-align: center;
  color: #2c3e50;
  margin-top: 20px;
}

.tree {
  height: 900px;
  width: 100%;
}

.graph-root {
  height: 100%;
  width: 100%;
}

.log  {
  height: 400px;
  overflow-x: auto;
  overflow-y: auto;
  overflow: auto;
  text-align: left;
}

</style>
