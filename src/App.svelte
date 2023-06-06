<script>
  // CORE IMPORTS
  import { setContext, onMount } from 'svelte'
  import { getMotion } from './utils.js'
  import { themes } from './config.js'
  import ONSHeader from './layout/ONSHeader.svelte'
  import ONSFooter from './layout/ONSFooter.svelte'
  import Header from './layout/Header.svelte'
  import Section from './layout/Section.svelte'
  import Media from './layout/Media.svelte'
  import Scroller from './layout/Scroller.svelte'
  import Filler from './layout/Filler.svelte'
  import Divider from './layout/Divider.svelte'
  import Toggle from './ui/Toggle.svelte'
  import Arrow from './ui/Arrow.svelte'
  import Em from './ui/Em.svelte'

  import CirclePacking from './layout/CirclePacking.svelte'
  import AlluvialChart from './layout/Alluvialdiagram.svelte'

  import SankeyDiagram from './layout/SankeyDiagram.svelte'
  import Line from './layout/Line.svelte'
  import MultiLevelSankey from './layout/MultiLevelSankey.svelte'

  // DEMO-SPECIFIC IMPORTS
  import bbox from '@turf/bbox'
  import { getData, setColors, getTopo, getBreaks, getColor } from './utils.js'
  import { colors, units } from './config.js'
  import { ScatterChart, LineChart, BarChart } from '@onsvisual/svelte-charts'
  import { Map, MapSource, MapLayer, MapTooltip } from '@onsvisual/svelte-maps'

  // CORE CONFIG (COLOUR THEMES)
  // Set theme globally (options are 'light', 'dark' or 'lightblue')
  let theme = 'light'
  setContext('theme', theme)
  setColors(themes, theme)

  // CONFIG FOR SCROLLER COMPONENTS
  // Config
  const threshold = 0.65
  // State
  let animation = getMotion() // Set animation preference depending on browser preference
  let id = {} // Object to hold visible section IDs of Scroller components
  let idPrev = {} // Object to keep track of previous IDs, to compare for changes
  onMount(() => {
    idPrev = { ...id }
  })

  // DEMO-SPECIFIC CONFIG
  // Constants
  const datasets = ['region', 'district']
  const topojson = './data/geo_lad2021.json'
  const mapstyle = 'https://bothness.github.io/ons-basemaps/data/style-omt.json'
  const mapbounds = {
    nigeria: [
      [2.676932, 4.240592],
      [14.652624, 13.865923],
    ],
  }

  // Data

  let data = { district: {}, region: {} }
  let metadata = { district: {}, region: {} }
  let geojson

  // Element bindings
  let map = null // Bound to mapbox 'map' instance once initialised

  // State
  let hovered // Hovered district (chart or map)
  let selected // Selected district (chart or map)
  $: region =
    selected && metadata.district.lookup
      ? metadata.district.lookup[selected].parent
      : null // Gets region code for 'selected'
  $: chartHighlighted =
    metadata.district.array && region
      ? metadata.district.array
          .filter((d) => d.parent == region)
          .map((d) => d.code)
      : [] // Array of district codes in 'region'
  let mapHighlighted = [] // Highlighted district (map only)
  let xKey = 'area' // xKey for scatter chart
  let yKey = null // yKey for scatter chart
  let zKey = null // zKey (color) for scatter chart
  let rKey = null // rKey (radius) for scatter chart
  let mapKey = 'density' // Key for data to be displayed on map
  let explore = false // Allows chart/map interactivity to be toggled on/off

  // FUNCTIONS (INCL. SCROLLER ACTIONS)

  // Functions for chart and map on:select and on:hover events
  function doSelect(e) {
    console.log(e)
    selected = e.detail.id
    if (e.detail.feature) fitById(selected) // Fit map if select event comes from map
  }
  function doHover(e) {
    hovered = e.detail.id
  }

  // Functions for map component
  function fitBounds(bounds) {
    if (map) {
      map.fitBounds(bounds, { animate: animation, padding: 30 })
    }
  }
  function fitById(id) {
    if (geojson && id) {
      let feature = geojson.features.find((d) => d.properties.AREACD == id)
      let bounds = bbox(feature.geometry)
      fitBounds(bounds)
    }
  }

  // Actions for Scroller components
  const actions = {
    map: {
      // Actions for <Scroller/> with id="map"
      map01: () => {
        // Action for <section/> with data-id="map01"
        fitBounds(mapbounds.nigeria)
        mapKey = 'density'
        mapHighlighted = []
        explore = false
      },
      map02: () => {
        fitBounds(mapbounds.nigeria)
        mapKey = 'age_med'
        mapHighlighted = []
        explore = false
      },
      map03: () => {
        let hl = [...data.district.indicators].sort(
          (a, b) => b.age_med - a.age_med,
        )[0]
        fitById(hl.code)
        mapKey = 'age_med'
        mapHighlighted = [hl.code]
        explore = false
      },
      map04: () => {
        fitBounds(mapbounds.nigeria)
        mapKey = 'age_med'
        mapHighlighted = []
        explore = true
      },
    },
    chart: {
      chart01: () => {
        xKey = 'area'
        yKey = null
        zKey = null
        rKey = null
        explore = false
      },
      chart02: () => {
        xKey = 'area'
        yKey = null
        zKey = null
        rKey = 'pop'
        explore = false
      },
      chart03: () => {
        xKey = 'area'
        yKey = 'density'
        zKey = null
        rKey = 'pop'
        explore = false
      },
      chart04: () => {
        xKey = 'area'
        yKey = 'density'
        zKey = 'parent_name'
        rKey = 'pop'
        explore = false
      },
      chart05: () => {
        xKey = 'area'
        yKey = 'density'
        zKey = null
        rKey = 'pop'
        explore = true
      },
    },
  }

  // Code to run Scroller actions when new caption IDs come into view
  function runActions(codes = []) {
    codes.forEach((code) => {
      if (id[code] != idPrev[code]) {
        if (actions[code][id[code]]) {
          actions[code][id[code]]()
        }
        idPrev[code] = id[code]
      }
    })
  }
  $: id && runActions(Object.keys(actions)) // Run above code when 'id' object changes

  // INITIALISATION CODE
  datasets.forEach((geo) => {
    getData(`./data/data_${geo}.csv`).then((arr) => {
      let meta = arr.map((d) => ({
        code: d.code,
        name: d.name,
        parent: d.parent ? d.parent : null,
      }))
      let lookup = {}
      meta.forEach((d) => {
        lookup[d.code] = d
      })
      metadata[geo].array = meta
      metadata[geo].lookup = lookup

      let indicators = arr.map((d, i) => ({
        ...meta[i],
        area: d.area,
        pop: d['2020'],
        density: d.density,
        age_med: d.age_med,
      }))

      if (geo == 'district') {
        ;['density', 'age_med'].forEach((key) => {
          let values = indicators.map((d) => d[key]).sort((a, b) => a - b)
          let breaks = getBreaks(values)
          indicators.forEach(
            (d, i) =>
              (indicators[i][key + '_color'] = getColor(
                d[key],
                breaks,
                colors.seq,
              )),
          )
        })
      }
      data[geo].indicators = indicators

      let years = [
        2001,
        2002,
        2003,
        2004,
        2005,
        2006,
        2007,
        2008,
        2009,
        2010,
        2011,
        2012,
        2013,
        2014,
        2015,
        2016,
        2017,
        2018,
        2019,
        2020,
      ]

      let timeseries = []
      arr.forEach((d) => {
        years.forEach((year) => {
          timeseries.push({
            code: d.code,
            name: d.name,
            value: d[year],
            year,
          })
        })
      })
      data[geo].timeseries = timeseries
    })
  })

  getTopo(topojson, 'geog').then((geo) => {
    geo.features.sort((a, b) =>
      a.properties.AREANM.localeCompare(b.properties.AREANM),
    )
    geojson = geo
  })

  // const sankey_data = {
  // 	nodes: [
  // 	{ name: 'Node 1' },
  // 	{ name: 'Node 2' },
  // 	{ name: 'Node 3' },
  // 	{ name: 'Node 4' },
  // 	{ name: 'Node 5' }
  // 	],
  // 	links: [
  // 	{ source: 0, target: 1, value: 10 },
  // 	{ source: 0, target: 2, value: 5 },
  // 	{ source: 1, target: 3, value: 15 },
  // 	{ source: 1, target: 4, value: 20 },
  // 	{ source: 2, target: 3, value: 5 },
  // 	{ source: 2, target: 4, value: 10 }
  // 	]
  // };

  const sankey_data = {
    nodes: [
      { name: 'First wave' },
      { name: 'second wave' },
      { name: 'third wave' },
      { name: 'Gwoza (GSS Camp)' },
      { name: 'Dusuman (Farm Centre Camp)' },
      { name: 'Dusuman (Bakassi Camp)' },
      { name: 'Maisandari (Stadium Camp)' },
      { name: 'Bolori I (Teachers’ Village Camp)' },
      { name: 'Dusuman (Custom House 1 and 2 Camp)' },
      { name: 'Dalori/Wanori (Dalori 1 Camp)' },
      { name: 'Dalori/Wanori (Dalori 2 Camp)' },
      { name: 'Mafoni (Mogolis Camp)' },
      { name: 'Maisandari (NYSC Camp)' },
      { name: 'Dusuman (Muna El Badawi Camp)' },
      { name: 'Auno/Chabbol (Gubio Camp)' },
      ///
      { name: 'Gwoza (Ngoshe)' },
      { name: 'Mafa (Ajiri)' },
      { name: 'Marte (Garadai)' },
      { name: 'Nganzai (Gajiran)' },
      { name: 'Kukawa (Baga)' },
      { name: 'Konduga (Auno)' },
      { name: 'Konduga (Kawuri)' },
      { name: 'Mafa (Mafe)' },
      { name: 'Gwoza (Gwoza)' },
      { name: 'Gwoza (Pulka)' },
      { name: 'Monguno (Monguno)' },
      { name: 'Mafa (Mafa)' },
      { name: 'Bama (Bama)' },
      { name: 'Maiduguri (Dalori Village)' },
      { name: 'Mafa (Ngarannam)' },
      { name: 'Mobbar (Damasak)' },
      { name: 'Dikwa (Dikwa)' },
      { name: 'Kala/Balge (Rann)' },
      { name: 'Ngala (Ngala)' },
      { name: 'Magumeri (Magumeri)' },
    ],
    links: [
      { source: 0, target: 1, value: 182172 },
      { source: 0, target: 2, value: 72333 },
      { source: 1, target: 2, value: 33334 },
      { source: 0, target: 3, value: 5320 },
      { source: 1, target: 3, value: 37000 },
      { source: 2, target: 3, value: 39900 },
      { source: 3, target: 4, value: 25350 },
      { source: 3, target: 8, value: 28764 },
      { source: 3, target: 5, value: 6000 },
      { source: 3, target: 6, value: 23895 },
      { source: 3, target: 7, value: 16574 },
      { source: 3, target: 9, value: 4701 },
      { source: 3, target: 10, value: 51135 },
      { source: 3, target: 11, value: 21198 },
      /////
      { source: 0, target: 8, value: 8750 },
      { source: 0, target: 9, value: 8750 },
      { source: 1, target: 7, value: 10000 },
      { source: 1, target: 11, value: 15532 },
      { source: 2, target: 10, value: 39650 },
      { source: 3, target: 19, value: 10227 },
      { source: 4, target: 17, value: 9334 },
      { source: 5, target: 6, value: 5264 },
      { source: 5, target: 11, value: 36224 },
    ],
  }

  // Line Data
  const chartData = [
    {
      name: 'Line 1',
      values: [
        { month: 1, value: 15.13 },
        { month: 2, value: 14.33 },
        { month: 3, value: 13.34 },
        { month: 4, value: 12.48 },
        { month: 5, value: 11.61 },
        { month: 6, value: 11.23 },
        { month: 7, value: 11.14 },
        { month: 8, value: 11.23 },
        // { month: 9, value: 11.28 },
        // { month: 10, value: 11.26 },
        // { month: 11, value: 11.28 },
        // { month: 12, value: 11.44 },
        // { month: 13, value: 11.37 },
        // { month: 14, value: 11.31 },
        // { month: 15, value: 11.25 },
        // { month: 16, value: 11.37 },
        // { month: 17, value: 11.4 },
        // { month: 18, value: 11.22 },
        // { month: 19, value: 11.08 },
        // { month: 20, value: 11.02 },
        // { month: 21, value: 11.24 },
        // { month: 22, value: 11.61 },
        // { month: 23, value: 11.85 },
        // { month: 24, value: 11.98 },
        // { month: 25, value: 12.13 },
        // { month: 26, value: 12.2 },
        // { month: 27, value: 12.26 },
        // { month: 28, value: 12.34 },
        // { month: 29, value: 12.4 },
        // { month: 30, value: 12.56 },
        // { month: 31, value: 12.82 },
        // { month: 32, value: 13.22 },
        // { month: 33, value: 13.71 },
        // { month: 34, value: 14.23 },
        // { month: 35, value: 14.89 },
        // { month: 36, value: 15.75 },
        // { month: 37, value: 16.47 },
        // { month: 38, value: 17.33 },
        // { month: 39, value: 18.17 },
        // { month: 40, value: 18.12 },
        // { month: 41, value: 17.93 },
        // { month: 42, value: 17.75 },
        // { month: 43, value: 17.38 },
        // { month: 44, value: 17.01 },
        // { month: 45, value: 16.63 },
        // { month: 46, value: 15.99 },
        // { month: 47, value: 15.4 },
        // { month: 48, value: 15.63 },
        // { month: 49, value: 15.6 },
        // { month: 50, value: 15.13 },
      ],
    },
  ]

  const sank_data = {
    nodes: [
      { name: 'First wave' },
      { name: 'Gwoza (Ngoshe)' },
      { name: 'Mafa (Ajiri)' },
      { name: 'Marte (Garadai)' },
      { name: 'Nganzai (Gajiran)' },
      { name: 'Kukawa (Baga)' },
      { name: 'Konduga (Auno)' },
      { name: 'Konduga (Kawuri)' },
      { name: 'Mafa (Mafe)' },
      { name: 'Gwoza (Gwoza)' },
      { name: 'Gwoza (Pulka)' },
      { name: 'Monguno (Monguno)' },
      { name: 'Mafa (Mafa)' },
      { name: 'Second wave' },
      { name: 'Bama (Bama)' },
      { name: 'Maiduguri (Dalori Village)' },
      { name: 'Mafa (Ngarannam)' },
      { name: 'Mobbar (Damasak)' },
      { name: 'Dikwa (Dikwa)' },
      { name: 'Kala/Balge (Rann)' },
      { name: 'Ngala (Ngala)' },
      { name: 'Magumeri (Magumeri)' },
      { name: 'Gwoza (GSS Camp)' },
      { name: 'Dusuman (Farm Centre Camp)' },
      { name: 'Dusuman (Bakassi Camp)' },
      { name: 'Maisandari (Stadium Camp)' },
      { name: 'Bolori I (Teachers’ Village Camp)' },
      { name: 'Dalori/Wanori (Dalori 1 Camp)' },
      { name: 'Dalori/Wanori (Dalori 2 Camp)' },
      { name: 'Mafoni (Mogolis Camp)' },
    ],
    links: [
      { source: 0, target: 1, value: 5320 },
      { source: 0, target: 2, value: 13727 },
      { source: 0, target: 3, value: 45443 },
      { source: 0, target: 4, value: 3500 },
      { source: 0, target: 5, value: 20450 },
      { source: 0, target: 6, value: 36224 },
      { source: 0, target: 7, value: 5264 },
      { source: 0, target: 8, value: 10000 },
      { source: 0, target: 9, value: 8750 },
      { source: 0, target: 10, value: 8750 },
      { source: 0, target: 11, value: 39650 },
      { source: 0, target: 12, value: 15532 },
      { source: 1, target: 13, value: 13390 },
      { source: 2, target: 13, value: 3858 },
      { source: 2, target: 14, value: 5000 },
      { source: 3, target: 14, value: 9334 },
      { source: 5, target: 15, value: 10227 },
      { source: 6, target: 15, value: 10227 },
      { source: 7, target: 15, value: 10227 },
      { source: 8, target: 16, value: 7066 },
      { source: 9, target: 17, value: 5320 },
      { source: 10, target: 17, value: 5320 },
      { source: 11, target: 18, value: 37000 },
      { source: 11, target: 19, value: 39900 },
      { source: 11, target: 20, value: 25350 },
      { source: 11, target: 21, value: 28764 },
      { source: 11, target: 22, value: 6000 },
      { source: 13, target: 23, value: 23895 },
      { source: 13, target: 24, value: 16574 },
      { source: 6, target: 23, value: 28002 },
      { source: 11, target: 25, value: 4701 },
      { source: 11, target: 26, value: 51135 },
      { source: 5, target: 27, value: 21198 },
    ],
  }
  // Bubble chart
  const bubble_cart = {
    name: 'root',
    children: [
      { name: 'ad', value: 10 },
      { name: 'sr', value: 20 },
      { name: 'Kwali', value: 30 },
      { name: 'Oyo West', value: 40 },
      { name: 'Batsari', value: 50 },
      { name: 'Kibiya', value: 60 },
      { name: 'Funakaye', value: 70 },
      { name: 'Ibadan', value: 80 },
      { name: 'Ibadan', value: 90 },
      { name: 'Garko', value: 100 },
      { name: 'Enugu South', value: 110 },
      { name: 'Ogo Oluwa', value: 120 },
      { name: 'ABaji', value: 130 },
      { name: 'Tafawa ', value: 140 },
      { name: 'Ifedaya', value: 150 },
      { name: 'Jos South', value: 160 },
      { name: 'Jos West', value: 170 },
      { name: 'Ezegu', value: 180 },
      { name: 'Ungogo', value: 190 },
      { name: 'Nembe', value: 200 },
      { name: 'Darazo', value: 210 },
      { name: 'Kondugo', value: 220 },
    ],
  }

  const sanky_chart_data = [
    { Age_Group: '0-17', Sex: 'Male', State: 'Adamawa' },
    { Age_Group: '0-17', Sex: 'Male', State: 'Borno' },
    { Age_Group: '0-17', Sex: 'Male', State: 'Yobe' },
    { Age_Group: '0-17', Sex: 'Female', State: 'Adamawa' },
    { Age_Group: '0-17', Sex: 'Female', State: 'Borno' },
    { Age_Group: '18-45', Sex: 'Female', State: 'Yobe' },
    { Age_Group: '18-45', Sex: 'Male', State: 'Adamawa' },
    { Age_Group: '18-45', Sex: 'Male', State: 'Borno' },
    { Age_Group: '18-45', Sex: 'Male', State: 'Yobe' },
    { Age_Group: '18-45', Sex: 'Female', State: 'Adamawa' },
    { Age_Group: '18-45', Sex: 'Female', State: 'Borno' },
    { Age_Group: '18-45', Sex: 'Female', State: 'Yobe' },
    { Age_Group: '46-60', Sex: 'Male', State: 'Adamawa' },
    { Age_Group: '46-60', Sex: 'Male', State: 'Borno' },
    { Age_Group: '46-60', Sex: 'Male', State: 'Yobe' },
    { Age_Group: '46-60', Sex: 'Female', State: 'Adamawa' },
    { Age_Group: '46-60', Sex: 'Female', State: 'Borno' },
    { Age_Group: '46-60', Sex: 'Female', State: 'Yobe' },
    { Age_Group: '60 and above', Sex: 'Male', State: 'Adamawa' },
    { Age_Group: '60 and above', Sex: 'Male', State: 'Borno' },
    { Age_Group: '60 and above', Sex: 'Male', State: 'Yobe' },
    { Age_Group: '60 and above', Sex: 'Female', State: 'Adamawa' },
    { Age_Group: '60 and above', Sex: 'Female', State: 'Borno' },
  ]

  // console.log(metadata['region']);
