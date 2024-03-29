<i18n>
en:
  all: "All"
  colors: "Colors"
  loading: "Loading"
  selectColumn: "Select data column"
  timeOfDay: "Time of day"
  bandwidths: "Widths: 1 pixel ="
  showDiffs: "Show Differences"
de:
  all: "Alle"
  colors: "Farben"
  loading: "Wird geladen"
  selectColumn: "Datenspalte wählen"
  timeOfDay: "Uhrzeit"
  bandwidths: "Linienbreiten: 1 pixel ="
  showDiffs: "Differenzen"
</i18n>

<template lang="pug">
.gl-viz(:class="{'hide-thumbnail': !thumbnail}"
        :style='{"background": urlThumbnail}' oncontextmenu="return false")

  link-deck-map.anim(v-if="!thumbnail" :props="mapProps")

  drawing-tool(v-if="!thumbnail")

  .left-side(v-if="!thumbnail")
    collapsible-panel(:darkMode="isDarkMode" :locked="true" direction="left")
      .panel-items

        //- heading
        .panel-item
          h3 {{ vizDetails.title }}
          p {{ vizDetails.description }}

        //- button/dropdown for selecting column
        .panel-item
          config-panel(
            :header="csvData.header"
            :activeColumn="csvData.activeColumn"
            :scaleWidth="scaleWidth"
            :selectedColorRamp="selectedColorRamp"
            :csvData="csvData"
            :useSlider="vizDetails.useSlider"
            @colors="clickedColorRamp"
            @column="handleNewDataColumn"
            @slider="handleNewDataColumn"
            @scale="handleScaleWidthChanged"
          )

        //- DIFF checkbox
        .panel-item.diff-section(v-if="csvBase.header.length")
          p: b {{ $t('showDiffs') }}
          toggle-button.toggle(:width="40" :value="showDiffs" :labels="false"
            :color="{checked: '#4b7cc4', unchecked: '#222'}"
            @change="showDiffs = !showDiffs")

  .nav(v-if="!thumbnail && myState.statusMessage")
    p.status-message {{ myState.statusMessage }}

</template>

<script lang="ts">
import { Vue, Component, Prop, Watch } from 'vue-property-decorator'
import { ToggleButton } from 'vue-js-toggle-button'
import { debounce } from 'debounce'
import Papaparse from 'papaparse'
import readBlob from 'read-blob'
import YAML from 'yaml'

import globalStore from '@/store'
import pako from '@aftersim/pako'
import CollapsiblePanel from '@/components/CollapsiblePanel.vue'
import TimeSlider from '@/plugins/links-gl/TimeSlider.vue'
import ConfigPanel from './ConfigPanel.vue'
import LinkDeckMap from './LinkDeckMap.vue'
import HTTPFileSystem from '@/util/HTTPFileSystem'
import DrawingTool from '@/components/DrawingTool/DrawingTool.vue'

import {
  ColorScheme,
  FileSystem,
  LegendItem,
  LegendItemType,
  FileSystemConfig,
  VisualizationPlugin,
  LIGHT_MODE,
  DARK_MODE,
  Status,
} from '@/Globals'

interface CSV {
  header: string[]
  headerMax: number[]
  rows: Float32Array
  activeColumn: number
}

@Component({
  components: {
    CollapsiblePanel,
    ConfigPanel,
    DrawingTool,
    LinkDeckMap,
    TimeSlider,
    ToggleButton,
  } as any,
})
class MyPlugin extends Vue {
  @Prop({ required: true })
  private root!: string

  @Prop({ required: true })
  private subfolder!: string

  @Prop({ required: false })
  private yamlConfig!: string

  @Prop({ required: false })
  private thumbnail!: boolean

  // store ALL the columns here, we'll just pass a reference to one column each to the view
  private buildColumnValues: Float32Array[] = []
  private baseColumnValues: Float32Array[] = []

  private mapState = { center: [0, 0], zoom: 10, bearing: 0, pitch: 20 }

  private isButtonActiveColumn = false

  private scaleWidth = 250

  private showDiffs = false
  private showTimeRange = false
  // private debounceTimeSlider = debounce(this.changedTimeSlider, 200)
  // private debounceScaleWidth = debounce(this.handleMaxWidthChanged, 500)

  private geojsonData: any[] = []

  private selectedColorRamp = 'viridis'

  private vizDetails = {
    title: '',
    description: '',
    csvFile: '',
    csvBase: '',
    useSlider: false,
    shpFile: '',
    dbfFile: '',
    geojsonFile: '',
    projection: '',
    scaleFactor: 1,
    thumbnail: '',
    sum: false,
  }

