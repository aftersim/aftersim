<template lang="pug">
#vega-container(v-if="myState.yamlConfig")
  .main-area
    .labels(v-show="myState.thumbnail")
      h5.center {{ vizDetails.description }}

    .labels(v-show="!(myState.thumbnail)")
      h3.center {{ title }}
      h5.center {{ description }}


    .chart-area.center-chart(v-if="hasFacets")
      .vega-chart(:id="cleanConfigId"
                  :style="{padding: thumbnail ? '0 0' : '1rem 1rem'}")

    .vega-chart(v-if="!hasFacets"
                :id="cleanConfigId"
                :style="{padding: thumbnail ? '0 0' : '1rem 1rem'}"
    )
</template>

<script lang="ts">
'use strict'

import nprogress from 'nprogress'
import vegaEmbed from 'vega-embed'
import yaml from 'yaml'
import { Vue, Component, Prop, Watch } from 'vue-property-decorator'

import globalStore from '@/store'
import { FileSystem, FileSystemConfig, VisualizationPlugin } from '../../Globals'
import HTTPFileSystem from '@/util/HTTPFileSystem'

@Component({ components: {} })
class VegaComponent extends Vue {
  @Prop({ required: false })
  private fileApi!: FileSystem

  @Prop({ required: false })
  private subfolder!: string

  @Prop({ required: false })
  private yamlConfig!: string

  @Prop({ required: false })
  private thumbnail!: boolean

  private globalState = globalStore.state

  private myState = {
    fileApi: this.fileApi,
    fileSystem: undefined as FileSystemConfig | undefined,
    subfolder: this.subfolder,
    yamlConfig: this.yamlConfig,
    thumbnail: this.thumbnail,
  }

  private vizDetails: any = { title: '', description: '' }

  private title = ''
  private description = ''

  private loadingText: string = 'Flow Diagram'
  private jsonChart: any = {}
  private totalTrips = 0

  private get cleanConfigId() {
    return this.myState.yamlConfig.replace(/[\W_]+/g, '')
  }

  public async mounted() {
    if (!this.yamlConfig) this.buildRouteFromUrl()

    await this.getVizDetails()
  }

  @Watch('globalState.authAttempts') authenticationChanged() {
    console.log('AUTH CHANGED - Reload')
    this.getVizDetails()
  }

  @Watch('yamlConfig') changedYaml() {
    this.myState.yamlConfig = this.yamlConfig
    this.getVizDetails()
  }

  @Watch('subfolder') changedSubfolder() {
    this.myState.subfolder = this.subfolder
    this.getVizDetails()
  }

  private get hasFacets() {
    if (!this.vizDetails) return false
    if (!this.vizDetails.encoding) return false

    if (
      this.vizDetails.encoding.facet ||
      this.vizDetails.encoding.row ||
      this.vizDetails.encoding.column
    )
      return true

    return false
  }

  private getFileSystem(name: string) {
    const svnProject: FileSystemConfig[] = globalStore.state.svnProjects.filter(
      (a: FileSystemConfig) => a.slug === name
    )
    if (svnProject.length === 0) {
      console.log('no such project')
      throw Error
    }
    return svnProject[0]
  }

  // this happens if viz is the full page, not a thumbnail on a project page
  private buildRouteFromUrl() {
    const params = this.$route.params
    if (!params.project || !params.pathMatch) {
      console.log('I CANT EVEN: NO PROJECT/PARHMATCH')
      return
    }

    // project filesystem
    const filesystem = this.getFileSystem(params.project)
    this.myState.fileApi = new HTTPFileSystem(filesystem)
    this.myState.fileSystem = filesystem

    // subfolder and config file
    const sep = 1 + params.pathMatch.lastIndexOf('/')
    const subfolder = params.pathMatch.substring(0, sep)
    const config = params.pathMatch.substring(sep)

    this.myState.subfolder = subfolder
    this.myState.yamlConfig = config
  }

