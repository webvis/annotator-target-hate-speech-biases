---
toc: false
---

<div class="grid grid-cols-4">
  <div class="aside">
    <h2>Visualizing Hate Speech Biases</h2>
    <!--<div class="disclaimer">WORK IN PROGRESS</div>-->
    <div class="authors">by Matteo Abrate, Clara Bacciu and Lorenzo Cima<br/>
    <a href="https://figshare.com/articles/software/Visualizing_Hate_Speech_Biases_-_Companion_visualization_for_the_scientific_paper_Human_and_LLM_Biases_in_Hate_Speech_Annotations_A_Socio-Demographic_Analysis_of_Annotators_and_Targets_/27220917?file=49766241">10.6084/m9.figshare.27220917</a></div>
    <hr/>
    <div class="authors"><b>Companion visualization for:</b><br/>
    Giorgi, T., Cima, L., Fagni, T., Avvenuti, M., & Cresci, S. (2025, June). <i>Human and LLM biases in hate speech annotations: A socio-demographic analysis of annotators and targets.</i> In Proceedings of the International AAAI Conference on Web and Social Media. DOI: <a href="https://doi.org/10.1609/icwsm.v19i1.35837">https://doi.org/10.1609/icwsm.v19i1.35837</a></div>
    <hr/>
    <div class="details_title">Metrics</div>
    <div class="details">
      <dl>
        <dt>Intensity</dt>
        <dd>Represents both the direction and strength of bias in annotations. It ranges from -1 (a strong tendency to underestimate hate speech towards individuals with different socio-demographic characteristics) to +1 (a strong tendency to overestimate hate speech). A value close to 0 suggests little to no bias.</dd>
        <dt>Prevalence</dt>
        <dd>Indicates how frequently bias appears in the annotation process. It ranges from 0 (no bias present) to 1 (bias is present in all cases).</dd>
        <dt>Agreement</dt>
        <dd>Measures how consistently annotators agree on their assessments, calculated using Cohen’s Kappa statistic.</dd>
        <dt>Significance</dt>
        <dd>P-value. Represents the statistical significance of the bias test applied to the annotations. A lower p-value suggests stronger evidence of significant bias.</dd>
        <dt>Unique Comments</dt>
        <dd>The total number of distinct comments that have been annotated.</dd>
        <dt>Annotations</dt>
        <dd>The total number of annotations made across all comments.</dd>
      </dl>
    </div>
    <hr/>
    ${filter_rows_input}
    ${filter_cols_input}
    ${significance_threshold_input}
    <hr/>
    ${options_input}
    <hr/>
    ${row_clustering_input}
    <hr/>
    ${col_clustering_input}
  </div>
  <div class="chart grid-colspan-3">
    <div>${legend}</div>
    <div>${resize((width) => matrix_chart({width}))}</div>
  </div>
</div>



<!-- Load and transform the data -->

```js
const csv = await FileAttachment("data/metrics.csv").csv({typed: true})
let data = csv.map(d => ({...d, matrix_total: d.FF+d.FM+d.FT+d.MF+d.MM+d.MT+d.TF+d.TM+d.TT}))
data.columns = csv.columns.concat(['matrix_total'])
```
```js
const VARS = ['intensity','prevalence','cohen_k','p_value','comments','matrix_total']
const readable = (d) => ({
  'intensity': 'intensity',
  'prevalence': 'prevalence',
  'cohen_k': 'agreement',
  'p_value': 'significance',
  'comments': 'unique comments',
  'matrix_total': 'annotations'
})[d]
const options_input = Inputs.form({
  variable: Inputs.select(VARS, {value: 'intensity', label: "Color by", format: readable}),
  quantize: Inputs.toggle({value: true, label: "Quantize"}),
  size_variable: Inputs.select(VARS, {value: 'prevalence', label: "Scale by", format: readable}),
  clustering_variable: Inputs.select(VARS, {value: 'intensity', label: "Cluster by", format: readable}),
  cell_size: Inputs.range([4, 40], {value: 12, step: 1, label: "Cell size"})
})
const options = view(options_input)
```

