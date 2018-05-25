<template>
  <div class="pa-2">

    <!--layers drop box-->
    <!--used @change in case of selecting same option twice-->
    <v-select
      label="Select Time Layer"
      :items="layersSelection"
      v-model="timeData"
      item-text="name"
      @change="resetAttribute()"
    />
    <v-select
      v-if="attributesSelection.length > 1 && !attribute"
      label="Select Attribute"
      :items="attributesSelection"
      v-model="attribute"
    />
    <div v-if="openInfo">
      <p v-if="!attribute">time-attribute: {{timeData.original_time_attribute}}</p>
      <p v-if="attribute">time-attribute: {{attribute}}</p>

      <!--double range slider-->
      <section class="range-slider">
        <span class="rangeValues"></span>
        <input
          @click="getNewUrl()"
          v-model="unix1"
          :min="sliderMin"
          :max="sliderMax"
          :step="step"
          type="range">
        <input
          @click="getNewUrl()"
          v-model="unix2"
          :min="sliderMin"
          :max="sliderMax"
          :step="step"
          type="range">
      </section>

      <!--datepicker 1-->
      <v-menu
        ref="menu1"
        lazy
        :close-on-content-click="false"
        v-model="menu1"
        transition="scale-transition"
        offset-y
        full-width
        :nudge-right="40"
        :return-value.sync="pickerDate1">
        <v-text-field
          slot="activator"
          v-model="userDate1"
          prepend-icon="event"
          readonly
        ></v-text-field>
        <v-date-picker
          v-bind:class="{ 'hide-child': !hasDate }"
          v-model="pickerDate1"
          no-title
          scrollable
          :min="datePickerMinMax[0]"
          :max="pickerDate2">
          <v-spacer></v-spacer>
          <v-time-picker
            v-if="hasTime"
            v-model="pickerTime1"
            format="24hr"
            no-title
          ></v-time-picker>
          <v-btn flat color="primary" @click="menu1 = false; resetTime(1)">
            Cancel
          </v-btn>
          <v-btn class="right" flat color="primary" @click="$refs.menu1.save(pickerDate1); getNewUrl()">
            OK
          </v-btn>
        </v-date-picker>
        <div>
        </div>
      </v-menu>

      <!--datepicker 2-->
      <v-menu
        ref="menu2"
        lazy
        :close-on-content-click="false"
        v-model="menu2"
        transition="scale-transition"
        offset-y
        full-width
        :nudge-right="40"
        :return-value.sync="pickerDate2">
        <v-text-field
          slot="activator"
          v-model="userDate2"
          prepend-icon="event"
          readonly
        ></v-text-field>
        <v-date-picker
          v-bind:class="{ 'hide-child': !hasDate }"
          v-model="pickerDate2"
          no-title
          scrollable
          :min="pickerDate1"
          :max="datePickerMinMax[1]">
          <v-spacer></v-spacer>
          <v-time-picker class="time-picker" v-if="hasTime" v-model="pickerTime2" format="24hr" no-title ></v-time-picker>
          <v-btn flat color="primary" @click="menu2 = false; resetTime(2)">
            Cancel
          </v-btn>
          <v-btn class="right" flat color="primary" @click="$refs.menu2.save(pickerDate2); getNewUrl()">
            OK
          </v-btn>
        </v-date-picker>
      </v-menu>

      <!--animation-->
      <v-flex xs12>
          <v-layout row wrap>
            <v-flex xs6>
              <v-btn
                v-if="animateStop"
                @click="animate(animateStop)">
                play
              </v-btn>
              <v-btn
                v-if="!animateStop"
                @click="animate(animateStop)">
                stop
              </v-btn>
            </v-flex>
            <v-flex xs6>
              <v-switch
                class="cumulatively"
                :label="`cumulative`"
                v-model="cumulatively"
              ></v-switch>
            </v-flex>
          </v-layout>
      </v-flex>

    </div>
  </div>
</template>