  public myState = {
    statusMessage: '',
    fileApi: undefined as HTTPFileSystem | undefined,
    fileSystem: undefined as FileSystemConfig | undefined,
    subfolder: this.subfolder,
    yamlConfig: this.yamlConfig,
    thumbnail: this.thumbnail,
  }

  private csvData: CSV = { header: [], headerMax: [], rows: new Float32Array(), activeColumn: -1 }
  private csvBase: CSV = { header: [], headerMax: [], rows: new Float32Array(), activeColumn: -1 }

  private buildData: Float32Array = new Float32Array()
  private baseData: Float32Array = new Float32Array()

  private linkOffsetLookup: { [id: string]: number } = {}
  private numLinks = 0

  private isDarkMode = this.$store.state.colorScheme === ColorScheme.DarkMode
  private isLoaded = false

  private get mapProps() {
    return {
      geojson: this.geojsonData,
      buildData: this.buildData,
      baseData: this.baseData,
      showDiffs: this.showDiffs,
      colors: this.selectedColorRamp,
      scaleWidth: this.scaleWidth,
      dark: this.isDarkMode,
      build: this.csvData,
    }
  }

  public buildFileApi() {
    const filesystem = this.getFileSystem(this.root)
    this.myState.fileApi = new HTTPFileSystem(filesystem)
    this.myState.fileSystem = filesystem
  }

  private thumbnailUrl = "url('assets/thumbnail.jpg') no-repeat;"
  private get urlThumbnail() {
    return this.thumbnailUrl
  }

  private getFileSystem(name: string) {
    const svnProject: FileSystemConfig[] = this.$store.state.svnProjects.filter(
      (a: FileSystemConfig) => a.slug === name
    )
    if (svnProject.length === 0) {
      console.log('no such project')
      throw Error
    }
    return svnProject[0]
  }

  private async getVizDetails() {
    if (!this.myState.fileApi) return

    // first get config
    try {
      const text = await this.myState.fileApi.getFileText(
        this.myState.subfolder + '/' + this.myState.yamlConfig
      )
      this.vizDetails = YAML.parse(text)
    } catch (e) {
      console.log('failed')
      // maybe it failed because password?
      if (this.myState.fileSystem && this.myState.fileSystem.needPassword && e.status === 401) {
        this.$store.commit('requestLogin', this.myState.fileSystem.slug)
      }
    }
    const t = this.vizDetails.title ? this.vizDetails.title : 'Network Links'
    this.$emit('title', t)
  }

  private async buildThumbnail() {
    if (!this.myState.fileApi) return

    if (this.thumbnail && this.vizDetails.thumbnail) {
      try {
        const blob = await this.myState.fileApi.getFileBlob(
          this.myState.subfolder + '/' + this.vizDetails.thumbnail
        )
        const buffer = await readBlob.arraybuffer(blob)
        const base64 = this.arrayBufferToBase64(buffer)
        if (base64)
          this.thumbnailUrl = `center / cover no-repeat url(data:image/png;base64,${base64})`
      } catch (e) {
        console.error(e)
      }
    }
  }

  @Watch('$store.state.colorScheme') private swapTheme() {
    this.isDarkMode = this.$store.state.colorScheme === ColorScheme.DarkMode
  }

  private arrayBufferToBase64(buffer: any) {
    var binary = ''
    var bytes = new Uint8Array(buffer)
    var len = bytes.byteLength
    for (var i = 0; i < len; i++) {
      binary += String.fromCharCode(bytes[i])
    }
    return window.btoa(binary)
  }

  private get buttonTitle() {
    if (this.csvData.activeColumn === -1) return 'Loading...'
    return this.csvData.header[this.csvData.activeColumn]
  }

  private clickedColorRamp(color: string) {
    this.selectedColorRamp = color
  }

  private handleScaleWidthChanged(value: number) {
    this.scaleWidth = value
  }

  private handleNewDataColumn(title: string) {
    console.log('handling column:', title)
    const column = this.csvData.header.indexOf(title)
    if (column === -1) return

    // // find max value for scaling
    if (!this.csvData.headerMax[column]) {
      let max = 0
      this.buildColumnValues[column].forEach(value => (max = Math.max(max, value)))
      if (max) this.csvData.headerMax[column] = max
    }

    this.buildData = this.buildColumnValues[column]
    this.baseData = this.baseColumnValues[column]

    this.csvData.activeColumn = column
    this.isButtonActiveColumn = false
  }

  private findCenter(data: any[]): [number, number] {
    return [13.45, 52.53]
  }