```js
const legend = Legend(config[options.variable].color_scale, {
  title: readable(options.variable),
  tickFormat: config[options.variable].tickFormat,
})
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
    html`<div style="position:relative;">${tooltipTemplate}</div>`
  )

  const tooltipDiv = container.select(".tooltip")

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
      .attr("visibility", d => d.p_value <= significance_threshold ? 'visible' : 'hidden')

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
      .call(tooltip, tooltipDiv)
  
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
  .authors, .disclaimer, .details_title, .details {
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
  .details_title {
    font-weight: bold;
    text-align: center;
  }
  .details {
    overflow-y: scroll;
    height: 100px;
  }
  dl {
    margin-top: 0;
  }
  dt {
    font-weight: bold;
  }
  dd {
    margin-left: 12px;
    margin-bottom: 6px;
  }
</style>


```js
const METHODS = ["complete", "single", "average","median","centroid"]
const row_clustering_input = Inputs.form({
  enabled: Inputs.toggle({label: "Cluster rows", value: false}),
  distance_metric: Inputs.select(Object.keys(distances), {value: "euclidean", label: "Distance Metric"}),
  method: Inputs.select(METHODS, {value: "complete", label: "Cluster Method"})
})
const row_clustering = view(row_clustering_input)
const col_clustering_input = Inputs.form({
  enabled: Inputs.toggle({label: "Cluster columns", value: false}),
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

const significance_threshold_input = Inputs.range([0, d3.max(data, d => d.p_value)], {value: 0.1, step: 0.01, label: "Significance threshold"})
const significance_threshold = view(significance_threshold_input)
```
```js
// preprocess data
const variables = data.columns.filter(d => !(['target', 'annotator'].includes(d)))
const indexed_rows = d3.group(data, d => d.annotator, d => d.target)
const indexed_cols = d3.group(data, d => d.target, d => d.annotator)
const all_rows = ["age_teenagers","age_young_adults","age_middle_aged","age_seniors","gender_men","gender_non_binary","gender_transgender_men","gender_transgender_unspecified","gender_transgender_women","gender_women","race_asian","race_black","race_latinx","race_middle_eastern","race_native_american","race_pacific_islander","race_white","religion_atheist","religion_buddhist","religion_christian","religion_hindu","religion_jewish","religion_mormon","religion_muslim","sexuality_bisexual","sexuality_gay","sexuality_lesbian","sexuality_straight","education_some_high_school","education_high_school_grad","education_some_college","education_college_grad_aa","education_college_grad_ba","education_masters","education_professional_degree","education_phd","ideology_slightly_liberal","ideology_liberal","ideology_neutral","ideology_slightly_conservative","ideology_conservative","ideology_extremeley_conservative","ideology_no_opinion","income_<10k","income_10k-50k","income_50k-100k","income_100k-200k","income_>200k"] // ordered as in the paper
const all_columns = ["age_teenagers","age_young_adults","age_middle_aged","age_seniors","gender_men","gender_non_binary","gender_transgender_men","gender_transgender_unspecified","gender_transgender_women","gender_women","race_asian","race_black","race_latinx","race_middle_eastern","race_native_american","race_pacific_islander","race_white","religion_atheist","religion_buddhist","religion_christian","religion_hindu","religion_jewish","religion_mormon","religion_muslim","sexuality_bisexual","sexuality_gay","sexuality_lesbian","sexuality_straight","disability_cognitive","disability_hearing_impaired","disability_neurological","disability_physical","disability_visually_impaired","disability_unspecific","origin_immigrant","origin_migrant_worker","origin_specific_country","origin_undocumented"] // ordered as in the paper
```
```js
// cluster data
const data_matrix = d3.map(rows, row_k => d3.map(columns, col_k => {
  const v = indexed_rows.get(row_k).get(col_k)
  return v && v[0] // propagate undefined
}))
const value_matrix = d3.map(data_matrix, row => d3.map(row, d => d === undefined ? config[options.clustering_variable].no_data : d[options.clustering_variable])) // propagate undefined

const agnes_rows = new hclust.agnes(value_matrix, {distanceFunction: distances[row_clustering.distance_metric], method: row_clustering.method})
const row_ordering = typeof agnes_rows.indices !== 'function' ? [] : !row_clustering.enabled ? d3.range(value_matrix.length) : agnes_rows.indices()
const row_sorted_matrix = row_ordering.map(i => value_matrix[i])
const row_sorted_columns = row_sorted_matrix.length === 0 ? [] : d3.range(row_sorted_matrix[0].length).map(j => row_sorted_matrix.map(row => row[j]))
const agnes_cols = new hclust.agnes(row_sorted_columns, {distanceFunction: distances[col_clustering.distance_metric], method: col_clustering.method})
const col_ordering = typeof agnes_cols.indices !== 'function' ? [] : !col_clustering.enabled ? d3.range(value_matrix[0].length) : agnes_cols.indices()
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

// inverted Viridis with additional colors
const iviridis_plus = d3.interpolateRgbBasis(['#740208','#8E4C1E','#dBd756',"#fde725","#35b779","#31688e","#440154"])

const config = ({
  cohen_k: {
    no_data: 1,
    color_scale: options.quantize ?
      d3.scaleQuantize()
        .domain([-1, 1])
        .range(d3.schemeBuPu[5].slice(1,5).reverse().concat(d3.schemeYlGn[5].slice(1,5)))
    :
      d3.scaleSequential()
        .domain([-1, 1])
        .interpolator(t => t < 0.5 ? d3.interpolateBuPu(1-t*1.5) : d3.interpolateYlGn((t-0.25)*1.5)), 
      
  },
  intensity: {
    no_data: 0,
    tickFormat: '.2f',
    color_scale: options.quantize ?
      d3.scaleQuantize()
        .domain([-1, 1])
        .range(d3.schemeRdYlBu[8].slice().reverse())
    :
      d3.scaleSequential()
        .domain([-1, 1])
        .interpolator(t => d3.interpolateRdYlBu(1-t)),
  },
  prevalence: {
    no_data: 0,
    tickFormat: '',
    color_scale: options.quantize ?
      d3.scaleQuantize()
        .domain([0, 1])
        .range(d3.range(5).map(i => d3.interpolateMagma(1-i/5)))
    :
      d3.scaleSequential()
        .domain([0, 1])
        .interpolator(t => d3.interpolateMagma(1-t)),
  },
  p_value: {
    no_data: 0,
    tickFormat: '',
    color_scale: options.quantize ?
      d3.scaleQuantize()
        .domain([0, d3.max(data, d => d.p_value)])
        .range(d3.range(5).map(i => d3.interpolateCividis(1-i/5)))
        .nice(true)
    :
      d3.scaleSequential()
        .domain([0, d3.max(data, d => d.p_value)])
        .nice(true)
        .interpolator(t => d3.interpolateCividis(1-t)),
  },
  comments: {
    no_data: 0,
    tickFormat: '~s',
    color_scale: d3.scaleSequential()
      .domain([0, d3.max(data, d => d.comments)])
      .nice(true)
      .interpolator(t => d3.interpolateWarm(1-t)),
  },
  matrix_total: {
    no_data: 0,
    tickFormat: '~s',
    color_scale: d3.scaleSequentialSqrt()
      .domain([0, d3.max(data, d => d.matrix_total)])
      .nice(true)
      .interpolator(t => d3.interpolateCool(1-t)),
  }
})
```

```js
const tooltip_value_format = d => d && d.toLocaleString()
```

```js
const tooltip_data = d => ({
  value: tooltip_value_format(d[options.variable]),
  annotator: d.row_label,
  target: d.col_label,
  values: {
    intensity: tooltip_value_format(d.intensity),
    prevalence: tooltip_value_format(d.prevalence),
    cohen_k: tooltip_value_format(d.cohen_k),
    p_value: tooltip_value_format(d.p_value),
    comments: tooltip_value_format(d.comments),
    matrix_total: tooltip_value_format(d.matrix_total),
  }
})
```

```js
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
  tooltipDiv.selectAll('hr').remove()
  
  tooltipDiv.append('p')
    .html(`<strong>${d.annotator}</strong> -> <strong>${d.target}</strong>`)
  
  tooltipDiv.append('hr')

  if(d.value !== undefined) {
    tooltipDiv.selectAll(".variables")
      .data(Object.entries(d.values).filter(([key, value]) => value !== null && value !== undefined))
      .join("p")
      .html(
        ([key, value]) =>
          `<span class="${options.variable == key ? 'selected' : ''}">${readable(key)}: ${
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

```js
// Copyright 2021, Observable Inc.
// Released under the ISC license.
// https://observablehq.com/@d3/color-legend
function Legend(color, {
    title,
    tickSize = 6,
    width = 320, 
    height = 44 + tickSize,
    marginTop = 18,
    marginRight = 0,
    marginBottom = 16 + tickSize,
    marginLeft = 0,
    ticks = width / 64,
    tickFormat,
    tickValues
  } = {}) {
  
    function ramp(color, n = 256) {
      const canvas = document.createElement("canvas");
      canvas.width = n;
      canvas.height = 1;
      const context = canvas.getContext("2d");
      for (let i = 0; i < n; ++i) {
        context.fillStyle = color(i / (n - 1));
        context.fillRect(i, 0, 1, 1);
      }
      return canvas;
    }
  
    const svg = d3.create("svg")
        .attr("width", width)
        .attr("height", height)
        .attr("viewBox", [0, 0, width, height])
        .style("overflow", "visible")
        .style("display", "block");
  
    let tickAdjust = g => g.selectAll(".tick line").attr("y1", marginTop + marginBottom - height);
    let x;
  
    // Continuous
    if (color.interpolate) {
      const n = Math.min(color.domain().length, color.range().length);
  
      x = color.copy().rangeRound(d3.quantize(d3.interpolate(marginLeft, width - marginRight), n));
  
      svg.append("image")
          .attr("x", marginLeft)
          .attr("y", marginTop)
          .attr("width", width - marginLeft - marginRight)
          .attr("height", height - marginTop - marginBottom)
          .attr("preserveAspectRatio", "none")
          .attr("xlink:href", ramp(color.copy().domain(d3.quantize(d3.interpolate(0, 1), n))).toDataURL());
    }
  
    // Sequential
    else if (color.interpolator) {
      x = Object.assign(color.copy()
          .interpolator(d3.interpolateRound(marginLeft, width - marginRight)),
          {range() { return [marginLeft, width - marginRight]; }});
  
      svg.append("image")
          .attr("x", marginLeft)
          .attr("y", marginTop)
          .attr("width", width - marginLeft - marginRight)
          .attr("height", height - marginTop - marginBottom)
          .attr("preserveAspectRatio", "none")
          .attr("xlink:href", ramp(color.interpolator()).toDataURL());
  
      // scaleSequentialQuantile doesn’t implement ticks or tickFormat.
      if (!x.ticks) {
        if (tickValues === undefined) {
          const n = Math.round(ticks + 1);
          tickValues = d3.range(n).map(i => d3.quantile(color.domain(), i / (n - 1)));
        }
        if (typeof tickFormat !== "function") {
          tickFormat = d3.format(tickFormat === undefined ? ",f" : tickFormat);
        }
      }
    }
  
    // Threshold
    else if (color.invertExtent) {
      const thresholds
          = color.thresholds ? [color.domain()[0]].concat(color.thresholds()).concat([color.domain()[1]]) // scaleQuantize WARNING modified
          : color.quantiles ? color.quantiles() // scaleQuantile
          : color.domain(); // scaleThreshold
  
      const thresholdFormat
          = tickFormat === undefined ? d => d
          : typeof tickFormat === "string" ? d3.format(tickFormat)
          : tickFormat;
  
      x = d3.scaleLinear()
          .domain([-1, color.range().length - 1])
          .rangeRound([marginLeft, width - marginRight]);
  
      svg.append("g")
        .selectAll("rect")
        .data(color.range())
        .join("rect")
          .attr("x", (d, i) => x(i - 1))
          .attr("y", marginTop)
          .attr("width", (d, i) => x(i) - x(i - 1))
          .attr("height", height - marginTop - marginBottom)
          .attr("fill", d => d);

      tickValues = d3.range(-1, thresholds.length-1); // WARNING modified
      tickFormat = i => thresholdFormat(thresholds[i+1], i); // WARNING modified
    }
  
    // Ordinal
    else {
      x = d3.scaleBand()
          .domain(color.domain())
          .rangeRound([marginLeft, width - marginRight]);
  
      svg.append("g")
        .selectAll("rect")
        .data(color.domain())
        .join("rect")
          .attr("x", x)
          .attr("y", marginTop)
          .attr("width", Math.max(0, x.bandwidth() - 1))
          .attr("height", height - marginTop - marginBottom)
          .attr("fill", color);
  
      tickAdjust = () => {};
    }
  
    svg.append("g")
        .attr("transform", `translate(0,${height - marginBottom})`)
        .call(d3.axisBottom(x)
          .ticks(ticks, typeof tickFormat === "string" ? tickFormat : undefined)
          .tickFormat(typeof tickFormat === "function" ? tickFormat : undefined)
          .tickSize(tickSize)
          .tickValues(tickValues))
        .call(tickAdjust)
        .call(g => g.select(".domain").remove())
        .call(g => g.append("text")
          .attr("x", marginLeft)
          .attr("y", marginTop + marginBottom - height - 6)
          .attr("fill", "currentColor")
          .attr("text-anchor", "start")
          .attr("font-weight", "bold")
          .attr("class", "title")
          .text(title));
  
    return svg.node();
  }
```

```js
const tooltip = (selectionGroup, tooltipDiv) => {
  selectionGroup.each(function () {
    d3.select(this)
      .on("mouseover.tooltip", handleMouseover)
      .on("mousemove.tooltip", handleMousemove)
      .on("mouseleave.tooltip", handleMouseleave);
  });

  function handleMouseover() {
    // show/reveal the tooltip, set its contents,
    // style the element being hovered on
    showTooltip();
    setContents(d3.select(this).datum(), tooltipDiv);
    setStyle(d3.select(this));
  }

  function handleMousemove(event) {
    // update the tooltip's position
    const [mouseX, mouseY] = d3.pointer(event, this);
    // add the left & top margin values to account for the SVG g element transform
    setPosition(mouseX + margin.left, mouseY + margin.top);
  }

  function handleMouseleave() {
    // do things like hide the tooltip
    // reset the style of the element being hovered on
    hideTooltip();
    resetStyle(d3.select(this));
  }

  function showTooltip() {
    tooltipDiv.style("display", "block");
  }

  function hideTooltip() {
    tooltipDiv.style("display", "none");
  }

  function setPosition(mouseX, mouseY) {
    tooltipDiv
      .style(
        "top",
        mouseY < height / 2 ? `${mouseY + MOUSE_POS_OFFSET}px` : "initial"
      )
      .style(
        "right",
        mouseX > width / 2
          ? `${width - mouseX + MOUSE_POS_OFFSET}px`
          : "initial"
      )
      .style(
        "bottom",
        mouseY > height / 2
          ? `${height - mouseY + MOUSE_POS_OFFSET}px`
          : "initial"
      )
      .style(
        "left",
        mouseX < width / 2 ? `${mouseX + MOUSE_POS_OFFSET}px` : "initial"
      );
  }
}
```

```js
const tooltipTemplate = html`<div class="tooltip tooltip-${tooltipId}">
  ${tooltipStyles}
  <div class="tooltip-contents"></div>
</div>`
```

```js
const tooltipId = 42
```

```js
const tooltipStyles = htl.html`<style>
  /* modify these styles to however you see fit */
  div.tooltip-${tooltipId} {
    box-sizing: border-box;
    position: absolute;
    display: none;
    top: 0;
    left: -100000000px;
    padding: 8px 12px;
    font-family: sans-serif;
    font-size: 12px;
    color: #333;
    background-color: #fff;
    border: 1px solid #333;
    border-radius: 4px;
    pointer-events: none;
    z-index: 1;
  }
  div.tooltip-${tooltipId} p {
    margin: 0;
  }
</style>`
```

```js
const MOUSE_POS_OFFSET = 8
```

```js
function nop(selection) {
  // do nothing, override tooltip lib default
}
```

```js
const setStyle = nop
```

```js
const resetStyle = nop
```