<script>
  import moment from 'moment'
  import ImageLayer from 'ol/layer/image'
  import { WebgisImageWMS, layersList } from '../../map-builder'

  export default {
    inject: ['$map', '$project', '$overlays'],
    name: 'timeslide',
    icon: 'tracking',

    data () {
      return {
        layers: layersList(this.$project.layers, true),
        timeData: null,
        attribute: null,
        openInfo: false,

        // selects lists
        layersSelection: [],
        attributesSelection: [],

        // unix time
        unix1: 0,
        unix2: null,

        // slider
        userDate1: null,
        userDate2: null,
        sliderMin: 0,
        sliderMax: 0,

        // datepicker
        pickerDate1: null,
        pickerDate2: null,
        menu1: false,
        menu2: false,

        // timepicker
        pickerTime1: null,
        pickerTime2: null,

        // animate
        animateStop: true,
        frameRate: 1, // sec
        cumulatively: false
      }
    },

    computed: {
      layerModel () {
        if (this.timeData && this.timeData.selectAllLayers) {
          return this.layersSelection[0]
        } else if (this.timeData) {
          return this.layers.find(l => l.name === this.timeData.name)
        }
      },
      step () {
        return (this.sliderMax - this.sliderMin) / 100
      },
      datePickerMinMax () {
        const min = moment(this.sliderMin * 1000).format('YYYY-MM-DD')
        const max = moment(this.sliderMax * 1000).format('YYYY-MM-DD')
        return [min, max]
      }
    },

    watch: {
      // triggers after layer is selected
      timeData (value) {
        this.attributesSelection = []
        this.attribute = null

        // case of all layers
        if (value.selectAllLayers) {
          this.checkMultipleAttributes()

          // case of one attribute
          if (this.attributesSelection.length === 1) {
            this.initializeSlider(this.attributesSelection[0])
          }

        // case of one layer
        } else if (!value.selectAllLayers) {
          this.outputDateMask = value.output_datetime_mask
          this.maskIncludeDate(this.outputDateMask)
          this.hasTime = this.outputDateMask.includes('HH:mm')
          this.setSliderValue()
          this.sliderMin = this.timeData.time_values[0]
          this.sliderMax = this.timeData.time_values[1]
          this.openInfo = true
          this.getNewUrl()
        }
      },
      // contain currently selected layer
      layerModel (model) {
        if (!model.visible) {
          this.setModelVisibility(model, true)
        }
      },
      attribute (value) {
        if (value) {
          this.initializeSlider(value)
        }
      },
      // slider values
      unix1 (val) {
        this.userDate1 = moment(val * 1000).format(this.outputDateMask)
        this.pickerDate1 = moment(val * 1000).format('YYYY-MM-DD')
        this.pickerTime1 = moment(val * 1000).format('HH:mm')
        if (val >= this.unix2 - this.step) {
          this.unix2 = parseInt(val) + this.step
        }
      },
      unix2 (val) {
        this.userDate2 = moment(val * 1000).format(this.outputDateMask)
        this.pickerDate2 = moment(val * 1000).format('YYYY-MM-DD')
        this.pickerTime2 = moment(val * 1000).format('HH:mm')
        if (val <= this.unix1 + this.step) {
          this.unix1 = parseInt(val) - this.step
        }
      },
      // date picker
      pickerDate1 (val) {
        const dateAndTime = `${val}-${this.pickerTime1}`
        this.unix1 = moment(dateAndTime, 'YYYY-MM-DD-HH:mm').unix()
      },
      pickerDate2 (val) {
        const dateAndTime = `${val}-${this.pickerTime2}`
        this.unix2 = moment(dateAndTime, 'YYYY-MM-DD-HH:mm').unix()
      },
      // time picker
      pickerTime1 (val) {
        const dateAndTime = `${this.pickerDate1}-${val}`
        this.unix1 = moment(dateAndTime, 'YYYY-MM-DD-HH:mm').unix()
      },
      pickerTime2 (val) {
        const dateAndTime = `${this.pickerDate2}-${val}`
        this.unix2 = moment(dateAndTime, 'YYYY-MM-DD-HH:mm').unix()
      }
    },

    created () {
      // initialize sliders https://codepen.io/ChrisSargent/pen/meMMye?editors=1010
      let sliderSections = document.getElementsByClassName('range-slider')
      for (let x = 0; x < sliderSections.length; x++) {
        let sliders = sliderSections[x].getElementsByTagName('input')
        for (let y = 0; y < sliders.length; y++) {
          if (sliders[y].type === 'range') {
            sliders[y].oninput = this.getSliderVals()
            sliders[y].oninput()
          }
        }
      }

      // add "select all layers" into layer select
      this.addAllIntoSelection()

      // disable map cashing
      const map = this.$map
      if (!(map.overlay instanceof ImageLayer)) {
        // create and switch to WMS layer
        this.originalLayer = map.overlay
        this.originalLayer.setVisible(false)

        this.layer = new ImageLayer({
          visible: true,
          extent: this.$project.project_extent,
          source: new WebgisImageWMS({
            resolutions: this.$project.tile_resolutions,
            url: this.$project.ows_url,
            visibleLayers: this.originalLayer.getSource().getVisibleLayers(),
            layersAttributions: {},
            params: {
              'FORMAT': 'image/png'
            },
            serverType: 'qgis',
            ratio: 1
          })
        })

        // set as new main map's layer
        map.addLayer(this.layer)
        map.overlay = this.layer
      }
    },

    beforeDestroy () {
      this.animateStop = true
      // remove all extra parameters and reset names
      for (let i = 0; i < this.layers.length; i++) {
        this.layers[i].title = this.layers[i].name
        delete this.layers[i].unix1
        delete this.layers[i].unix2
        delete this.layers[i].timeFilter
      }
      if (this.originalLayer) {
        // switch back to original cached layer
        const map = this.$map
        map.removeLayer(this.layer)
        map.overlay = this.originalLayer
        map.overlay.getSource().setVisibleLayers(this.layer.getSource().getVisibleLayers())
        map.overlay.setVisible(true)
      }
    },

    methods: {
      // set selected layer visible
      setModelVisibility (model, visible) {
        const visibleLayers = this.$overlays.list.filter(l => l.visible)
        if (visible && !model.selectAllLayers) {
          visibleLayers.push(model)
          model._visible = true
          model.visible = true
        }
        this.layer.getSource().setVisibleLayers(visibleLayers.map(l => l.name))
      },

      // initialize slider min,max and values -- multiple layers
      initializeSlider (attribute) {
        const visibleLayers = this.$overlays.list.filter(l => l.visible && l.original_time_attribute && l.original_time_attribute === attribute)
        this.setDateMask(visibleLayers)
        this.maskIncludeDate(this.outputDateMask)
        this.hasTime = this.outputDateMask.includes('HH:mm')
        const minmax = this.getSliderRange(visibleLayers)
        this.sliderMin = minmax[0]
        this.sliderMax = minmax[1]
        this.unix1 = minmax[0]
        this.unix2 = minmax[0]
        this.openInfo = true
      },

      // set slider values by last used one
      setSliderValue () {
        if (this.layerModel.unix1) {
          this.unix1 = this.layerModel.unix1
        } else {
          this.unix1 = this.timeData.time_values[0]
        }
        if (this.layerModel.unix2) {
          this.unix2 = this.layerModel.unix2
        } else {
          this.unix2 = this.timeData.time_values[0]
        }
      },

      // make filter for non selected time layers
      getFilterFromLayers (layers, layerModel) {
        const modelIndex = layers.indexOf(layerModel)
        let filter = ''
        for (let i = 0; i < layers.length; i++) {
          if (layers[i].timeFilter && i !== modelIndex) {
            filter += `;${layers[i].timeFilter}`
          }
        }
        return filter
      },

      // update WMS url
      getNewUrl () {
        if (this.layerModel.selectAllLayers) {
          this.updateMultipleLayers()
        } else {
          this.updateSingleLayer()
        }
      },

      updateSingleLayer () {
        console.log('SINGLE')
        const otherLayerFilter = this.getFilterFromLayers(this.layers, this.layerModel)
        let modelFilter = ''
        if (this.timeData.unix) {
          modelFilter = `${this.timeData.name}:"${this.timeData.time_attribute}" >= '${this.unix1}' AND "${this.timeData.time_attribute}" <= '${this.unix2}'`
        } else {
          const dateTimeUnix1 = moment(this.unix1 * 1000).format(this.timeData.input_datetime_mask)
          const dateTimeUnix2 = moment(this.unix2 * 1000).format(this.timeData.input_datetime_mask)
          modelFilter = `${this.timeData.name}:"${this.timeData.original_time_attribute}" >= '${dateTimeUnix1}' AND "${this.timeData.original_time_attribute}" <= '${dateTimeUnix2}'`
        }
        const filter = `${modelFilter}${otherLayerFilter}`
        this.layerModel.title = `${this.timeData.name}-${moment(this.unix1 * 1000).format(this.outputDateMask)}`
        this.layerModel.unix1 = this.unix1
        this.layerModel.unix2 = this.unix2
        this.layerModel.timeFilter = modelFilter
        this.oldPickerTime1 = moment(this.unix1 * 1000).format('HH:mm')
        this.oldPickerTime2 = moment(this.unix2 * 1000).format('HH:mm')
        this.layer.getSource().updateParams({'FILTER': filter})
      },

      updateMultipleLayers () {
        console.log('MULTIPLE')
        const attribute = this.attribute || this.attributesSelection[0]
        const visibleLayers = this.$overlays.list.filter(l => l.visible && l.original_time_attribute)
        let filterIncrement = ''
        let filter = ''
        for (let i = 0; i < visibleLayers.length; i++) {
          if (visibleLayers[i].original_time_attribute === attribute) {
            if (visibleLayers[i].unix) {
              filterIncrement = `;${visibleLayers[i].name}:"${visibleLayers[i].time_attribute}" >= '${this.unix1}' AND "${visibleLayers[i].time_attribute}" <= '${this.unix2}'`
            } else {
              const dateTimeUnix1 = moment(this.unix1 * 1000).format(visibleLayers[i].input_datetime_mask)
              const dateTimeUnix2 = moment(this.unix2 * 1000).format(visibleLayers[i].input_datetime_mask)
              filterIncrement = `;${visibleLayers[i].name}:"${visibleLayers[i].original_time_attribute}" >= '${dateTimeUnix1}' AND "${visibleLayers[i].original_time_attribute}" <= '${dateTimeUnix2}'`
            }
            filter += filterIncrement
            visibleLayers[i].title = `${visibleLayers[i].name}-${moment(this.unix1 * 1000).format(this.outputDateMask)}`
            visibleLayers[i].unix1 = this.unix1
            visibleLayers[i].unix2 = this.unix2
            visibleLayers[i].timeFilter = filterIncrement
          } else {
            filter += `;${visibleLayers[i].timeFilter}`
          }
        }
        this.oldPickerTime1 = moment(this.unix1 * 1000).format('HH:mm')
        this.oldPickerTime2 = moment(this.unix2 * 1000).format('HH:mm')
        this.layer.getSource().updateParams({'FILTER': filter})
      },

      // in case that one layer is selected twice
      resetAttribute () {
        this.cumulatively = false
        this.animateStop = true
        if (this.timeData && this.timeData.selectAllLayers) {
          this.attribute = null
          this.openInfo = false
          this.attributesSelection = []
          this.checkMultipleAttributes()
          if (this.attributesSelection.length === 1) {
            this.attribute = this.attributesSelection[0]
            this.initializeSlider(this.attributesSelection[0])
          }
        }
      },

      // add "All visible layers" option into layers select
      addAllIntoSelection () {
        const all = {
          name: 'All visible layers',
          title: 'All visible layers',
          selectAllLayers: true
        }
        this.layersSelection.push(all)
        this.$project.layers.forEach(l => { this.layersSelection.push(l) })
      },

      // find unique attributes
      checkMultipleAttributes () {
        this.openInfo = false
        const visibleLayers = this.$overlays.list.filter(l => l.visible && l.original_time_attribute)
        this.attributesSelection = [...new Set(visibleLayers.map(l => l.original_time_attribute))]
      },

      // get min and max slider range
      getSliderRange (visibleLayers) {
        let min = 1E+100
        let max = -1E+100
        for (let i = 0; i < visibleLayers.length; i++) {
          if (min > visibleLayers[i].time_values[0]) {
            min = visibleLayers[i].time_values[0]
          }
          if (max < visibleLayers[i].time_values[1]) {
            max = visibleLayers[i].time_values[1]
          }
        }
        return [min, max]
      },

      // set date mask in cas of multiple layers
      setDateMask (visibleLayers) {
        for (let i = 0; i < visibleLayers.length; i++) {
          if (visibleLayers[i].output_datetime_mask.includes('HH:mm') && visibleLayers[i].output_datetime_mask.includes('YYYY')) {
            this.outputDateMask = visibleLayers[i].output_datetime_mask
            this.hasTime = true
            return
          }
        }
        for (let i = 0; i < visibleLayers.length; i++) {
          if (visibleLayers[i].output_datetime_mask.includes('YYYY')) {
            this.outputDateMask = visibleLayers[i].output_datetime_mask
            this.hasTime = true
            return
          }
        }
        this.outputDateMask = visibleLayers[0].output_datetime_mask
        this.hasTime = false
      },

      // double slider range functionality https://codepen.io/ChrisSargent/pen/meMMye?editors=1010
      getSliderVals () {
        // get slider values
        let parent = this.parentNode
        let slides = parent.getElementsByTagName('input')
        let slide1 = parseFloat(slides[0].value)
        let slide2 = parseFloat(slides[1].value)
        let displayElement = parent.getElementsByClassName('rangeValues')[0]
        displayElement.innerHTML = `$ ${slide1}k - $${slide2}k`
      },

      // set old time from date picker in case of "cancel" button pressed
      resetTime (num) {
        if (num === 1) {
          const dateAndTime = `${this.pickerDate1}-${this.oldPickerTime1}`
          this.unix1 = moment(dateAndTime, 'YYYY-MM-DD-HH:mm').unix()
        } else {
          const dateAndTime = `${this.pickerDate2}-${this.oldPickerTime2}`
          this.unix2 = moment(dateAndTime, 'YYYY-MM-DD-HH:mm').unix()
        }
      },

      // simple animation
      animate (play) {
        if (play) {
          this.animateStop = false
          this.newFrame()
        } else {
          this.animateStop = true
        }
      },

      // one animation step
      newFrame () {
        if (this.cumulatively) {
          if (this.unix2 < this.sliderMax - this.step) {
            this.unix2 += this.step
          } else if (this.unix1 < this.sliderMax - 2 * this.step) {
            this.unix1 += this.step
          } else {
            this.animateStop = true
            return
          }
        } else {
          if (this.unix2 < this.sliderMax - this.step) {
            this.unix1 += this.step
            this.unix2 += this.step
          } else {
            this.animateStop = true
            return
          }
        }
        this.getNewUrl()
        if (!this.animateStop) {
          setTimeout(this.newFrame, this.frameRate * 1000)
        }
      },

      // check if mask includes year, month or days
      maskIncludeDate (mask) {
        if (
          mask.includes('YYYY') || mask.includes('MM') || mask.includes('DD')
        ) {
          this.hasDate = true
        } else {
          this.hasDate = false
        }
      }
    }
  }