  private async mounted() {
    this.$store.commit('setFullScreen', !this.thumbnail)

    this.buildFileApi()

    await this.getVizDetails()
    if (this.thumbnail) return

    this.myState.statusMessage = 'Dateien laden...'

    // load network fully, first
    const allLinks = await this.loadGeojsonFeatures()
    if (!allLinks) return

    this.geojsonData = allLinks
    this.isLoaded = true

    // runs in background
    this.mapState.center = this.findCenter([])

    this.buildThumbnail()

    this.myState.statusMessage = ''

    // then load CSVs in background
    this.loadCSVFiles()
  }

  private async loadGeojsonFeatures() {
    if (!this.myState.fileApi) return

    try {
      this.linkOffsetLookup = {}
      this.numLinks = 0

      this.myState.statusMessage = 'Loading network...'

      const network = `/${this.myState.subfolder}/${this.vizDetails.geojsonFile}`
      const text = await this.myState.fileApi.getFileText(network)

      const json = JSON.parse(text)
      this.numLinks = json.features.length

      // super-efficient format is [ offset, coordsFrom[], coordsTo[] ]
      const linkElements: any[] = []

      for (let i = 0; i < this.numLinks; i++) {
        const feature = json.features[i]
        const link = [i, feature.geometry.coordinates[0], feature.geometry.coordinates[1]]
        linkElements.push(link)

        this.linkOffsetLookup[feature.properties.id] = i
      }

      return linkElements
    } catch (e) {
      this.myState.statusMessage = '' + e

      this.$store.commit('setStatus', {
        type: Status.WARNING,
        msg: `Could not find: ${this.myState.subfolder}/${this.vizDetails.geojsonFile}`,
      })

      return null
    }
  }

  private beforeDestroy() {
    this.$store.commit('setFullScreen', false)
  }

  private handleClickColumnSelector() {
    console.log('click!')
    this.isButtonActiveColumn = !this.isButtonActiveColumn
  }

  private csvFilesToLoad: string[] = []

  private loadCSVFiles() {
    this.myState.statusMessage = 'Loading CSV data...'

    // these will load sequentially and on a worker, to make things seem faster
    this.csvFilesToLoad = [this.vizDetails.csvFile]
    if (this.vizDetails.csvBase) this.csvFilesToLoad.push(this.vizDetails.csvBase)

    // start the first load. The rest will happen sequentially,
    // because papaparse will call finishedLoadingCSV() when it's done loading & parsing
    this.loadCSVFile(this.csvFilesToLoad[0])
  }

  private async finishedLoadingCSV(parsed: any) {
    const currentCSV = this.csvFilesToLoad.shift()

    console.log('FINISHED PARSING: ', currentCSV)
    this.myState.statusMessage = 'Analyzing...'

    // an array containing a separate Float32Array for each CSV column
    const allLinks: Float32Array[] = []
    const numColumns = parsed.data[0].length - (this.vizDetails.useSlider ? 0 : 1)

    console.log('number of columns', numColumns)
    for (let i = 0; i < numColumns; i++) {
      allLinks.push(new Float32Array(this.numLinks))
    }

    let globalMax = 0

    for (const link of parsed.data.splice(1)) {
      // get array offset, or skip if this link isn't in the network!
      const offset = this.linkOffsetLookup[link[0].toString()]
      if (offset === undefined) continue

      if (this.vizDetails.useSlider) {
        const entries = link.slice(1) // skip first element (contains link-id)
        const total = entries.reduce((a: number, b: number) => a + b, 0)

        globalMax = Math.max(globalMax, total)
        allLinks[0][offset] = total // total comes first
        entries.forEach((value: number, i: number) => {
          allLinks[i + 1][offset] = value
        })
      } else {
        const entries = link.slice(1) // skip first element (contains link-id)
        entries.forEach((value: number, i: number) => {
          allLinks[i][offset] = value
        })
      }
    }

    const rowZero = parsed.data[0] as string[]
    const header = rowZero.slice(1) // skip first column with link id's

    if (this.vizDetails.useSlider) header.unshift(`${this.$t('all')}`)

    //  "8 AM" is a lot narrower than "08:00:00"
    const cleanHeaders = header.map(h => h.replace(':00:00', ''))

    const details = {
      header: cleanHeaders,
      headerMax: this.vizDetails.useSlider
        ? new Array(this.csvData.header.length).fill(globalMax)
        : [],
      rows: new Float32Array(),
      activeColumn: -1,
    }

    if (this.csvFilesToLoad.length) {
      // if there's still a file to load, then we just loaded build
      this.buildColumnValues = allLinks
      this.csvData = details
      this.handleNewDataColumn(this.csvData.header[0])
    } else if (this.vizDetails.csvBase) {
      // otherwise we loaded baseline
      this.baseColumnValues = allLinks
      this.csvBase = details
    } else {
      // we are ONLY loading build
      this.buildColumnValues = allLinks
      this.csvData = details
      this.handleNewDataColumn(this.csvData.header[0])
    }

    // ARE WE DONE?
    if (this.csvFilesToLoad.length) {
      this.myState.statusMessage = 'Loading CSV Baseline...'
      this.loadCSVFile(this.csvFilesToLoad[0])
    } else {
      this.myState.statusMessage = ''
      this.handleNewDataColumn(this.csvData.header[0])
    }
  }

