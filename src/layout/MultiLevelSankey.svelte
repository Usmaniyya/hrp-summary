<script>
    import { onMount } from 'svelte';
    import * as d3 from 'd3';
    import * as d3Sankey from 'd3-sankey';
  
    let svgNode;

onMount(() => {
  const svg = d3.select(svgNode);

  const width = 800;
  const height = 600;
  
  const margin = { top: 10, right: 10, bottom: 10, left: 10 };
  const innerWidth = width - margin.left - margin.right;
  const innerHeight = height - margin.top - margin.bottom;
  
  const sankey = d3Sankey.sankey()
    .nodeWidth(15)
    .nodePadding(10)
    .size([innerWidth, innerHeight]);
  
  const path = sankey.link();

  sankey
    .nodes(data.nodes)
    .links(data.links)
    .layout(32);

  const link = svg.append('g')
    .attr('class', 'links')
    .selectAll('path')
    .data(data.links)
    .enter()
    .append('path')
    .attr('d', path)
    .style('stroke-width', (d) => Math.max(1, d.width))
    .style('stroke', (d) => {
      const color = d3.scaleOrdinal(d3.schemeCategory10);
      return color(d.source.id.replace(/ .*/, ''));
    })
    .sort((a, b) => b.width - a.width);
  
  link.append('title')
    .text((d) => `${d.source.id} → ${d.target.id}\n${d.value}`);
  
  const node = svg.append('g')
    .attr('class', 'nodes')
    .selectAll('rect')
    .data(data.nodes)
    .enter()
    .append('rect')
    .attr('x', (d) => d.x0)
    .attr('y', (d) => d.y0)
    .attr('height', (d) => d.y1 - d.y0)
    .attr('width', (d) => d.x1 - d.x0)
    .style('fill', (d) => {
        const color = d3.scaleOrdinal(d3.schemeCategory10);
      return color(d.source.name);
    })

    .attr('opacity', 0.8);

  // Add the link titles
  g.append('title').text((d) => `${d.source.name} → ${d.target.name}\n${format(d.value)}`);

  // Append the nodes to the diagram
  node = svg
    .append('g')
    .selectAll('.node')
    .data(data.nodes)
    .enter()
    .append('g')
    .attr('class', 'node')
    .attr('transform', (d) => `translate(${d.x},${d.y})`)
    .call(
      d3
        .drag()
        .subject((d) => d)
        .on('start', () => {
          this.parentNode.appendChild(this);
        })
        .on('drag', dragmove),
    );

  // Add the rectangles to the nodes
  node
    .append('rect')
    .attr('height', (d) => d.dy)
    .attr('width', sankey.nodeWidth())
    .style('fill', (d) => {
      const color = d3.scaleOrdinal(d3.schemeCategory10);
      return color(d.name.replace(/ .*/, ''));
    })
    .style('stroke', (d) => {
      return d3.rgb(d.color).darker(2);
    })
    .append('title')
    .text((d) => `${d.name}\n${format(d.value)}`);

  // Add the labels to the nodes
  node
    .append('text')
    .attr('x', -6)
    .attr('y', (d) => d.dy / 2)
    .attr('dy', '.35em')
    .attr('text-anchor', 'end')
    .attr('transform', null)
    .text((d) => d.name)
    .filter((d) => d.x < width / 2)
    .attr('x', 6 + sankey.nodeWidth())
    .attr('text-anchor', 'start');
});


  function dragmove(d) {
    d3.select(this).attr('transform', `translate(${(d.x = Math.max(0, Math.min(width - d.dx, d3.event.x)))},${(d.y = Math.max(0, Math.min(height - d.dy, d3.event.y)))})`);
    sankey.relayout();
    link.attr('d', sankey.linkHorizontal());
  }
</script>


