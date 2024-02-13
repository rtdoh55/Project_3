<script>
  import { onMount } from 'svelte';
  import * as d3 from 'd3';

  let accidentsData; // Placeholder for the accident data from CSV
  let filteredData; // Placeholder for filtered data by year

  // Set dimensions for the SVG container
  const svgWidth = 800;
  const svgHeight = 400;
  const margin = { top: 20, right: 30, bottom: 50, left: 60 };

  let startYear = null;
  let endYear = null;

  const filterDataByYear = () => {
    filteredData = accidentsData.filter(d => {
      const year = new Date(d.date_time).getFullYear();
      return year >= startYear && year <= endYear;
    });
    drawChart(filteredData);
  }

  onMount(async () => {
    // Read CSV data
    const response = await fetch('pd_collisions_details_datasd.csv'); // Update with your CSV file path
    const accidentsCsv = await response.text();

    // Parse CSV data
    accidentsData = d3.csvParse(accidentsCsv, d3.autoType);

    // Set initial filtering to show all data
    filteredData = accidentsData;

    // Set initial start and end year
    startYear = d3.min(accidentsData, d => new Date(d.date_time).getFullYear());
    endYear = d3.max(accidentsData, d => new Date(d.date_time).getFullYear());

    // Filter data by year
    filterDataByYear();
  });

  const drawChart = (data) => {
    // Set up scales
    const xScale = d3.scaleTime()
      .domain(d3.extent(data, d => new Date(d.date_time)))
      .range([margin.left, svgWidth - margin.right]);

    const yScale = d3.scaleLinear()
      .domain([0, d3.max(data, d => d.injured + d.killed)])
      .nice()
      .range([svgHeight - margin.bottom, margin.top]);

    // Define line function
    const line = d3.line()
      .x(d => xScale(new Date(d.date_time)))
      .y(d => yScale(d.injured + d.killed));

    // Remove previous chart elements
    d3.select("#line-chart").selectAll("*").remove();

    // Create SVG
    const svg = d3.select("#line-chart")
      .attr("width", svgWidth)
      .attr("height", svgHeight);

    // Draw line
    svg.append("path")
      .datum(data)
      .attr("fill", "none")
      .attr("stroke", "steelblue")
      .attr("stroke-width", 2) // Increase the stroke width
      .attr("d", line);

    // Add markers (circles) at each data point with tooltips
    svg.selectAll("circle")
      .data(data)
      .enter()
      .append("circle")
      .attr("cx", d => xScale(new Date(d.date_time)))
      .attr("cy", d => yScale(d.injured + d.killed))
      .attr("r", 4) // Set the radius of the circles
      .attr("fill", "steelblue")
      .on("mouseover", (event, d) => {
        tooltip.transition()
          .duration(200)
          .style("opacity", .9);
        tooltip.html(`Date: ${new Date(d.date_time).toLocaleDateString()}<br/>Injuries: ${d.injured}<br/>Fatalities: ${d.killed}`)
          .style("left", (event.pageX + 10) + "px")
          .style("top", (event.pageY - 28) + "px");
      })
      .on("mouseout", () => {
        tooltip.transition()
          .duration(500)
          .style("opacity", 0);
      });

    // Add X axis
    svg.append("g")
      .attr("transform", `translate(0, ${svgHeight - margin.bottom})`)
      .call(d3.axisBottom(xScale));

    // Add Y axis
    svg.append("g")
      .attr("transform", `translate(${margin.left}, 0)`)
      .call(d3.axisLeft(yScale));

    // Add tooltip element
    const tooltip = d3.select("body")
      .append("div")
      .attr("class", "tooltip")
      .style("opacity", 0);
  }
</script>

<div>
  <label>Start Year:
    <input type="number" bind:value={startYear} on:input={filterDataByYear}>
  </label>
  <label>End Year:
    <input type="number" bind:value={endYear} on:input={filterDataByYear}>
  </label>
</div>

<svg id="line-chart"></svg>

<style>
  .tooltip {
    position: absolute;
    background-color: white;
    border: 1px solid #ddd;
    border-radius: 4px;
    padding: 6px;
    font-size: 12px;
    pointer-events: none;
  }
</style>
