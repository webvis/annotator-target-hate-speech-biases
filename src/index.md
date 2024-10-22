---
toc: false
---

<div class="grid grid-cols-4">
  <div class="aside">
    <h2>Visualizing Hate Speech Biases</h2>
    <div class="disclaimer">WORK IN PROGRESS</div>
    <div class="authors">by Matteo Abrate, Clara Bacciu and Lorenzo Cima<br/>
    <a href="https://figshare.com/articles/software/Visualizing_Hate_Speech_Biases_-_Companion_visualization_for_the_scientific_paper_Human_and_LLM_Biases_in_Hate_Speech_Annotations_A_Socio-Demographic_Analysis_of_Annotators_and_Targets_/27220917?file=49766241">10.6084/m9.figshare.27220917</a></div>
    <hr/>
    <div class="authors"><b>Companion visualization for:</b><br/>
    Giorgi, T., Cima, L., Fagni, T., Avvenuti, M., & Cresci, S. (2024). <i>Human and LLM Biases in Hate Speech Annotations: A Socio-Demographic Analysis of Annotators and Targets.</i> <a href="https://arxiv.org/abs/2410.07991">arXiv preprint arXiv:2410.07991.</a></div>
    <hr/>
    ${filter_rows_input}
    ${filter_cols_input}
    <hr/>
    ${options_input}
    <hr/>
    ${row_clustering_input}
    <hr/>
    ${col_clustering_input}
  </div>
  <div class="chart grid-colspan-3">${resize((width) => matrix_chart({width}))}</div>
</div>

<!-- Load and transform the data -->

```js
const csv = await FileAttachment("data/metrics.csv").csv({typed: true})
let data = csv.map(d => ({...d, matrix_total: d.FF+d.FM+d.FT+d.MF+d.MM+d.MT+d.TF+d.TM+d.TT}))
data.columns = csv.columns.concat(['matrix_total'])
```
```js
const VARS = ['intensity','prevalence','cohen_k','p_value','comments','matrix_total']
const options_input = Inputs.form({
  variable: Inputs.select(VARS, {value: 'intensity', label: "Color by"}),
  size_variable: Inputs.select(VARS, {value: 'prevalence', label: "Scale by"}),
  clustering_variable: Inputs.select(VARS, {value: 'intensity', label: "Cluster by"}),
  cell_size: Inputs.range([4, 40], {value: 12, step: 1, label: "Cell size"})
})
const options = view(options_input)
```

```js
// layout
const margin = ({ top: options.cell_size*6, right: 1, bottom: 10, left: options.cell_size*6 })
const height = rows.length*options.cell_size + margin.top + margin.bottom
```

