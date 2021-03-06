<template>
<modal override-width="1200px" :name="name" :value="value">
  <div class="modal-header">
    <h3>
      {{route.label}}: {{route.from}} &mdash; {{route.to}}

      <button @click="resolve()" class="btn btn-default">
        <span class="glyphicon glyphicon-remove"></span>
      </button>
    </h3>
  </div>

  <div class="modal-body">
    <gmap-map :center="{lat: 1.38, lng: 103.8}" :zoom="12" ref="map" class="view-trips-map">
      <!-- Route path -->
      <gmap-polyline v-if="routePath" :path="routePath" />

      <!-- Pings by other drivers? -->
      <PingPath v-for="(pings, driverId) in pingsByDriverId"
                 :pings="pings"
                 :key="driverId"
                 @click="selectedPing = $event"
      />

      <!-- Trip stops -->
      <template v-if="selectedTrip">
        <TripStopMarker
            v-for="(ts, index) in selectedTrip.tripStops"
            :tripStop="ts"
            :index="index"
            :key="ts.id"
            @click="selectedStop = ts"
          />
      </template>

      <gmap-info-window
        v-if="selectedStop"
        :position="f.pointToLatLng(selectedStop.stop.coordinates)"
        :opened="selectedStop !== undefined"
        @closeclick="selectedStop = undefined"
        >
        <div>
          <b>{{selectedStop.stop.description}}</b>
          <br/>
          {{f.date(selectedStop.time, 'HH:MM')}}
        </div>
      </gmap-info-window>

      <gmap-info-window
        v-if="selectedPing"
        :position="f.pointToLatLng(selectedPing.coordinates)"
        :opened="selectedPing !== undefined"
        @closeclick="selectedPing = undefined"
        >
        <div>
          <b>{{f.date(selectedPing.time, 'HH:MM:ss')}}</b>
          <br/>
          Driver Id: #{{selectedPing.driverId}} {{selectedPingDriverVehicle && selectedPingDriverVehicle.driver.name}}
          <br/>
          Vehilcle Id: #{{selectedPing.vehicleId}} {{selectedPingDriverVehicle && selectedPingDriverVehicle.vehicleNumber}}
        </div>
      </gmap-info-window>
    </gmap-map>
    <div>
      <date-picker
        class="date-picker-trips"
        :multiple="false"

        v-model="selectedTrip"
        :specialDates="trips.map(trip => ({ date: trip.date, enabled: true, annotation: 1, rawAnnotation: trip }))"
        :defaultDisable="true"
        :otherMonthSelectable="true"
        :toModel="(d) => d.rawAnnotation"
        :extractDateFromModel="extractDateFromModel"
        >
      </date-picker>
    </div>
  </div>
</modal>
</template>
<script>
import {mapState, mapActions} from 'vuex'
import _ from 'lodash'

import DatePicker from '@/components/DatePicker.vue'
import filters from '@/filters'

export default {
  props: ['route', 'value'],
  components: {
    DatePicker
  },
  mixins: [
    require('./ModalMixin')
  ],
  data () {
    return {
      routeWithTrips: null,
      selectedTrip: null,
      selectedStop: null,
      selectedPing: null,
      pingsByDriverId: null,
      routePath: null,
    }
  },
  created() {
    this.fetch('vehicles')
  },
  computed: {
    ...mapState(['axios']),
    ...mapState('shared', ['vehicles']),
    f: () => filters,
    routePromise() {
      if (!this.route) return

      return this.getRoute({
        id: this.route.id,
        options: {includeTrips: true}
      })
    },
    pingsPromise() {
      if (!this.selectedTrip) return

      return this.getPings({
        tripId: this.selectedTrip.id,
        options: {
          limit: 10000,
          // TODO: flexible time boundaries for night
          startTime: new Date(
            this.selectedTrip.date.getUTCFullYear(),
            this.selectedTrip.date.getUTCMonth(),
            this.selectedTrip.date.getUTCDate()
          ).toISOString(),
          endTime: new Date(
            this.selectedTrip.date.getUTCFullYear(),
            this.selectedTrip.date.getUTCMonth(),
            this.selectedTrip.date.getUTCDate() + 1
          ).toISOString(),
          byTripId: true,
        }
      })
    },
    selectedPingDriverVehicle() {
      if (!this.vehicles || !this.selectedPing) return

      return this.vehicles.find((value) => {
        return value.id === this.selectedPing.vehicleId && value.driverId === this.selectedPing.driverId
      })
    },
    trips () {
      return this.routeWithTrips
        ? _.orderBy(this.routeWithTrips.trips, ['date'], ['desc']) : []
    },
    routePathPromise () {
      const promise = this.routePromise &&
        this.routePromise.then((route) => {
          const path = route.path

          return typeof path === 'string'
            ? google.maps.geometry.encoding.decodePath(path)
            : path

          return path
        })
      return promise
    }
  },
  watch: {
    routePromise: {
      immediate: true,
      handler(p) {
        if (p) p.then(route => {
          this.routeWithTrips = route
        })
      }
    },
    routePathPromise: {
      immediate: true,
      handler(p) {
        if (p) p.then(path => this.routePath = path)
      }
    },
    pingsPromise: {
      immediate: true,
      handler(p) {
        if (p) p.then((pings) => this.pingsByDriverId = _.groupBy(pings, 'driverId'))
      }
    },
    selectedTrip: {
      immediate: false,
      handler(selectedTrip) {
        if (selectedTrip) {
          var bounds = new google.maps.LatLngBounds();
          for (let tripStop of selectedTrip.tripStops) {
            bounds.extend({
              lat: tripStop.stop.coordinates.coordinates[1],
              lng: tripStop.stop.coordinates.coordinates[0]
            })
          }
          this.$refs.map.panToBounds(bounds)
        }
      }
    }
  },
  methods: {
    ...mapActions('resources', ['getRoute', 'getPings']),
    ...mapActions('shared', ['fetch']),
    extractDateFromModel: (trip, effectiveOffset) => new Date(trip.date.getTime() + effectiveOffset),
  }
}
</script>

<style scoped>
.modal-dialog {
  width: 75%;
}
.modal-body {
  display: flex;
  flex-direction: row;
}
.modal-body > * {
  flex: 1 1 50%;
}
.view-trips-map {
  width: 50%;
  height: 400px;
}
</style>
<style lang="scss">
.date-picker-trips {
  width: 100%;
  td, th {
    text-align: center;
    line-height: 3.0;
    position: relative;
    cursor: pointer;

    &.selected {
      background-color: #008;
      color: #FFF;
    }
    &.disabled {
      background-color: #888;
      color: #CCC;
    }
    &.different-month {
      opacity: 0.5;
    }
    &:not(.different-month) {
      font-weight: bold;
    }
    &.public-holiday {
      color: #F00;
    }
    div.annotation {
      background-color: #FF6C6A;
      color: #F4F4F4;
      position: absolute;
      bottom: 0;
      right: 0;
      line-height: 1.6;
      font-size: 12px;
      padding: 1px 5px;
    }
  }
  th:not([colspan]) {
    width: 14%;
  }
}
</style>
