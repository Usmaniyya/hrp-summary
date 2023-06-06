<script>
  import { onMount } from 'svelte'
  import * as d3 from 'd3'
  import { select } from 'd3'
  export let data
  export let width = 600
  export let height = 600

  const margin = { top: 10, right: 10, bottom: 10, left: 10 }
  const color = d3.scaleOrdinal(d3.schemeCategory10)

  onMount(() => {
    const pack = (data) =>
      d3
        .pack()
        .size([
          width - margin.left - margin.right,
          height - margin.top - margin.bottom,
        ])
        .padding(3)(
        d3
          .hierarchy(data)
          .sum((d) => d.value)
          .sort((a, b) => b.value - a.value),
      )

    const root = pack(data)

    const svg = select('#chart')
      .append('svg')
      .attr('width', width)
      .attr('height', height)
      .attr('viewBox', `0 0 ${width} ${height}`)
      .attr('preserveAspectRatio', 'xMinYMin meet')
      .append('g')
      .attr('transform', `translate(${margin.left},${margin.top})`)

    const node = svg
      .selectAll('g')
      .data(
        d3
          .pack()
          .size([
            width - margin.left - margin.right,
            height - margin.top - margin.bottom,
          ])
          .padding(3)(
            d3
              .hierarchy(data)
              .sum((d) => d.value)
              .sort((a, b) => b.value - a.value),
          )
          .descendants(),
      )
      .enter()
      .append('g')
      .attr('transform', (d) => `translate(${d.x},${d.y})`)

    node
      .append('circle')
      .attr('r', (d) => d.r)

      .style('fill', (d) => color(d.value))

      .style('stroke', (d) => (d.children ? color(d.depth) : 'white'))
      .on('mouseover', function (d) {
        // d3.select(this).transition().duration(200).attr('r', d.r * 1.1);
        d3.select(this).style('fill', 'aliceblue')
      })
      .on('mouseout', function (d) {
        // d3.select(this).transition().duration(200).attr('r', d.r);
        d3.select(this).style('fill', (d) => color(d.value))
      })

    node
      .append('text')
      .attr('x', 0)
      .attr('y', 0)
      .style('text-anchor', 'middle')
      .style('font-size', '8px')
      .style('fill', 'black')
      .style('background-color', (d) => color(d.value))

      .text((d) => (d.children ? '' : d.data.name))

    node.append('title').text((d) => `${d.data.name}\n${d.data.value}`)
  })
</script>

<div id="chart" />