```js
function matrix_chart({width} = {}) {
  // Create an HTML container to hold the tooltip
  const container = d3.select(
    html`<div style="position:relative;"></div>`
  )

  //const tooltipDiv = container.select(".tooltip")

  // Create the SVG container.
  const svg = container.append("svg")
      .attr("viewBox", [0, 0, width, height])
      .attr("width", width)
      .attr("height", height)
      .attr("style", "max-width: 100%; height: auto;")
  
  // Create the scales.
  const x = d3.scaleBand()
    .domain(d3.range(columns.length))
    .rangeRound([0, columns.length*options.cell_size])

  const y = d3.scaleBand()
    .domain(d3.range(rows.length))
    .rangeRound([0, rows.length*options.cell_size])

  const color = config[options.variable].color_scale

  const size = d3.scaleSqrt() // FIXME this is not centered, nor valid if negative
    .domain([0, Math.max( Math.abs(d3.max(sorted_data, d => d[options.size_variable])), Math.abs(d3.min(sorted_data, d => d[options.size_variable])) )])
    .range([0, options.cell_size])
  
  /*  
  // Append the axes.
  svg.append("g")
      .call(g => g.append("g")
        .attr("transform", `translate(0,${marginTop})`)
        .call(d3.axisTop(x).ticks(null, "d"))
        .call(g => g.select(".domain").remove()))
      .call(g => g.append("g")
        .attr("transform", `translate(0,${height - marginBottom + 4})`)
        .call(d3.axisBottom(x)
            .tickValues([data.year])
            .tickFormat(x => x)
            .tickSize(marginTop + marginBottom - height - 10))
        .call(g => g.select(".tick text")
            .clone()
            .attr("dy", "2em")
            .style("font-weight", "bold")
            .text("Measles vaccine introduced"))
        .call(g => g.select(".domain").remove()));

  svg.append("g")
      .attr("transform", `translate(${marginLeft},0)`)
      .call(d3.axisLeft(y).tickSize(0))
      .call(g => g.select(".domain").remove());

  // Create a cell for each (state, year) value.
  const f = d3.format(",d");
  const format = d => isNaN(d) ? "N/A cases"
      : d === 0 ? "0 cases"
      : d < 1 ? "<1 case"
      : d < 1.5 ? "1 case"
      : `${f(d)} cases`;
  */

  // data cells
  svg.append("g")
      .attr("transform", `translate(${margin.left},${margin.top})`)
      .attr("style", "shape-rendering: crispEdges;")
    .selectAll(".cell")
    .data(sorted_data)
    .join("rect")
      .attr('class', 'cell')
      .attr("y", (d, i) => y(d.row) + (options.cell_size - size(Math.abs(d[options.size_variable])))/2.0)
      .attr("x", (d, i) => x(d.col) + (options.cell_size - size(Math.abs(d[options.size_variable])))/2.0)
      .attr("width", d => size(Math.abs(d[options.size_variable])))
      .attr("height", d => size(Math.abs(d[options.size_variable])))
      .attr("fill", d => d[options.variable] === undefined ? 'transparent' : color(d[options.variable]))

  // interaction cells
  svg.append("g")
      .attr("transform", `translate(${margin.left},${margin.top})`)
    .selectAll(".icell")
    .data(sorted_data)
    .join("rect")
      .attr('class', 'icell')
      .attr("y", (d, i) => y(d.row))
      .attr("x", (d, i) => x(d.col))
      .attr("width", options.cell_size)
      .attr("height", options.cell_size)
      .attr("fill", 'transparent')
      //.call(tooltip, tooltipDiv)
  
  // ticks
  svg.append('g')
    .attr("transform", `translate(${margin.left},${margin.top})`)
    .selectAll('.label')
    .data(sorted_rows)
    .join('text')
      .attr('class', 'label row')
      .style('font-size', `${Math.floor(options.cell_size*0.6)}px`)
      .attr('x', -4)
      .attr("y", (d, i) => y(i)+options.cell_size*0.7)
      .text(d => d.split('_').slice(1).join(' '))
      //.attr("fill", d => core.includes(d) ? 'brown' : '#444')
    .append('title')
      .text(d => d)
  
  svg.append('g')
    .attr("transform", `translate(${margin.left},${margin.top})`)
    .selectAll('.label')
    .data(sorted_cols)
    .join('text')
      .attr('class', 'label col')
      .style('font-size', `${Math.floor(options.cell_size*0.6)}px`)
      .attr('transform', 'rotate(-90)')
      .attr('x', 4)
      .attr("y", (d, i) => x(i)+options.cell_size*0.7)
      .text(d => d.split('_').slice(1).join(' '))
      //.attr("fill", d => core.includes(d) ? 'brown' : '#444')
    .append('title')
      .text(d => d)
  
  return container.node()
}
```

<style>
  body {
    font-family: sans-serif;
  }
  #observablehq-center, #observablehq-main, .grid {
    padding: 0;
    margin: 0;
  }
  .aside {
    margin: 12px;
    background: rgb(241, 239, 229);
    padding: 12px;
  }
  hr {
    padding: 12px;
    padding-top: 6px;
    padding-bottom: 6px;
    margin: 0;
  }
  .authors, .disclaimer {
    font-size: 12px;
  }
  .disclaimer {
    color: red;
  }
  .chart {
    padding: 12px;
  }
  .label {
    user-select: none;
    font-family: sans-serif;
  }
  .label.row {
    text-anchor: end;
  }
  .label:hover {
    fill: magenta !important;
  }
  .icell:hover {
    stroke: magenta;
  }
  .selected {
    font-weight: bold;
  }
</style>


```js
const METHODS = ["complete", "single", "average","median","centroid"]
const row_clustering_input = Inputs.form({
  enabled: Inputs.toggle({label: "Cluster rows", value: true}),
  distance_metric: Inputs.select(Object.keys(distances), {value: "euclidean", label: "Distance Metric"}),
  method: Inputs.select(METHODS, {value: "complete", label: "Cluster Method"})
})
const row_clustering = view(row_clustering_input)
const col_clustering_input = Inputs.form({
  enabled: Inputs.toggle({label: "Cluster columns", value: true}),
  distance_metric: Inputs.select(Object.keys(distances), {value: "euclidean", label: "Distance Metric"}),
  method: Inputs.select(METHODS, {value: "complete", label: "Cluster Method"})
})
const col_clustering = view(col_clustering_input)
```

```js
const filter_rows_input = Inputs.search(all_rows, {placeholder:'Filter annotators'})
const rows = view(filter_rows_input)
const filter_cols_input = Inputs.search(all_columns, {placeholder:'Filter targets'})
const columns = view(filter_cols_input)
```
```js
// preprocess data
const variables = data.columns.filter(d => !(['target', 'annotator'].includes(d)))
const indexed_rows = d3.group(data, d => d.annotator, d => d.target)
const indexed_cols = d3.group(data, d => d.target, d => d.annotator)
const all_rows = Array.from(indexed_rows.keys())
const all_columns = Array.from(indexed_cols.keys())
```