</script>

<style>
  /* Styles specific to elements within the demo */
  :global(svelte-scroller-foreground) {
    pointer-events: none !important;
  }
  :global(svelte-scroller-foreground section div) {
    pointer-events: all !important;
  }
  select {
    max-width: 350px;
  }
  .chart {
    margin-top: 45px;
    width: calc(100% - 5px);
  }
  .chart-full {
    margin: 0 20px;
  }
  .chart-sml {
    font-size: 0.85em;
  }

  .title {
    font-size: 36px;
  }
  .font {
    font-family: 'Roboto Slab', serif;
    font-weight: bold;
  }
  /* The properties below make the media DIVs grey, for visual purposes in demo */
  .media {
    background-color: #f0f0f0;
    display: -webkit-box;
    display: -ms-flexbox;
    display: flex;
    -webkit-box-orient: vertical;
    -webkit-box-direction: normal;
    -ms-flex-flow: column;
    flex-flow: column;
    -webkit-box-pack: center;
    -ms-flex-pack: center;
    justify-content: center;
    text-align: center;
    color: #aaa;
  }
  .image {
    height: 450px;
  }
  .column {
    display: flex;
    justify-content: space-between;
    align-items: center;
  }
  .center {
    text-align: center;
  }
  .sankey_graph,
  .circle_graph {
    display: flex;
    align-items: center;
    justify-content: center;
  }

  ul {
    padding: 0;
  }
  li {
    list-style-type: none;
  }
