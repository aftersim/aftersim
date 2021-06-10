<template lang="pug">
.map(:id="mapID")
</template>

<script lang="ts">
import { Vue, Component, Watch, Prop } from 'vue-property-decorator'
import { GeoJsonLayer } from '@deck.gl/layers'
import { scaleLinear, scaleThreshold } from 'd3-scale'
import colormap from 'colormap'

import LayerManager from '@/util/LayerManager'

const SCALED_COLORS = scaleThreshold()
  .domain([0, 0.1, 0.2, 0.3, 0.4, 0.5, 0.6, 0.7, 0.8, 0.9])
  .range([
    [26, 152, 80],
    [102, 189, 99],
    [166, 217, 106],
    [217, 239, 139],
    [255, 255, 191],
    [254, 224, 139],
    [253, 174, 97],
    [244, 109, 67],
    [215, 48, 39],
    [168, 0, 0],
  ] as any)

@Component({ components: {} })
export default class VueComponent extends Vue {
  @Prop({ required: true })
  private props!: {
    shapefile: { data: any[]; header: any; prj: string }
    dark: boolean
    colors: string
    activeColumn: string
    maxValue: number
  }

  private layerManager!: LayerManager
  private mapID = `map-id-${Math.floor(1e12 * Math.random())}`

  private get viewState() {
    return this.$store.state.viewState
  }

  @Watch('viewState') viewMoved() {
    this.layerManager.deckInstance.setProps({ viewState: this.viewState })
  }

  @Watch('props.dark') swapTheme() {
    this.layerManager.updateStyle()
  }

  @Watch('props')
  private handlePropsChanged() {
    if (this.layerManager) this.updateLayers()
  }

  private mounted() {
    console.log(this.mapID)
    this.setupLayerManager()
    this.updateLayers()
  }

  private beforeDestroy() {
    this.layerManager.destroy()
  }

  private setupLayerManager() {
    this.layerManager = new LayerManager()

    this.layerManager.init({
      container: `#${this.mapID}`,
      viewState: this.$store.state.viewState,
      pickingRadius: 3,
      onViewStateChange: ({ viewState }: any) => {
        this.$store.commit('setMapCamera', viewState)
      },
    })
  }

  private handleClick() {
    console.log('click!')
  }

  private getTooltip({ hoverInfo }: any) {
    const { object, x, y } = hoverInfo
    if (!object) return

    if (object.properties.centerX) delete object.properties.centerX
    if (object.properties.centerY) delete object.properties.centerY

    // round fractions
    for (const key of Object.keys(object.properties)) {
      const value = object.properties[key]
      if (!isNaN(value)) {
        object.properties[key] = Math.round(1000 * value) / 1000
      }
    }

    // try to figure out how tall it is? So tooltip doesn't go below the screen bottom
    let tooltipHeight = 24 + 22 * Object.keys(object.properties).length
    if (y + tooltipHeight < window.innerHeight) tooltipHeight = 0

    // const tooltip = (
    //   <div
    //     id="shape-tooltip"
    //     className="tooltip"
    //     style={{
    //       backgroundColor: dark ? '#445' : 'white',
    //       color: dark ? 'white' : '#222',
    //       padding: '1rem 1rem',
    //       position: 'absolute',
    //       left: x + 15,
    //       top: y - tooltipHeight,
    //       boxShadow: '0px 2px 10px #22222266',
    //     }}
    //   >
    //     <div>
    //       {Object.keys(object.properties).map((prop, i) => {
    //         return (
    //           <div key={i}>
    //             <b>{prop}:&nbsp;</b>
    //             {object.properties[prop]}
    //           </div>
    //         )
    //       })}
    //     </div>
    //   </div>
    // )

    // return tooltip
  }

  private renderTooltip({ hoverInfo }: any) {
    const { object, x, y } = hoverInfo
    if (!object) return null

    const id = object.properties?.id
    const row = this.props.shapefile.data[id]
    if (!row) return null

    const value: any = row[this.props.activeColumn]
    if (value === undefined) return null

    // return (
    //   <div
    //     className="tooltip"
    //     style={{
    //       backgroundColor: dark ? '#445' : 'white',
    //       color: dark ? 'white' : '#222',
    //       padding: '1rem 1rem',
    //       position: 'absolute',
    //       left: x + 20,
    //       top: y - 80,
    //       boxShadow: '0px 2px 10px #22222266',
    //     }}
    //   >
    //     <big>
    //       <b>{header}</b>
    //     </big>
    //     <p>{value}</p>
    //   </div>
    // )
  }

  private updateLayers() {
    const builtColors = colormap({
      colormap: this.props.colors,
      nshades: 20,
      format: 'rba',
    }).map((a: number[]) => [a.slice(0, 3)])

    const fetchColor = scaleThreshold()
      .domain(new Array(20).fill(0).map((v, i) => 0.05 * i))
      .range(builtColors)

    this.layerManager.removeLayer('shapefile')
    this.layerManager.addLayer(
      new GeoJsonLayer({
        id: 'shapefileLayer',
        data: this.props.shapefile.data,
        filled: true,
        lineWidthUnits: 'pixels',
        lineWidthMinPixels: 1,
        pickable: true,
        stroked: true,
        opacity: 0.5,
        autoHighlight: true,
        // highlightColor: [255, 128, 255, 255], // [64, 255, 64],
        parameters: {
          depthTest: true,
        },

        getLineColor: [255, 255, 255, 128],
        getFillColor: (f: any) =>
          SCALED_COLORS(f.properties[this.props.activeColumn] / this.props.maxValue),
        getLineWidth: 2,
        // onHover: this.setHoverInfo,

        updateTriggers: {
          getFillColor: {
            dark: this.props.dark,
            colors: this.props.colors,
            activeColumn: this.props.activeColumn,
            maxValue: this.props.maxValue,
          },
        },

        transitions: {
          getFillColor: 250,
        },
      })
    )
  }

  // return (
  //   /*
  //   //@ts-ignore */
  //   <DeckGL
  //     layers={layers}
  //     initialViewState={initialView}
  //     controller={true}
  //     pickingRadius={5}
  //     getCursor={() => 'pointer'}
  //     onClick={handleClick}
  //   >
  //     {
  //       /*
  //       // @ts-ignore */
  //       <StaticMap
  //         reuseMaps
  //         mapStyle={dark ? MAP_STYLES.dark : MAP_STYLES.light}
  //         preventStyleDiffing={true}
  //         mapboxApiAccessToken={MAPBOX_TOKEN}
  //       />
  //     }
  //     {getTooltip({ hoverInfo })}
  //   </DeckGL>
  // )
}
</script>