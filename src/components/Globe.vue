<template>
  <div>
    <Toast/>
    <div class="wrapper p-p-3">
      <div class="p-d-flex">
        <InputText type="text" v-model="domain" placeholder="For ex. google.com" class="p-mr-3" />
        <Button v-on:click="startApp" label="DISCOVER" icon="pi pi-compass" iconPos="right" class="p-text-bold"></Button>
      </div>
      <div class="p-d-flex p-flex-column" v-if="timeTaken && distance">
        <span class="p-mt-3">Total time taken (miliseconds): {{ timeTaken }}</span>
        <span class="p-mt-3">Distance in straight line (kilometers): {{ distance.toFixed(2) }}</span>
      </div>
    </div>
    <div id="globe"></div>
    <div class="footer">
      Displayed results are only approximation. Created by Matas Skaržauskas. Powered by <a href="https://vuejs.org/" alt="Vue.js">Vue.js</a>, <a href="https://globe.gl/" alt="globe.gl">globe.gl</a> and <a href="http://ip-api.com/" alt="ip-api.com">ip-api.com</a>
    </div>
  </div>
</template>

<script>
import Globe from 'globe.gl';

if (typeof (Number.prototype.toRad) === "undefined") {
    Number.prototype.toRad = function () {
        return this * Math.PI / 180;
    }
}

//-- Define degrees function
if (typeof (Number.prototype.toDeg) === "undefined") {
    Number.prototype.toDeg = function () {
        return this * (180 / Math.PI);
    }
}

export default {
  name: 'Globe',
  data: function () {
    return {
      globe: null,
      domain: '',
      timeTaken: 0,
      distance: 0,
    }
  },
  mounted() {
    this.globe = Globe({animateIn: true})
    .globeImageUrl('img/earth-night.jpg')
    .backgroundImageUrl('//unpkg.com/three-globe/example/img/night-sky.png')(document.getElementById("globe"))
  },
  methods: {
    startApp() {
      let domain = this.domain;
      if (domain.length > 0) {
        let psl = require('psl');
        domain = psl.get(this.extractHostname(this.domain))
      } else {
        this.$toast.add({severity:'error', summary: 'Enter domain address', detail:'Input field can not be empty', life: 3000});
        return;
      }

      this.axios.get('http://ip-api.com/json/').then((response) => {
        if (response.data.status === 'success') {
          this.getTargetIp(response, domain);
        }
      })
    },
    getTargetIp(start, domain) {
      let timerStart = Date.now();
      this.axios.get('http://ip-api.com/json/' + domain).then((response) => {
        if (response.data.status === 'success') {
          this.timeTaken = Date.now() - timerStart;

          const arcsData = [Array().keys()].map(() => ({
            startLat: start.data.lat,
            startLng: start.data.lon,
            endLat: response.data.lat,
            endLng: response.data.lon,
            color: ['#f1fb56', '#18dfff']
          }));

          const labelData = [
            {
              lat: start.data.lat,
              lon: start.data.lon,
              text: "You",
              color: "#f1fb56"
            },
            {
              lat: response.data.lat,
              lon: response.data.lon,
              text: (response.data.city) ? response.data.city : domain,
              color: "#18dfff"
            }
          ]

          let middlePoint = this.middlePoint(start.data.lat, start.data.lon, response.data.lat, response.data.lon);
          let distance = this.distance = this.getDistanceFromLatLonInKm(start.data.lat, start.data.lon, response.data.lat, response.data.lon);
          if (distance < 100) {
            labelData.shift();
          }
          
          this.globe
          .pointOfView({ lat: middlePoint.lat, lng: middlePoint.lon, altitude: (distance / 5000 < 0.1) ? 0.1 : distance / 5000 }, 2000)
          .arcsData(arcsData)
          .arcColor('color')
          .arcLabel('test')
          .arcDashLength(0.3)
          .arcDashGap(0.01)
          .arcDashAnimateTime(3000)
          .arcStroke(0.5)
          .labelsData(labelData)
          .labelLat(d => d.lat)
          .labelLng(d => d.lon)
          .labelText(d => d.text)
          .labelSize(0.3)
          .labelDotRadius((distance < 100) ? 0.3 : 0.7)
          .labelColor(d => d.color)
          .labelResolution(5)

        } else {
          this.$toast.add({severity:'error', summary: 'Invalid domain address', detail:'Check your input and try again', life: 3000});
        }
      })
    },
    getDistanceFromLatLonInKm(lat1, lon1, lat2, lon2) {
      var R = 6371; // Radius of the earth in km
      var dLat = this.deg2rad(lat2-lat1);  // deg2rad below
      var dLon = this.deg2rad(lon2-lon1); 
      var a = 
        Math.sin(dLat/2) * Math.sin(dLat/2) +
        Math.cos(this.deg2rad(lat1)) * Math.cos(this.deg2rad(lat2)) * 
        Math.sin(dLon/2) * Math.sin(dLon/2)
        ; 
      var c = 2 * Math.atan2(Math.sqrt(a), Math.sqrt(1-a)); 
      var d = R * c; // Distance in km
      return d;
    },
    middlePoint(lat1, lng1, lat2, lng2) {
      
        //-- Longitude difference
        var dLng = (lng2 - lng1).toRad();

        //-- Convert to radians
        lat1 = lat1.toRad();
        lat2 = lat2.toRad();
        lng1 = lng1.toRad();

        var bX = Math.cos(lat2) * Math.cos(dLng);
        var bY = Math.cos(lat2) * Math.sin(dLng);
        var lat3 = Math.atan2(Math.sin(lat1) + Math.sin(lat2), Math.sqrt((Math.cos(lat1) + bX) * (Math.cos(lat1) + bX) + bY * bY));
        var lng3 = lng1 + Math.atan2(bY, Math.cos(lat1) + bX);

        //-- Return result
        return { lat: lat3.toDeg(), lon: lng3.toDeg() };
    },
    deg2rad(deg) {
      return deg * (Math.PI/180)
    },
    extractHostname(url) {
      let hostname;
      if (url.indexOf("//") > -1) {
          hostname = url.split('/')[2];
      }
      else {
          hostname = url.split('/')[0];
      }
      hostname = hostname.split(':')[0];
      hostname = hostname.split('?')[0];

      return hostname;
    }
  }
}
</script>

<style>
html, body {
  background-image: url('//unpkg.com/three-globe/example/img/night-sky.png');
  font-family: -apple-system, BlinkMacSystemFont, "Segoe UI", Roboto, Helvetica, Arial, sans-serif, "Apple Color Emoji", "Segoe UI Emoji", "Segoe UI Symbol";
}

.wrapper {
  position: fixed;
  top: 50px;
  left: 50px;
  border-radius: .25rem;
  color: #fff;
  background: rgba(255, 255, 255, 0.1);
  z-index: 9;
  transition: all 300ms ease-in-out;
}

.footer {
  font-size: 13px;
  position: fixed;
  bottom: 1rem;
  left: 1rem;
  color: rgb(78, 78, 78);
}

.footer a {
  color: rgb(133, 133, 133);
  text-decoration: none;
}

</style>