</script>

<style lang="scss">

  /*cumulate switch*/
  .cumulatively {
    margin: 10px 0 -10px;
  }

  /*time picker*/

  .hide-child > :first-child{
    display: none;
  }

  .time-picker {
    display: flex !important;
  }

  .picker__actions {
    display: block;
    padding: 8px 0;
  }

  .picker--time {
    margin: 0;
  }

  .picker {
    display: inherit;
  }

  .right {
    float: right;
  }

  /*slider*/

  @mixin range-slider($width, $height, $input-top, $input-bg-color, $input-thumb-color, $float:none, $input-height:20px, $input-border-radius:14px) {
    position: relative;
    width: $width;
    height: $height;
    float: $float;
    text-align: center;

    input[type="range"] {
      pointer-events: none;
      position: absolute;
      -webkit-appearance: none;
      -webkit-tap-highlight-color: rgba(255, 255, 255, 0);
      border: none;
      border-radius: $input-border-radius;
      background: $input-bg-color;
      box-shadow: inset 0 1px 0 0 darken($input-bg-color,15%), inset 0 -1px 0 0 darken($input-bg-color,10%);
      -webkit-box-shadow: inset 0 1px 0 0 darken($input-bg-color,15%), inset 0 -1px 0 0 darken($input-bg-color,10%);
      overflow: hidden;
      left: 0;
      top: $input-top;
      width: $width;
      outline: none;
      height: $input-height;
      margin: 0;
      padding: 0;
    }

    input[type="range"]::-webkit-slider-thumb {
      pointer-events: all;
      position: relative;
      z-index: 1;
      outline: 0;
      -webkit-appearance: none;
      width: $input-height;
      height: $input-height;
      border: none;
      border-radius: $input-border-radius;
      background-image: -webkit-gradient(linear, left top, left bottom, color-stop(0%, lighten($input-thumb-color,60%)), color-stop(100%, $input-thumb-color)); /* android <= 2.2 */
      background-image: -webkit-linear-gradient(top , lighten($input-thumb-color,60%) 0, $input-thumb-color 100%); /* older mobile safari and android > 2.2 */;
      background-image: linear-gradient(to bottom, lighten($input-thumb-color,60%) 0, $input-thumb-color 100%); /* W3C */
    }

    input[type="range"]::-moz-range-thumb {
      pointer-events: all;
      position: relative;
      z-index: 10;
      -moz-appearance: none;
      width: $input-height;
      height: $input-height;
      border: none;
      border-radius: $input-border-radius;
      background-image: linear-gradient(to bottom, lighten($input-thumb-color,60%) 0, $input-thumb-color 100%); /* W3C */
    }

    input[type="range"]::-ms-thumb {
      pointer-events: all;
      position: relative;
      z-index: 10;
      -ms-appearance: none;
      width: $input-height;
      height: $input-height;
      border-radius: $input-border-radius;
      border: 0;
      background-image: linear-gradient(to bottom, lighten($input-thumb-color,60%) 0, $input-thumb-color 100%); /* W3C */
    }

    input[type=range]::-moz-range-track {
      position: relative;
      z-index: -1;
      background-color: rgba(0, 0, 0, 1);
      border: 0;
    }

    input[type=range]:last-of-type::-moz-range-track {
      -moz-appearance: none;
      background: none transparent;
      border: 0;
    }

    input[type=range]::-moz-focus-outer {
      border: 0;
    }
  }

  section.range-slider {
    @include range-slider(100%, 50px, 5px, #F1EFEF, #413F41, left);
  }

</style>
