<template>
  <div id="tree-container" :style="{ width: 600, height: 600 }"></div>
</template>
<script>
import * as d3 from 'd3';
import * as d3h from 'd3-hierarchy';
import originData from './flare-2.json';

export default {
  name: 'Tree',
  props: ['options'],
  data() {
    return {
      tree: null,
      g: null,
      zoomsvg: null,
      chart: null,
      tooltip: null,
      click: null,
      panSpeed: 30,
      duration: 750,
      expandAll: false,
      title: null,
      titleText: 'Tree',
      titleRectWidth: 500,
      titleRectHeight: 50,
      gLinkStroke: { color: '#000', opacity: 0.8, width: 2 },
      text: { fontSize: 15, fontFamily: 'Arial', fontColor: '#000' },
      node: {
        size: 5,
        borderWidth: 2,
        nonLeafNodeFill: '#fff',
        leafNodeFill: '#888',
      },
      tooltipPadding : {top: 5, right: 5, bottom: 5, left: 5},
      treeLayout: { width: 25, height: 300, direction: 'left' },
      width: 460,
      height: 460,
      chartPadding: { top: 80, right: 80, bottom: 80, left: 80 },
      margin: { top: 10, right: 120, bottom: 10, left: 40 },
      data: {}
    };
  },
  mounted() {
    // 这里会在实例被挂载后调用
    // 初始化图表
    this.initTree();
  },
  watch: {
    // watch的作用可以监控一个值的变换，并调用因为变化需要执行的方法。可以通过watch动态改变关联的状态。
    // 这里是一些可被修改的配置项，例如图表标题内容、标题是否显示等
    'options.titleText': {
      handler() {
        this.titleText = this.options.titleText;
        this.title.select('text').text(this.titleText);
      },
    },
    'options.titleIsShow': {
      handler() {
        if (this.options.titleIsShow) {
          this.title.attr('style', 'display: block');
        } else {
          this.title.attr('style', 'display: none');
        }
      },
    },
    'options.titleBackground': {
      handler() {
        this.title
          .select('rect')
          .attr('fill', this.options.titleBackground);
      },
    },
    'options.titleFontColor': {
      handler() {
        this.title
          .select('text')
          .attr('fill', this.options.titleFontColor);
      },
    },
    'options.titleTextPosition': {
      handler() {
        // 修改text相对标题rect的位置,来更改文本对齐方式
        switch (this.options.titleTextPosition) {
          case 'center':
            this.title
              .select('text')
              .attr('text-anchor', 'middle')
              .attr('x', this.titleRectWidth/2);
            break;
          case 'left':
            this.title
              .select('text')
              .attr('text-anchor', 'start')
              .attr('x', 10);
            break;
          case 'right':
            this.title
              .select('text')
              .attr('text-anchor', 'end')
              .attr('x', 690);
            break;
          default:
            break;
        }
      },
    },
    'options.titleFontFamily': {
      handler() {
        this.title
          .select('text')
          .attr('font-family', this.options.titleFontFamily);
      },
    },
    'options.titleFontSize': {
      handler() {
        this.title
          .select('text')
          .attr('font-size', this.options.titleFontSize);
      },
    },
    'options.nonLeafNodeFill': {
      handler() {
        this.node.nonLeafNodeFill = `${this.options.nonLeafNodeFill}`;
        this.gNode
          .selectAll('circle')
          .attr('fill', (d) =>
            d._children
              ? `${this.node.nonLeafNodeFill}`
              : `${this.node.leafNodeFill}`
          );
      },
    },
    'options.leafNodeFill': {
      handler() {
        this.node.leafNodeFill = `${this.options.leafNodeFill}`;
        this.gNode
          .selectAll('circle')
          .attr('fill', (d) =>
            d._children
              ? `${this.node.nonLeafNodeFill}`
              : `${this.node.leafNodeFill}`
          );
      },
    },
    'options.textIsShow': {
      handler() {
        if (this.options.textIsShow) {
          this.gNode.selectAll('text').attr('style', 'display: block');
        } else {
          this.gNode.selectAll('text').attr('style', 'display: none');
        }
      },
    },
    'options.textFontFamily': {
      handler() {
        this.text.fontFamily = this.options.textFontFamily;
        this.gNode
          .selectAll('text')
          .attr('font-family', this.text.fontFamily);
      },
    },
    'options.textFontColor': {
      handler() {
        this.text.fontColor = `${this.options.textFontColor}`;
        this.gNode.selectAll('text').attr('fill', this.text.fontColor);
      },
    },
    'options.linkStrokeColor': {
      handler() {
        this.gLinkStroke.color = `${this.options.linkStrokeColor}`;
        this.gLink.attr('stroke', this.gLinkStroke.color);
      },
    },
    'options.linkStrokeOpacity': {
      handler() {
        this.gLinkStroke.opacity = this.options.linkStrokeOpacity / 10;
        this.gLink.attr('stroke-opacity', this.gLinkStroke.opacity);
      },
    },
    'options.linkStrokeWidth': {
      handler() {
        this.gLinkStroke.width = this.options.linkStrokeWidth;
        this.gLink.attr('stroke-width', this.gLinkStroke.width);
      },
    },
    'options.tooltipIsShow': {
      handler(){
        if(this.options.tooltipIsShow){
          this.tooltip.attr('style', 'display: block');
        }else{
          this.tooltip.attr('style', 'display: none');
        }
      }
    },
    'options.tooltipColor': {
      handler() {
        this.tooltip.select('rect')
          .attr('fill', this.options.tooltipColor);
      }
    },
    'options.tooltipBorder': {
      handler() {
        this.tooltip.select('rect')
          .attr('style', `stroke-width: ${this.options.tooltipBorder}`);
      }
    },
    'options.tooltipBorderRadius': {
      handler() {
        this.tooltip.select('rect')
          .attr('rx', `${this.options.tooltipBorderRadius}`)
          .attr('ry', `${this.options.tooltipBorderRadius}`);
      }
    },
  },
  methods: {
    initTree() {
      // 选择svg容器
      d3.select('#tree-container')
        .style('width', '100rem')
        .style('height', '100rem');

      // 添加base svg
      this.svg = d3
        .select('#tree-container')
        .append('svg')
        .attr('style', 'background: white')
        .attr('width', '100rem')
        .attr('height', '100rem')
        .call(
          d3.zoom().on('zoom', () => {
            console.log(d3.event.transform);
            this.zoomsvg.attr('transform', d3.event.transform);
          })
        );
      this.zoomsvg = this.svg.append('g');
      this.g = this.zoomsvg
        .append('g')
        .attr('transform', `translate(${this.width / 2},${this.height})`);
        // .on('click',()=>{
        //   console.log('svg');
        // })
        // ;
      // 添加links svg
      this.gLink = this.g
        .append('g')
        .attr('fill', 'none')
        .attr('stroke', this.gLinkStroke.color)
        .attr('stroke-opacity', this.gLinkStroke.opacity)
        .attr('stroke-width', this.gLinkStroke.width);
      // 添加nodes svg
      this.gNode = this.g
        .append('g')
        .attr('cursor', 'pointer')
        .attr('pointer-events', 'all');
      // load data
      this.treeRoot = d3h.hierarchy(originData);
      this.treeRoot.x0 = this.treeLayout.width / 2;
      this.treeRoot.y0 = 0;
      this.treeRoot.descendants().forEach((d, i) => {
        d.id = i;
        d._children = d.children;
        if (d.depth>=2 || (d.depth && d.data.name.length !== 7 )) d.children = null;
      });
      // 初始化tree
      this.tree = d3.tree();
      this.tree.nodeSize([this.treeLayout.width, this.treeLayout.height]);
      // console.log('init');
      this.tree(this.treeRoot);
      this.updateTree(this.treeRoot);
      // 添加图表标题
      this.title = d3
        .select('#tree-container svg')
        .append('g')
        .attr('transform', 'translate(0,0)')
        .attr('style', 'display: none'); // 默认不显示
      // 标题背景框
      this.title
        .append('rect')
        .attr('class', 'title')
        .attr('width', this.titleRectWidth)
        .attr('height', this.titleRectHeight)
        .attr('fill', '#E3E3E3')
        .attr('x', '0')
        .attr('y', '0');
      // 标题文本
      this.title
        .append('text')
        .text(this.titleText)
        .attr('x', this.titleRectWidth/2)
        .attr('y', this.titleRectHeight-5)
        .attr('text-anchor', 'middle')
        .attr('fill', '#000');
      this.tooltip = this.g
        .append('g')
        .attr('class', 'tooltip')
        .attr('opacity', 0);
      this.tooltip.append('rect')
        .attr('fill', '#eeeeee')
        .attr('width', 150)
        .attr('height', 23)
        .attr('rx', 0)
        .attr('ry', 0)
        .attr('stroke', 'black')
        .attr('style', 'stroke-width:1');
      this.tooltip.append('text')
        .attr('font-size',12)
        .attr('font-color', '#000')
        .attr('transform', `translate(${this.tooltipPadding.left},${this.tooltipPadding.top+12})`);
    },
    /**
     * @description: update tree layout when needed
     * @param {*} source
     * @return {*}
     */
    updateTree(source) {
      // console.log('update');
      this.linkH = d3
        .linkHorizontal()
        .x((d) => d.y)
        .y((d) => d.x);
      this.g
        .attr('writing-mode', 'horizontal-tb');

      const nodes = this.treeRoot.descendants().reverse();
      const links = this.treeRoot.links();
      // Compute the new tree layout.
      this.tree(this.treeRoot);
      // transition setting
      const transition = this.g
        .transition().duration(this.duration);
      // Update the nodes…
      const node = this.gNode.selectAll('g').data(nodes, (d) => d.id);

      // Enter any new nodes at the parent's previous position.
      const nodeEnter = node
        .enter()
        .append('g')
        .attr('transform', `translate(${source.y0},${source.x0})`)
        .attr('fill-opacity', 0)
        .attr('stroke-opacity', 0)
        .on('click', (d) => {
          if (d3.event.defaultPrevented) return; // click suppressed
          d.children = d.children ? null : d._children;
          this.updateTree(d);
        });

      nodeEnter
        .append('circle')
        .attr('class', 'node')
        .attr('r', this.node.size)
        .attr('fill', (d) =>
          d._children ? this.node.nonLeafNodeFill : this.node.leafNodeFill
        )
        .attr('stroke', this.color)
        .attr('stroke-width', this.node.borderWidth);

      nodeEnter
        .append('text')
        .attr('x', (d) => (d._children ? -6 : 6))
        .attr('dy', '.31em')
        .attr('class', 'nodeText')
        .attr('text-anchor', (d) => (d._children ? 'end' : 'start'))
        .text((d) => d.data.name)
        .attr('font-size', this.text.fontSize)
        .attr('font-family', this.text.fontFamily)
        .attr('fill', this.text.fontColor)
        .clone(true)
        .lower()
        .attr('stroke-linejoin', 'round')
        .attr('stroke-width', 3)
        .attr('stroke', 'white');
      
      this.gNode.selectAll('text')
        .on('mouseover', function(d){
          if(! d._children){
            const w = this.getBBox().width*(d.data.name.length+5)/d.data.name.length;
            const x = d.y0 + w;
            const y = d.x0 - 12;
            d3.select('.tooltip')
              .attr('transform', `translate(${x},${y})`)
              .attr('opacity', 0.7);
            d3.select('.tooltip').select('rect')
              .attr('width', w);
            d3.select('.tooltip').select('text')
              .text(`(${d.data.name}: ${d.data.value})`);
          }
        })
        .on('mouseout', function(d){
          if(! d._children){
            d3.select('.tooltip')
              .attr('opacity',0);
          }
        });

      if (this.options.textIsShow) {
        this.gNode.selectAll('text').attr('style', 'display: block');
      } else {
        this.gNode.selectAll('text').attr('style', 'display: none');
      }
      // Transition nodes to their new position.

      node
        .merge(nodeEnter)
        .transition(transition)
        .attr('transform', (d) => `translate(${d.y},${d.x})`)
        .attr('fill-opacity', 1)
        .attr('stroke-opacity', 1);

      // Transition exiting nodes to the parent's new position.
      node
        .exit()
        .transition(transition)
        .remove()
        .attr('transform', `translate(${source.y0},${source.x0})`)
        .attr('fill-opacity', 0)
        .attr('stroke-opacity', 0);

      // update the links...
      const link = this.gLink.selectAll('path').data(links, (d) => d.target.id);
      // Enter any new links at the parent's previous position.
      const linkEnter = link
        .enter()
        .append('path')
        .attr('d', () => {
          const o = { x: source.x0, y: source.y0 };
          return this.linkH({ source: o, target: o });
        });

      // Transition links to their new position.
      link
        .merge(linkEnter)
        .transition(transition)
        .attr('d', this.linkH);

      // Transition exiting nodes to the parent's new position.
      link
        .exit()
        .transition(transition)
        .remove()
        .attr('d', () => {
          const o = { x: source.x, y: source.y };
          return this.linkH({ source: o, target: o });
        });
      // Stash the old positions for transition.
      nodes.forEach((d) => {
        d.x0 = d.x;
        d.y0 = d.y;
      });
    },
    color(node){
      const color = d3.scaleOrdinal()
        .domain(this.treeRoot.descendants().filter(d => d.depth<=1).map(d => d.data.name))
        .range(d3.schemeCategory10);
      if(node.depth===0){
        return color(node.data.name);
      }
      while(node.depth>1){
        node = node.parent;
      }
      return color(node.data.name);
    }
  },
};
</script>
<style scoped>
text {
  background: '#000';
}
#tree-container {
  overflow: hidden;
}
</style>
