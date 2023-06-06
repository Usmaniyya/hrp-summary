<script>
  import { onMount } from 'svelte'
  import * as d3 from 'd3'
  import { select } from 'd3'
  import { sankey, sankeyLinkHorizontal } from 'd3-sankey'

  export let data
  export let width = 400
  export let height = 400

  const sankeyLayout = sankey()
    .nodeWidth(4)
    .nodePadding(7)
    .extent([
      [1, 1],
      [width - 1, height - 1],
    ])

  onMount(() => {
    const { nodes, links } = sankeyLayout(data)

    const svg = select('#sankey_div')
      .append('svg')
      .attr('width', width)
      .attr('height', height)

    // add nodes
    const node = svg
      .selectAll('.node')
      .data(nodes)
      .enter()
      .append('g')
      .attr('class', 'node')
      .attr('transform', (d) => `translate(${d.x0},${d.y0})`)
      .on('mouseover', function (d) {
        // add mouseover event listener
        select(this).select('rect').attr('fill', 'blue')
        tooltip.transition().duration(200).style('opacity', 0.9)
        tooltip
          .html(`${d.name}<br/>${d.value} units`)
          .style('left', `${d3.event.pageX}px`)
          .style('top', `${d3.event.pageY - 28}px`)
      })
      .on('mouseout', function () {
        // add mouseout event listener
        select(this).select('rect').attr('fill', 'orange')
        tooltip.transition().duration(500).style('opacity', 0)
      })

    node
      .append('rect')
      .attr('height', (d) => d.y1 - d.y0)
      .attr('width', (d) => d.x1 - d.x0)
      .attr('fill', 'orange')
      .append('title') // add title element
      .text((d) => d.name) // set title text to node name

    node
      .append('text')
      .attr('x', -6)
      .attr('y', (d) => (d.y1 - d.y0) / 2)
      .attr('dy', '0.35em')
      .attr('text-anchor', 'end')
      .style('font-size', '10px')
      .text((d) => d.name)
      .filter((d) => d.x0 < width / 2)
      .attr('x', 6 + sankeyLayout.nodeWidth())
      .attr('text-anchor', 'start')

    // add links
    svg
      .append('g')
      .attr('class', 'links')
      .selectAll('path')
      .data(links)
      .enter()
      .append('path')
      .attr('d', sankeyLinkHorizontal())
      .attr('stroke', '#000')
      .attr('stroke-opacity', 0.2)
      .attr('fill', 'none')
      .attr('stroke-width', (d) => Math.max(1, d.width))

    // add link labels
    svg
      .selectAll('text.link-label')
      .data(links)
      .enter()
      .append('text')
      .attr('class', 'link-label')
      .attr('x', (d) => (d.source.x1 + d.target.x0) / 2)
      .attr('y', (d) => (d.y1 + d.y0) / 2)
      .attr('text-anchor', 'middle')
      .attr('dy', '0.35em')
      .text((d) => d.value)
      .style('font-size', '10px')
      .style('pointer-events', 'none') // disable pointer events on text elements

    // add the hover effect to nodes
    svg
      .selectAll('.node')
      .on('mouseover', function () {
        select(this)
          .select('rect')
          .transition()
          .duration('50')
          .attr('fill', 'blue')
        svg
          .selectAll('.link-label')
          .filter((d) => d.source === this || d.target === this)
          .transition()
          .duration('50')
          .style('fill', 'orange')
      })
      .on('mouseout', function () {
        select(this)
          .select('rect')
          .transition()
          .duration('50')
          .attr('fill', 'orange')
        svg
          .selectAll('.link-label')
          .filter((d) => d.source === this || d.target === this)
          .transition()
          .duration('50')
          .style('fill', 'black')
      })
  })
</script>

<div id="sankey_div" />