</style>

<ONSHeader filled={true} center={false} />

<!-- <AlluvialChart {sanky_chart_data} /> -->

<!-- <MultiLevelSankey data={multi_data} /> -->

<Header
  bgcolor="#f29d09"
  bgfixed={true}
  theme="dark"
  center={false}
  short={true}>
  <div class="font">
    <h1 class="title">NIGERIA HUMANITARIAN RESPONSE PLAN SUMMARY</h1>
    <p style="margin-top: 20px">ISSUED FEBRUARY 2023</p>
    <div style="margin-top: 90px;">
      <Arrow color="white" {animation}>Scroll to begin</Arrow>
    </div>
  </div>

</Header>

<Filler theme="lightblue" short={true} wide={true} center={false}>
  <div class="column">
    <p class="text-big">This is a large, left-aligned text caption</p>
    <img class="image" src="images/woman.png" alt="" />
  </div>
</Filler>
<Section>
  <h2>This is a section title</h2>
  <p>
    This is a short paragraph of text to demonstrate the standard "medium"
    column width, font size and line spacing of the template.
  </p>
  <p>
    This is a second short paragraph of text to demonstrate the size of the
    paragraph spacing in the template.
  </p>
  <blockquote class="text-indent">
    "This is an example of a large embedded quotation."&mdash;A. Person
  </blockquote>
