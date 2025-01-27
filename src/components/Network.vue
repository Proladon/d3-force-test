<template>
  <div
    id="network"
    :style="{ width: svgSize.width + 'px', height: svgSize.height + 'px' }"
  >
    <div
      v-show="linkTextVisible"
      class="linkText"
      :style="linkTextPosition"
      v-text="linkTextContent"
    />
    <svg
      xmlns="http://www.w3.org/2000/svg"
      xmlns:xlink="http://www.w3.org/1999/xlink"
      :width="svgSize.width"
      :height="svgSize.height"
      :style="{ 'background-color': theme.bgcolor }"
      @click="clickEle"
      @mouseover.prevent="svgMouseover"
      @mouseout="svgMouseout"
      @contextmenu.prevent="$emit('clear')"
    >
      <g id="container">
        <!-- links and link-text 注：先绘制邊 -->
        <g>
          <g v-for="link in links" :key="link.index">
            <line
              :class="`${link[linkTypeKey]} ${link.selected} link element`"
              :stroke="theme.linkStroke"
              :stroke-width="linkWidth"
            />
            <!-- dx dy 文字左下角坐标 -->
            <text
              v-if="showLinkText"
              dx="0"
              dy="0"
              class="link-text"
              :fill="theme.textFill"
              :font-size="linkTextFrontSize"
            >
              {{ link[linkTextKey] }}
            </text>
          </g>
        </g>

        <!-- node and node-text -->
        <g id="node-group">
          <g v-for="node in nodes" :key="node.index">
            <circle
              :fill="nodeColor(node[nodeTypeKey])"
              :stroke-width="highlightNodes.indexOf(node.id) == -1 ? 1.5 : 10"
              :stroke="
                highlightNodes.indexOf(node.id) == -1
                  ? theme.nodeStroke
                  : 'gold'
              "
              :class="`layer-${node[nodeTypeKey]} ${
                node.showText ? 'selected' : ''
              } node element`"
              :r="nodeSize"
            />

            <!-- 節點按鈕 -->
            <foreignObject
              v-show="pinned.includes(node.index)"
              xmlns="http://www.w3.org/2000/svg"
              class="rect flex"
              width="15"
              height="15"
            >
              <vs-button class="delete-btn" color="danger" @click="$emit('deleteNode', node)">X</vs-button>
            </foreignObject>
            <!-- 下方text v-show="node.showText" -->
            <text
              :dx="nodeSize + 5"
              dy="0"
              class="node-text"
              :class="{'focus-text': pinned.includes(node.index)}"
              :fill="theme.textFill"
              :font-size="nodeTextFontSize"
            >
              {{ node[nodeTextKey] }}
            </text>
          </g>
          <g />
        </g>
      </g>
    </svg>
  </div>
  <!-- </div> -->
</template>

<script>
import * as d3 from 'd3'
import { mapState } from 'vuex'
// import * as d3Force from 'd3-force'
// import * as d3Zoom from 'd3-zoom'
// import * as d3Scale from 'd3-scale'
// import * as d3Selection from 'd3-selection'
// import * as d3Drag from 'd3-drag'
// import * as d3ScaleChromatic from 'd3-scale-chromatic'

// import d3SelectionMulti from "d3-selection-multi";

// const d3 = Object.assign({}, d3Force, d3Zoom, d3Scale, d3Selection, d3Drag)

// 元素的 classList 是 DOMTokenList
DOMTokenList.prototype.indexOf = Array.prototype.indexOf

export default {
  name: 'network',
  props: {
    nodeList: Array,
    linkList: Array,
    // node
    nodeSize: {
      type: Number,
      default: 14,
    },
    nodeTextKey: {
      type: String,
      default: 'id',
    },
    nodeTypeKey: {
      type: String,
      default: 'group',
    },
    nodeTextFontSize: {
      type: Number,
      default: 14,
    },
    // link
    linkWidth: {
      type: Number,
      default: 1.5,
    },
    showLinkText: {
      type: Boolean,
      default: false,
    },
    linkTextKey: {
      type: String,
      default: 'value',
    },
    linkTypeKey: {
      type: String,
      default: 'type',
    },
    linkTextFrontSize: {
      type: Number,
      default: 10,
    },
    linkDistance: {
      type: Number,
      default: 50,
    },
    // svg
    svgSize: {
      type: Object,
      default: () => {
        return {
          width: window.innerWidth,
          height: window.innerHeight,
        }
      },
    },
    svgTheme: {
      type: String,
      default: 'light', // dark or light
    },
    bodyStrength: {
      type: Number,
      default: -100,
    },
    // others
    highlightNodes: {
      type: Array,
      default: () => {
        return []
      },
    },
  },
  data () {
    return {
      selection: {
        links: [],
        nodes: [],
      },
      pinned: [], // 被釘住的節點的下標
      force: null,
      zoom: d3.zoom(),
      nodeColor: d3.scaleOrdinal(d3.schemeTableau10),
      linkTextVisible: false,
      linkTextPosition: {
        top: 0,
        left: 0,
      },
      linkTextContent: '',
    }
  },
  computed: {
    ...mapState('layer', ['activatedLayer']),
    nodes () {
      // 節點去重
      let nodes = this.nodeList.slice()
      const nodeIds = []
      nodes = nodes.filter((node) => {
        if (nodeIds.indexOf(node.id) === -1) {
          nodeIds.push(node.id)
          return true
        } else {
          return false
        }
      })
      return nodes
    },
    links () {
      return this.linkList
    },
    theme () {
      if (this.svgTheme === 'light') {
        return {
          bgcolor: '#353f52',
          nodeStroke: 'white',
          linkStroke: 'lightgray',
          textFill: 'lightgray',
        }
      } else {
        // dark
        return {
          bgcolor: 'black',
          nodeStroke: 'white',
          linkStroke: 'rgba(200,200,200)',
          textFill: 'white',
        }
      }
    },
  },
  watch: {
    activatedLayer () {
      d3.selectAll('circle').attr('r', 10)
      d3.selectAll(`.layer-${this.activatedLayer}`).attr('r', 15)
    },
    bodyStrength: function () {
      this.force.stop()
      this.initData()
      this.$nextTick(function () {
        this.initDragTickZoom()
        this.force.restart()
      })
    },
    linkDistance: function () {
      this.force.stop()
      this.initData()
      this.$nextTick(function () {
        this.initDragTickZoom()
        this.force.restart()
      })
    },
    nodes: function () {
      this.force.stop()
      this.initData()
      this.$nextTick(function () {
        // 以下這個函數重新在 node 上調用了拖拽
        // 只有在 mounted 後才有用
        // 所以要使用 $nextTick
        this.initDragTickZoom()
        this.force.restart()
      })
    },
  },
  created () {
    this.initData()
  },
  mounted () {
    this.initDragTickZoom()
  },
  methods: {
    initData () {
      this.force = d3
        .forceSimulation(this.nodes)
        .force(
          'link',
          d3
            .forceLink(this.links)
            .id((d) => d.id)
            .distance(this.linkDistance),
        )
        .force('charge', d3.forceManyBody().strength(this.bodyStrength)) // The strength of the attraction or repulsion
        .force(
          'center',
          d3.forceCenter(this.svgSize.width / 2, this.svgSize.height / 2),
        )

      // console.log(this.nodes);
      // console.log(this.links);
    },
    initDragTickZoom () {
      // 給節點添加拖拽
      d3.selectAll('.node').call(this.drag(this.force))
      this.force.on('tick', () => {
        // 更新連線座標
        d3.selectAll('.link')
          .data(this.links)
          .attr('x1', (d) => d.source.x)
          .attr('y1', (d) => d.source.y)
          .attr('x2', (d) => d.target.x)
          .attr('y2', (d) => d.target.y)
        // 更新節點座標
        d3.selectAll('.rect')
          .data(this.nodes)
          .attr('x', (d) => d.x + 10)
          .attr('y', (d) => d.y - 30)
        d3.selectAll('.node')
          .data(this.nodes)
          .attr('cx', (d) => {
            return d.x
          })
          .attr('cy', (d) => d.y)
        // 更新文字座標
        d3.selectAll('.node-text')
          .data(this.nodes)
          .attr('x', (d) => d.x)
          .attr('y', (d) => d.y)
        d3.selectAll('.link-text')
          .data(this.links)
          .attr('x', (d) => (d.source.x + d.target.x) / 2)
          .attr('y', (d) => (d.source.y + d.target.y) / 2)
      })

      // 初始化 zoom
      this.zoom.scaleExtent([0.1, 4]).on('zoom', this.zoomed)

      d3.select('svg').call(this.zoom).on('dblclick.zoom', null)
    },
    clickLink (e) {
      this.$emit('clickLink', e, e.target.__data__)
    },
    clickNode (e) {
      let focus = false
      if (this.pinned.length === 0) {
        // 如果都還沒有被點選的
        this.pinnedState(e)
        focus = true
      } else {
        // 如果已經有被點選的
        // d3.selectAll('.element').style('opacity', 0.2)
        this.pinned = []
        focus = false
      }
      this.$emit('clickNode', e, e.target.__data__)
      if (!focus) this.$emit('deFocus')
    },
    clickEle (e) {
      if (e.target.tagName === 'circle') {
        this.clickNode(e)
      } else if (e.target.tagName === 'line') {
        this.clickLink(e)
      }
    },
    svgMouseover (e) {
      if (e.target.nodeName === 'circle') {
        if (this.pinned.length === 0) {
          this.selectedState(e)
        }
        // 强制刷新
        this.$forceUpdate()
        this.$emit('hoverNode', e, e.target.__data__)
      } else if (e.target.nodeName === 'line') {
        // 顯示連線關係文字
        this.linkTextPosition = {
          left: e.clientX + 'px',
          top: e.clientY - 50 + 'px',
        }
        this.linkTextContent = e.target.__data__[this.linkTextKey]
        this.linkTextVisible = true
        this.$emit('hoverLink', e, e.target.__data__)
      }
    },
    svgMouseout (e) {
      this.linkTextVisible = false
      if (e.target.nodeName === 'circle') {
        if (this.pinned.length === 0) {
          this.noSelectedState(e)
        }
        // 强制刷新
        this.$forceUpdate()
      }
    },
    selectedState (e) {
      // 節點自身顯示文字、增加 selected class、添加進 selection
      e.target.__data__.showText = true
      e.target.classList.add('selected')
      this.selection.nodes.push(e.target.__data__)
      this.$store.commit('network/ADD_REF_NODES', e.target.__data__)
      // 周圍節點顯示文字、邊和節點增加 selected class、添加進 selection
      this.lightNeibor(e.target.__data__)
      // 除了 selected 的其餘節點透明度減小
      d3.selectAll('.element').style('opacity', 0.2) // hover
    },
    noSelectedState (e) {
      // 節點自身不顯示文字、移除 selected class
      e.target.__data__.showText = false
      // e.target.classList.remove("selected");
      // 周圍節點不顯示文字、邊和節點移除 selected class
      this.darkenNerbor()
      // 所有節點透明度恢復
      d3.selectAll('.element').style('opacity', 1)
    },
    pinnedState (e) {
      this.pinned.push(e.target.__data__.index)
      d3.selectAll('.element').style('opacity', 0.05) // click
    },
    lightNeibor (node) {
      this.links.forEach((link) => {
        if (link.source.index === node.index) {
          link.selected = 'selected'
          this.selection.links.push(link)
          this.selection.nodes.push(link.target)
          this.$store.commit('network/ADD_REF_NODES', link.target)
          this.lightNode(link.target)
        }
        if (link.target.index === node.index) {
          link.selected = 'selected'
          this.selection.links.push(link)
          this.selection.nodes.push(link.source)
          this.$store.commit('network/ADD_REF_NODES', link.source)
          this.lightNode(link.source)
        }
      })
    },
    lightNode (selectedNode) {
      this.nodes.forEach((node) => {
        if (node.index === selectedNode.index) {
          node.showText = true
        }
      })
    },
    darkenNerbor () {
      this.links.forEach((link) => {
        this.selection.links.forEach((selectedLink) => {
          if (selectedLink.id === link.id) {
            link.selected = ''
          }
        })
      })
      this.nodes.forEach((node) => {
        this.selection.nodes.forEach((selectednode) => {
          if (selectednode.id === node.id) {
            node.showText = false
          }
        })
      })
      // 清空 selection
      this.selection.nodes = []
      this.$store.commit('network/SET_REF_NODES', [])
      this.selection.links = []
    },
    zoomed () {
      // 縮放中：以鼠標所在的位置為中心
      d3.select('#container').attr(
        'transform',
        'translate(' +
          d3.event.transform.x +
          ',' +
          d3.event.transform.y +
          ') scale(' +
          d3.event.transform.k +
          ')',
      )
    },
    drag (simulation) {
      function dragstarted (d) {
        if (!d3.event.active) simulation.alphaTarget(0.3).restart()
        // if (!d3.event.active) simulation.alpha(0.3).restart()
        d.fx = d.x
        d.fy = d.y
      }

      function dragged (d) {
        d.fx = d3.event.x
        d.fy = d3.event.y
      }

      function dragended (d) {
        if (!d3.event.active) simulation.alphaTarget(0)
        // if (!d3.event.active) simulation.alpha(0)
        d.fx = null
        d.fy = null
      }

      return d3
        .drag()
        .on('start', dragstarted)
        .on('drag', dragged)
        .on('end', dragended)
    },
  },
}
</script>

<style scoped lang="postcss">
svg {
  /* border-radius: 5px; */
  background-color: darkblue;
}

.element {
  transition: opacity 0.5s ease;
}
.selected {
  opacity: 1 !important;
  stroke: rgb(110, 231, 183);
  stroke-width: 2px;
}
.node,
.link {
  @apply cursor-pointer;
}
.linkText {
  @apply absolute z-10 text-white p-[10px] rounded-[10px];
  background-color: rgba(75, 75, 75, 0.596);
}

.delete-btn {
  @apply !w-[15px] !h-[15px] text-[12px] !p-0;
}

.node-text {
  transition: ease-in-out .3s;
}
.focus-text {
  @apply  text-[18px] z-[99];
  fill: rgb(255, 229, 79);
  transition: ease-in-out .3s;
}
</style>
