<script>
  import { onMount } from 'svelte'
  import * as d3 from 'd3'
  import { sankey as d3Sankey } from 'd3-sankey'
  import { select } from 'd3'

  export let data
  export let width = 600
  export let height = 600

  const margin = { top: 10, right: 10, bottom: 10, left: 10 }
  const color = d3.scaleOrdinal(d3.schemeCategory10)

  onMount(() => {
    const sankey = d3Sankey()
      .nodeSort(null)
      .linkSort(null)
      .nodeWidth(15)
      .nodePadding(10)
      .extent([
        [margin.left, margin.top],
        [width - margin.right, height - margin.bottom],
      ])

    const { nodes, links } = sankey(data)

    const svg = select('#chart')
      .append('svg')
      .attr('width', width)
      .attr('height', height)
      .attr('viewBox', `0 0 ${width} ${height}`)
      .attr('preserveAspectRatio', 'xMinYMin meet')
      .append('g')
      .attr('transform', `translate(${margin.left},${margin.top})`)

    svg
      .append('g')
      .selectAll('rect')
      .data(nodes)
      .join('rect')
      .attr('x', (d) => d.x0)
      .attr('y', (d) => d.y0)
      .attr('height', (d) => d.y1 - d.y0)
      .attr('width', (d) => d.x1 - d.x0)
      .attr('fill', (d) => color(d.index))
      .append('title')
      .text((d) => `${d.name}\n${d.value}`)

    const link = svg
      .append('g')
      .attr('fill', 'none')
      .attr('stroke-opacity', 0.5)
      .selectAll('g')
      .data(links)
      .join('g')
      .style('mix-blend-mode', 'multiply')

    link
      .append('path')
      .attr('d', d3.sankeyLinkHorizontal())
      .attr('stroke', (d) => color(d.source.index))
      .attr('stroke-width', (d) => Math.max(1, d.width))
      .attr('stroke-opacity', 0.5)
      .append('title')
      .text((d) => `${d.source.name} → ${d.target.name}\n${d.value}`)

    link
      .append('g')
      .attr('font-family', 'sans-serif')
      .attr('font-size', 10)
      .selectAll('text')
      .data(links)
      .join('text')
      .attr('x', (d) => (d.source.x1 + d.target.x0) / 2)
      .attr('y', (d) => (d.y0 + d.y1) / 2)
      .attr('dy', '1.35em')
      .attr('text-anchor', 'middle')
      .attr('fill', '#000')
      .text((d) => d.value)
  })
</script>

<style>
  .node:hover rect {
    fill: red;
  }
</style>

<div id="chart" />