```js
// cluster data
const data_matrix = d3.map(rows, row_k => d3.map(columns, col_k => {
  const v = indexed_rows.get(row_k).get(col_k)
  return v && v[0] // propagate undefined
}))
const value_matrix = d3.map(data_matrix, row => d3.map(row, d => d === undefined ? config[options.clustering_variable].no_data : d[options.clustering_variable])) // propagate undefined

const row_ordering = !row_clustering.enabled ? d3.range(value_matrix.length) : (new hclust.agnes(value_matrix, {distanceFunction: distances[row_clustering.distance_metric], method: row_clustering.method})).indices()
const row_sorted_matrix = row_ordering.map(i => value_matrix[i])
const row_sorted_columns = d3.range(row_sorted_matrix[0].length).map(j => row_sorted_matrix.map(row => row[j]))
const col_ordering = !col_clustering.enabled ? d3.range(value_matrix[0].length) : (new hclust.agnes(row_sorted_columns, {distanceFunction: distances[col_clustering.distance_metric], method: col_clustering.method})).indices()
const sorted_matrix = row_sorted_matrix.map(row => col_ordering.map(j => row[j]) )
const flat_matrix = sorted_matrix.map((row, i) => row.map((value, j) => ({row: i, col: j, value: value == config[options.clustering_variable].no_data ? undefined : value}) )).flat()
const sorted_data = row_ordering.map((row,i) => col_ordering.map((col,j) => {
  let o = data_matrix[row][col] || {}
  o.row = i
  o.col = j
  o.row_label = rows[row]
  o.col_label = columns[col]
  return o
})).flat()
const sorted_rows = row_ordering.map(row => rows[row])
const sorted_cols = col_ordering.map(col => columns[col])
```

```js
// variables config
const config = ({
  cohen_k: {
    no_data: 1,
    color_scale: d3.scaleSequential()
      .domain([-1, 1])
      .interpolator(d3.interpolateRgbBasis(['#740208','#8E4C1E','#dBd756',"#fde725","#35b779","#31688e","#440154"])), // inverted Viridis with additional colors
  },
  intensity: {
    no_data: 0,
    color_scale: d3.scaleSequential()
      .domain([-1, 1])
      .interpolator(t => d3.interpolateRdYlBu(1-t)),
  },
  prevalence: {
    no_data: 0,
    color_scale: d3.scaleSequential()
      .domain([0, 1])
      .interpolator(t => d3.interpolateMagma(1-t)),
  },
  p_value: {
    no_data: 0,
    color_scale: d3.scaleSequential()
      .domain([0, d3.max(data, d => d.p_value)])
      .interpolator(t => d3.interpolateCividis(1-t)),
  },
  comments: {
    no_data: 0,
    color_scale: d3.scaleSequential()
      .domain([0, d3.max(data, d => d.comments)])
      .interpolator(t => d3.interpolateWarm(1-t)),
  },
  matrix_total: {
    no_data: 0,
    color_scale: d3.scaleSequential()
      .domain([0, d3.max(data, d => d.matrix_total)])
      .interpolator(t => d3.interpolateCool(1-t)),
  }
})
```

```js
const tooltip_value_format = d => d && d.toLocaleString()
const tooltip_data = d => ({
  value: tooltip_value_format(d[options.variable]),
  annotator: d.row_label,
  target: d.col_label,
  values: {
    intensity: tooltip_value_format(d.intensity),
    prevalence: tooltip_value_format(d.prevalence),
    cohen_k: tooltip_value_format(d.cohen_k),
    matrix_total: tooltip_value_format(d.matrix_total),
  }
})

function setContents(datum, tooltipDiv) {
  // customize this function to set the tooltip's contents however you see fit
  /*tooltipDiv
    .selectAll("p")
    .data(Object.entries(tooltip_data(datum)).filter(([key, value]) => value !== null && value !== undefined))
    .join("p")
    .html(
      ([key, value]) =>
        `<strong>${key}</strong>: ${
          typeof value === "object" ? value.toLocaleString("en-US") : value
        }`
    );*/
  const d = tooltip_data(datum)
  tooltipDiv.selectAll('p').remove()
  if(d.value !== undefined) {
    tooltipDiv.selectAll(".variables")
      .data(Object.entries(d.values).filter(([key, value]) => value !== null && value !== undefined))
      .join("p")
      .html(
        ([key, value]) =>
          `<span class="${options.variable == key ? 'selected' : ''}">${key}: ${
            typeof value === "object" ? value.toLocaleString("en-US") : value
          }</span>`
      )
      .attr('class', 'variable')
  }
  else {
    tooltipDiv.append('p').html(d.value !== undefined ? `<strong>${d.value}</strong>` : 'No data')
  }
}
```

```js
const distances = {
  'euclidean': ml_distance.distance.euclidean,
  'manhattan': ml_distance.distance.manhattan,
  'chebyshev': ml_distance.distance.chebyshev,
  'jaccard': ml_distance.distance.jaccard,
  'cosine': (a,b) => 1 - ml_distance.similarity.cosine(a,b),
}
```

```js
// dependencies
const hclust = await import("https://cdn.skypack.dev/ml-hclust@3.1.0?min")
const ml_distance = await import("https://cdn.skypack.dev/ml-distance@4.0.1?min")
```