</Section>

<Divider />

<Section>
  <h2>Embedded charts or media</h2>
  <p>
    Below is an embedded chart. It is set to the same width as the column,
    "medium" (680px), but could also be "narrow" (540px), "wide" (980px) or
    "full" width. All options are responsive to fit the width of narrow screens.
  </p>
</Section>

{#if data.region.indicators}
  <Media col="medium" caption="Source: ONS mid-year population estimates.">
    <div class="chart-sml">
      <BarChart
        data={[...data.region.indicators].sort((a, b) => a.pop - b.pop)}
        xKey="pop"
        yKey="name"
        snapTicks={false}
        xFormatTick={(d) => d / 1e6}
        xSuffix="m"
        height={350}
        padding={{ top: 0, bottom: 15, left: 140, right: 0 }}
        area={false}
        title="Population by region/nation, 2020" />
    </div>
  </Media>
{/if}

<Divider />

<Section>
  <h2>Gridded charts or media</h2>
  <p>
    Below is a grid that can contain charts or any other kind of visual media.
    The grid can fit in a medium, wide or full-width column, and the media width
    itself can be narrow (min 200px), medium (min 300px), wide (min 500px) or
    full-width. The grid is responsive, and will re-flow on smaller screens.
  </p>
</Section>

{#if data.region.timeseries && data.region.indicators}
  <Media
    col="wide"
    grid="narrow"
    gap={20}
    caption="Source: ONS mid-year population estimates.">
    {#each [...data.region.indicators].sort((a, b) => b.pop - a.pop) as region}
      <div class="chart-sml">
        <LineChart
          data={data.region.timeseries}
          xKey="year"
          yKey="value"
          zKey="code"
          color="lightgrey"
          lineWidth={1}
          xTicks={2}
          snapTicks={false}
          yFormatTick={(d) => d / 1e6}
          ySuffix="m"
          height={200}
          padding={{ top: 0, bottom: 20, left: 30, right: 15 }}
          selected={region.code}
          area={false}
          title={region.name} />
      </div>
    {/each}
  </Media>
{/if}

<Divider />

<Section>
  <h2>This is a dynamic chart section</h2>
  <p>
    The chart below will respond to the captions as you scroll down. The
    "Scroller" component is set to "splitscreen" mode, which means the captions
    will be on the left side on larger screens.
  </p>
  <p>
    The interactions are via Javascript functions that are called when each
    caption scrolls into view.
  </p>
</Section>

<Scroller {threshold} bind:id={id['chart']} splitscreen={true}>
  <div slot="background">
    <figure>
      <div class="col-wide height-full">
        {#if data.district.indicators && metadata.region.lookup}
          <div class="chart">
            <ScatterChart
              height="calc(100vh - 150px)"
              data={data.district.indicators.map((d) => ({
                ...d,
                parent_name: metadata.region.lookup[d.parent],
              }))}
              colors={explore ? ['lightgrey'] : colors.cat}
              {xKey}
              {yKey}
              {zKey}
              {rKey}
              idKey="code"
              labelKey="name"
              r={[3, 10]}
              xScale="log"
              xTicks={[10, 100, 1000, 10000]}
              xFormatTick={(d) => d.toLocaleString()}
              xSuffix=" sq.km"
              yFormatTick={(d) => d.toLocaleString()}
              legend={zKey != null}
              labels
              select={explore}
              selected={explore ? selected : null}
              on:select={doSelect}
              hover
              {hovered}
              on:hover={doHover}
              highlighted={explore ? chartHighlighted : []}
              colorSelect="#206095"
              colorHighlight="#999"
              overlayFill
              {animation} />
          </div>
        {/if}
      </div>
    </figure>
  </div>

  <div slot="foreground">
    <section data-id="chart01">
      <div class="col-medium">
        <p>
          This chart shows the
          <strong>area in square kilometres</strong>
          of each local authority district in the nigera. Each circle represents
          one district. The scale is logarithmic.
        </p>
      </div>
    </section>
    <section data-id="chart02">
      <div class="col-medium">
        <p>
          The radius of each circle shows the
          <strong>total population</strong>
          of the district.
        </p>
      </div>
    </section>
    <section data-id="chart03">
      <div class="col-medium">
        <p>
          The vertical axis shows the
          <strong>density</strong>
          of the district in people per hectare.
        </p>
      </div>
    </section>
    <section data-id="chart04">
      <div class="col-medium">
        <p>
          The colour of each circle shows the
          <strong>part of the country</strong>
          that the district is within.
        </p>
      </div>
    </section>
    <section data-id="chart05">
      <div class="col-medium">
        <h3>Select a district</h3>
        <p>
          Use the selection box below or click on the chart to select a
          district. The chart will also highlight the other districts in the
          same part of the country.
        </p>
        {#if geojson}
          <p>
            <!-- svelte-ignore a11y-no-onchange -->
            <select bind:value={selected}>
              <option value={null}>Select one</option>
              {#each geojson.features as place}
                <option value={place.properties.AREACD}>
                  {place.properties.AREANM}
                </option>
              {/each}
            </select>
          </p>
        {/if}
      </div>
    </section>
  </div>
</Scroller>

<Divider />

<Section>
  <h2>This is a full-width chart demo</h2>
  <p>
    Below is an example of a media grid where the column with is set to "full".
    This allows for full width images and charts.
  </p>
  <p />
</Section>

<Media
  col="full"
  height={600}
  caption="This is an optional caption for the above chart or media. It can
  contain HTML code and wrap onto multiple lines.">
  <div class="chart-full">
    {#if data.district.timeseries}
      <LineChart
        data={data.district.timeseries}
        padding={{ left: 50, right: 150, top: 0, bottom: 0 }}
        height="500px"
        xKey="year"
        yKey="value"
        zKey="code"
        color="lightgrey"
        lineWidth={1}
        yFormatTick={(d) => (d / 1e6).toFixed(1)}
        ySuffix="m"
        select
        {selected}
        on:select={doSelect}
        hover
        {hovered}
        on:hover={doHover}
        highlighted={chartHighlighted}
        colorSelect="#206095"
        colorHighlight="#999"
        area={false}
        title="Mid-year population by district, 2001 to 2020"
        labels
        labelKey="name" />
    {/if}
  </div>
</Media>

<Divider />

<Section>
  <h2 class="center">Trends of inflation 2018 - September 2023</h2>
</Section>
<Media col="full" height={200} caption="">
  <Line />
</Media>

<Section>
  <h2 class="center">Camp Resetellment</h2>
  <br />
</Section>
<div class="sankey_graph">
  <SankeyDiagram data={sank_data} width={900} height={700} />
</div>

<Divider />

<Section>
  <h2 class="center">CircleArea Data</h2>
  <br />
</Section>
<div class="circle_graph">
  <CirclePacking data={bubble_cart} width={900} height={700} />
</div>
<Divider />

<Section>
  <h2>This is a dynamic map section</h2>
  <p class="mb">
    The map below will respond to the captions as you scroll down. The scroller
    is not set to splitscreen, so captions are placed over the map on any screen
    size.
  </p>
</Section>

{#if geojson && data.district.indicators}
  <Scroller {threshold} bind:id={id['map']}>
    <div slot="background">
      <figure>
        <div class="col-full height-full">
          <!-- location={{ bounds: [11.4, 7.5, 13.9, 10.2] }} -->
          <Map
            style={mapstyle}
            bind:map
            interactive={false}
            location={{ center: [9.3375, 12.3964], zoom: 7 }}>
            <MapSource
              id="lad"
              type="geojson"
              data={geojson}
              promoteId="AREACD"
              maxzoom={13}>
              <MapLayer
                id="lad-fill"
                idKey="code"
                colorKey={mapKey + '_color'}
                data={data.district.indicators}
                type="fill"
                select
                {selected}
                on:select={doSelect}
                clickIgnore={!explore}
                hover
                {hovered}
                on:hover={doHover}
                highlight
                highlighted={mapHighlighted}
                paint={{ 'fill-color': ['case', ['!=', ['feature-state', 'color'], null], ['feature-state', 'color'], 'rgba(255, 255, 255, 0)'], 'fill-opacity': 0.7 }}>
                <MapTooltip
                  content={hovered ? `${metadata.district.lookup[hovered].name}<br/><strong>${data.district.indicators
                        .find((d) => d.code == hovered)
                        [
                          mapKey
                        ].toLocaleString()} ${units[mapKey]}</strong>` : ''} />
              </MapLayer>
              <MapLayer
                id="lad-line"
                type="line"
                paint={{ 'line-color': ['case', ['==', ['feature-state', 'hovered'], true], 'orange', ['==', ['feature-state', 'selected'], true], 'black', ['==', ['feature-state', 'highlighted'], true], 'black', 'rgba(255,255,255,0)'], 'line-width': 2 }} />
            </MapSource>
          </Map>
        </div>
      </figure>
    </div>

    <div slot="foreground">
      <section data-id="map01">
        <div class="col-medium">
          <p>
            This map shows
            <strong>population density</strong>
            by district. Districts are coloured from
            <Em color={colors.seq[0]}>least dense</Em>
            to
            <Em color={colors.seq[4]}>most dense</Em>
            . You can hover to see the district name and density.
          </p>
        </div>
      </section>
      <section data-id="map02">
        <div class="col-medium">
          <p>
            The map now shows
            <strong>median age</strong>
            , from
            <Em color={colors.seq[0]}>youngest</Em>
            to
            <Em color={colors.seq[4]}>oldest</Em>
            .
          </p>
        </div>
      </section>
      <section data-id="map03">
        <div class="col-medium">
          <!-- This gets the data object for the district with the oldest median age -->
          {#each [[...data.district.indicators].sort((a, b) => b.age_med - a.age_med)[0]] as district}
            <p>
              The map is now zoomed on
              <Em color={district.age_med_color}>{district.name}</Em>
              , the district with the oldest median age, {district.age_med}
              years.
            </p>
          {/each}
        </div>
      </section>
      <section data-id="map04">
        <div class="col-medium">
          <h3>Select a district</h3>
          <p>
            Use the selection box below or click on the map to select and zoom
            to a district.
          </p>
          {#if geojson}
            <p>
              <!-- svelte-ignore a11y-no-onchange -->
              <select bind:value={selected} on:change={() => fitById(selected)}>
                <option value={null}>Select one</option>
                {#each geojson.features as place}
                  <option value={place.properties.AREACD}>
                    {place.properties.AREANM}
                  </option>
                {/each}
              </select>
            </p>
          {/if}
        </div>
      </section>
    </div>
  </Scroller>
{/if}

<Divider />

<Section>
  <div class="row">
    <h2>Get the latest updates</h2>
    <br />
    <p class="">
      Get the latest updates, OCHA coordinates humanitarian action to ensure
      crisis-affected people receive the assistance and protection they need. It
      works to overcome obstacles that impede humanitarian assistance from
      reaching people affected by crises, and provides leadership in mobilizing
      assistance and resources on behalf of the humanitarian systems
    </p>
    <ul>
      <li>
        <a
          href="https://unocha.org/nigeria"
          class="link"
          target="_blank"
          style="color: #efb95c">
          www.unocha.org/nigeria
        </a>
      </li>
      <li>
        <a
          href="https://reports.unocha.org/en/country/nigeria"
          class="link"
          target="_blank"
          style="color: #efb95c">
          https://reports.unocha.org/en/country/nigeria
        </a>
      </li>
      <li>
        <a
          href="https://twitter.com/ochanigeria"
          class="link"
          target="_blank"
          style="color: #efb95c">
          twitter.com/ochanigeria
        </a>
      </li>
    </ul>
  </div>
  <div class="row">
    <h2>reliefweb</h2>
    <br />
    <p class="">
      ReliefWeb aims to be the central website for Information Management tools
      and services, enabling information exchange between clusters and IASC
      members operating within a protracted or sudden onset crisis.
    </p>
    <ul>
      <li>
        <a
          href="https://reliefweb.int/country/nga"
          class="link"
          target="_blank"
          style="color: #efb95c">
          https://reliefweb.int/country/ngas
        </a>
      </li>
      <li>
        <p>2023 Humanitarian Response Plan</p>
        <a
          href="https://reliefweb.int/country/nga"
          class="link"
          target="_blank"
          style="color: #efb95c">
          https://to be shared
        </a>
      </li>
      <li>
        <p>2023 Humanitarian Needs Overview</p>
        <a
          href="https://reliefweb.int/country/nga"
          class="link"
          target="_blank"
          style="color:#2e88b6">
          https://to be shared
        </a>
      </li>
    </ul>
  </div>
  <div class="row">
    <h2>Humanitarian inSight</h2>
    <br />
    <p class="">
      Humanitarian InSight supports decision-makers by giving them access to key
      humanitarian data. It provides the latest verified information on needs
      and delivery of the humanitarian response as well as financial
      contributions.
    </p>
    <ul>
      <li>
        <a
          href="https://humanitarianaction.info/plan/1110"
          class="link"
          target="_blank"
          style="color: #efb95c">
          https://humanitarianaction.info/plan/1110
        </a>
      </li>
    </ul>
  </div>
  <div class="row">
    <h2>Financial Tracking Service</h2>
    <br />
    <p class="">
      The Financial Tracking Service (FTS) is the primary provider of
      continuously updated data on global humanitarian funding, and is a major
      contributor to strategic decision making by highlighting gaps and
      priorities, thus contributing to effective, efficient and principled
      humanitarian assistance.
    </p>
    <ul>
      <li>
        <a
          href="https://fts.unocha.org/countries/163/summary/2023"
          class="link"
          target="_blank"
          style="color: #efb95c">
          https://fts.unocha.org/countries/163/summary/2023
        </a>
      </li>
    </ul>
  </div>
</Section>

<ONSFooter />