  private loadCSVFile(filename: string) {
    if (!this.myState.fileApi) return

    const csvFilename = this.myState.fileApi.cleanURL(`${this.myState.subfolder}/${filename}`)

    try {
      Papaparse.parse(csvFilename, {
        // preview: 10000,
        download: true,
        header: false,
        skipEmptyLines: true,
        dynamicTyping: true,
        worker: true,
        complete: this.finishedLoadingCSV,
      })
    } catch (e) {
      console.error(e)

      this.$store.commit('setStatus', {
        type: Status.WARNING,
        msg: `Could not find: ${this.myState.subfolder}/${filename}`,
      })

      return { allColumns: [], header: [], headerMax: [] }
    }
  }

  private changedTimeSlider(value: any) {
    if (value.length && value.length === 1) value = value[0]

    this.handleNewDataColumn(value)
  }
}

// !register plugin!
globalStore.commit('registerPlugin', {
  kebabName: 'links-gl',
  prettyName: 'Links',
  description: 'Network link attributes',
  filePatterns: ['viz-gl-link*.y?(a)ml'],
  component: MyPlugin,
} as VisualizationPlugin)

export default MyPlugin
</script>

<style scoped lang="scss">
@import '~vue-slider-component/theme/default.css';
@import '@/styles.scss';

.gl-viz {
  display: grid;
  pointer-events: none;
  min-height: $thumbnailHeight;
  background: url('assets/thumbnail.jpg') no-repeat;
  background-size: cover;
  grid-template-columns: auto 1fr auto;
  grid-template-rows: auto 1fr;
  grid-template-areas:
    'leftside    .  rightside'
    '.           .  rightside';
}

.gl-viz.hide-thumbnail {
  background: none;
}

.nav {
  z-index: 5;
  grid-column: 1 / 4;
  grid-row: 1 / 4;
  box-shadow: 0px 2px 10px #22222266;
  display: flex;
  flex-direction: row;
  margin: auto auto 0 0;
  background-color: var(--bgPanel);
  padding: 0rem 3rem;

  a {
    color: white;
    text-decoration: none;

    &.router-link-exact-active {
      color: white;
    }
  }

  p {
    margin: auto 0.5rem auto 0;
    font-weight: normal;
    padding: 0 0;
    color: var(--textFancy);
  }
}

.legend-block {
  margin-top: 2rem;
}

.status-message {
  padding: 0rem 0;
  font-size: 1.5rem;
  line-height: 3.25rem;
  font-weight: bold;
}

.big {
  padding: 0rem 0;
  // margin-top: 1rem;
  font-size: 2rem;
  line-height: 3.75rem;
  font-weight: bold;
}

.left-side {
  display: flex;
  flex-direction: column;
  grid-area: leftside;
  background-color: var(--bgPanel);
  font-size: 0.8rem;
  pointer-events: auto;
}

.right-side {
  z-index: 1;
  display: flex;
  flex-direction: row;
  grid-area: rightside;
  margin: 0 0 auto 0;
}

.anim {
  grid-column: 1 / 4;
  grid-row: 1 / 4;
  pointer-events: auto;
}

.panel-items {
  margin: 0.5rem 0.5rem;
}

.panel-item {
  h3 {
    line-height: 1.7rem;
    // margin-bottom: 0.5rem;
  }

  p {
    font-size: 0.9rem;
  }
}

input {
  border: none;
  background-color: var(--bgCream2);
  color: var(--bgDark);
}

.row {
  display: 'grid';
  grid-template-columns: 'auto 1fr';
}

label {
  margin: auto 0 auto 0rem;
  text-align: 'left';
}

.toggle {
  margin-bottom: 0.25rem;
  margin-right: 0.5rem;
}

#dropdown-menu-color-selector {
  background-color: var(--bgBold);

  p {
    color: #888;
  }
}

.panel-item {
  margin-top: 0rem;
}

.diff-section {
  margin-top: 1rem;
}

@media only screen and (max-width: 640px) {
  .nav {
    padding: 0.5rem 0.5rem;
  }

  .right-side {
    font-size: 0.7rem;
  }

  .big {
    padding: 0 0rem;
    margin-top: 0.5rem;
    font-size: 1.3rem;
    line-height: 2rem;
  }
}
</style>