  private async getVizDetails() {
    this.vizDetails = await this.loadFiles()
    if (this.vizDetails) this.jsonChart = this.processInputs()

    this.loadingText = ''
    nprogress.done()
  }

  private async loadFiles() {
    let json: any = { data: {} }

    try {
      this.loadingText = 'Loading chart...'

      json = await this.myState.fileApi.getFileJson(
        this.myState.subfolder + '/' + this.myState.yamlConfig
      )

      this.description = json.description ? json.description : ''
      this.title = json.title
        ? json.title
        : this.myState.yamlConfig
            .substring(0, this.myState.yamlConfig.length - 10)
            .replace(/_/g, ' ')

      this.$emit('title', this.title)
    } catch (e) {
      console.error({ e })
      this.loadingText = '' + e

      // maybe it failed because password?
      if (this.myState.fileSystem && this.myState.fileSystem.needPassword && e.status === 401) {
        globalStore.commit('requestLogin', this.myState.fileSystem.slug)
      }
      return
    }

    // if there is a URL in the schema, try to load/download it locally first
    if (json.data.url) {
      try {
        let data = ''
        const localUrl = this.myState.subfolder + '/' + json.data.url
        if (json.data.url.endsWith('.json')) {
          data = await this.myState.fileApi.getFileJson(localUrl)
          if (data) json.data = { values: data }
        } else if (json.data.url.endsWith('.csv')) {
          data = await this.myState.fileApi.getFileText(localUrl)
          if (data) json.data = { values: data, format: { type: 'csv' } }
        }
      } catch (e) {
        // didn't work -- let Vega try on its own.
        console.warn(e)
      }
    }
    return json
  }

  private processInputs() {
    this.loadingText = 'Building chart...'

    const exportActions = { export: true, source: false, compiled: false, editor: false }

    const embedOptions = {
      actions: this.thumbnail ? false : exportActions,
      hover: true,
      scaleFactor: 2.0, // make exported PNGs bigger
      padding: { top: 2, left: 8, right: 8, bottom: 8 },
    }

    // remove legends on thumbnails so chart fits better
    if (this.thumbnail && this.vizDetails.encoding) {
      for (const layer of Object.keys(this.vizDetails.encoding)) {
        this.vizDetails.encoding[layer].legend = null
      }
    }

    vegaEmbed(`#${this.cleanConfigId}`, this.vizDetails, embedOptions)
  }
}

// !register plugin!
globalStore.commit('registerPlugin', {
  kebabName: 'vega-lite',
  prettyName: 'Chart',
  description: 'Interactive chart visualization',
  filePatterns: ['*.vega.json'],
  component: VegaComponent,
} as VisualizationPlugin)

export default VegaComponent
</script>

<style scoped>
#vega-container {
  width: 100%;
  display: grid;
  background-color: white;
  grid-template-columns: auto 1fr;
  grid-template-rows: auto auto;
}

h1 {
  margin: 0px auto;
  font-size: 1.5rem;
}

h3 {
  margin: 0px auto;
}

h4,
p {
  margin: 1rem 1rem;
}

.details {
  font-size: 12px;
}

.bigtitle {
  font-weight: bold;
  font-style: italic;
  font-size: 20px;
  margin: 20px 0px;
}

.info-header {
  background-color: #097c43;
  padding: 0.5rem 0rem;
  border-top: solid 1px #888;
  border-bottom: solid 1px #888;
}

.main-area {
  padding-top: 1rem;
  grid-row: 1/3;
  grid-column: 1/3;
  width: 100%;
  display: flex;
  flex-direction: column;
}

.labels {
  margin-bottom: 1rem;
}

.center-chart {
  margin: 0 auto;
}

.vega-chart {
  width: 100%;
  max-width: 60rem;
  height: auto;
  margin: 0px auto;
}

.center {
  text-align: center;
}
</style>
