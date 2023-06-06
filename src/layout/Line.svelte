<script>
  import * as d3 from 'd3';
  import { onMount } from 'svelte';

  // set the dimensions and margins of the graph
  const margin = {top: 10, right: 30, bottom: 30, left: 60},
      width = 800 - margin.left - margin.right,
      height = 500 - margin.top - margin.bottom;

  let svg;

  function createSvg() {
    svg = d3.select('#my_dataviz')
      .append('svg')
      .attr('width', width + margin.left + margin.right)
      .attr('height', height + margin.top + margin.bottom)
      .append('g')
      .attr('transform', `translate(${margin.left},${margin.top})`);
  }

  onMount(createSvg);

  // function(data) {
  let base_url = window.location.origin;
  d3.csv(base_url+"/public/data/line_chart_data.csv", function(d) {
      return {
        month: d.Month,
        all_items: +d.All_Items,
        food: +d.Food
      };
    }).then(function(data) {

      // console.log(data);

      const x = d3.scaleBand()
        .domain(data.map(d => d.month))
        .range([0, width])
        .padding(0.2);

      svg.append("g")
        .attr("transform", `translate(0, ${height})`)
        .call(d3.axisBottom(x));

      // Add Y axis
      const y = d3.scaleLinear()
        .domain([0, d3.max(data, d => Math.max(d.all_items, d.food))])
        .range([height, 0]);

      svg.append("g")
        .call(d3.axisLeft(y));

      // Add the lines
      svg.append("path")
        .datum(data)
        .attr("fill", "none")
        .attr("stroke", "blue")
        .attr("stroke-width", 1.5)
        .attr("d", d3.line()
            .x(d => x(d.month))
            .y(d => y(d.all_items))
        );

      svg.append("path")
        .datum(data)
        .attr("fill", "none")
        .attr("stroke", "orange")
        .attr("stroke-width", 1.5)
        .attr("d", d3.line()
            .x(d => x(d.month))
            .y(d => y(d.food))
        );  


      // Add circles for each data point
      const circles = svg.selectAll("circle")
  .data(data)
  .enter()
  .append("circle")
  .attr("cx", d => x(d.month))
  .attr("cy", d => y(d.all_items))
  .attr("r", 6)
  .attr("fill", "blue")
  .attr("stroke", "white")
  .attr("stroke-width", 2)
  .attr("opacity", 0.5)
  .on("mouseover", function(d) {
    // console.log("mouseover", d.offsetY);
    d3.select(this)
      .transition()
      .duration(200)
      .attr("r", 10)
      .attr("opacity", 1)
      .style("cursor", "pointer");

    // Add tooltip
    svg.append("text")
      .attr("id", "tooltip")
      .attr("x", x(d.month) - 30)
      .attr("y", y(d.all_items) - 20)
      .attr("font-size", 14)
      .attr("fill", "blue")
      .text(`All Items:${d.offsetY}`);


  }).on("mouseout", function() {
          d3.select(this)
            .transition()
            .duration(200)
            .attr("r", 6)
            .attr("opacity", 0.5);

          // Remove tooltip
          svg.select("#tooltip").remove();
        });

       
      // Add circles for food data
      svg.selectAll("circle2")
        .data(data)
        .enter()
        .append("circle")
        .attr("cx", d => x(d.month))
        .attr("cy", d => y(d.food))
        .attr("r", 6)
        .attr("fill", "orange")
        .attr("stroke", "white")
        .attr("stroke-width", 2)
        .attr("opacity", 0.5)
        .on("mouseover", function(d) {
        

          d3.select(this)
            .transition()
            .duration(200)
            .attr("r", 10)
            .attr("opacity", 1)
            .style("cursor", "pointer");

           

          // Add tooltip
          svg.append("text")
            .attr("id", "tooltip")
            .attr("x", x(d.month) - 30)
            .attr("y", y(d.food) + 20)
            .attr("font-size", 14)
            .attr("fill", "orange")
            .text(`Food: ${d.offsetX}`);

           
            
        })
        .on("mouseout", function() {
          d3.select(this)
            .transition()
            .duration(200)
            .attr("r", 6)
            .attr("opacity", 0.5);

          // Remove tooltip
          svg.select("#tooltip").remove();
        });

      // Add legend
      const legend = svg.append("g")
        .attr("transform", `translate(${width - 70}, 30)`)
        .attr("font-size", 12)
        .attr("font-family", "sans-serif")
        .attr("text-anchor", "end");

      legend.append("text")
        .attr("x", -5)
        .attr("y", -10)
        .text("All Items");

      legend.append("circle")
        .attr("cx", -70)
        .attr("cy", -15)
        .attr("r", 6)
        .attr("fill", "blue");

      legend.append("text")
        .attr("x", -5)
        .attr("y", 10)
        .text("Food");

      legend.append("circle")
        .attr("cx", -70)
        .attr("cy", 5)
        .attr("r", 6)
        .attr("fill", "orange");
  });
</script>


<div id="my_dataviz"></div>