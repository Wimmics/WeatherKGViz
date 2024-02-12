<script>
//Imports
import { latLngBounds, icon } from "leaflet";
import "leaflet/dist/leaflet.css";
import {
  LMap,
  LTileLayer,
  LGeoJson,
  LMarker,
  LPopup,
  LTooltip,
} from "@vue-leaflet/vue-leaflet";
import axios from "axios";
import { toRaw } from "vue";
import regionsJson from "../assets/regions.json";
import * as d3 from "d3";
import { rgb, scaleOrdinal, schemeCategory10 } from "d3";

import {
  getAvgTempPerRegion,
  buildQuery_getAllStationsAvgTemp,
  buildQuery_avgRainQtyPerStation,
  buildQuery_getAllStationsAvgHumidity,
  buildQuery_getAllStationsAvgWindSpeed,
  buildQuery_getAllStationsAvgWindDirection,
  buildQuery_slices,
} from "../queries/queries.js";
import hsl from "hsl-to-hex";
import img from "../assets/arrow.svg";
import img2 from "../assets/hexagon.svg";
//
export default {
  //Nom du component
  name: "Map",
  //Components utilisés
  components: {
    LMap,
    LTileLayer,
    LGeoJson,
    LMarker,
    LPopup,
    LTooltip,
  },
  //
  computed: {
    stations() {
      return this.$store.getters.getStations;
    },
  },
  //Variables définies au chargement du component
  mounted() {
    this.getavg = this.AvgTemp();
    this.getstationAvg = this.AvgStationTemp(buildQuery_getAllStationsAvgTemp);
    this.getstationRain = this.AvgStationTemp(buildQuery_avgRainQtyPerStation);
    this.getstationWindSpeed = this.AvgStationTemp(
      buildQuery_getAllStationsAvgWindSpeed
    );
    this.getstationWindDirection = this.AvgStationTemp(
      buildQuery_getAllStationsAvgWindDirection
    );
    this.getstationHumidity = this.AvgStationTemp(
      buildQuery_getAllStationsAvgHumidity
    );
  },
  //Propriétés définies au chargement du component

  data() {
    return {
      zoom: 6,
      show: true,
      showMark: false,
      showTime: false,
      showTemp: false,
      showRain: false,
      showHumidity: false,
      showWind: false,
      reload: true,
      loading: false,
      defaultYear: 2022,
      geoJson: regionsJson,
      yearSlide: 2022,
      geoJsonOptions: {
        onEachFeature: this.onEachFeature,
      },
      getavg: "",
      getstationAvg: "",
      getstationRain: "",
      getstationWindSpeed: "",
      getstationWindDirection: "",
      getstationHumidity: "",
      region: "region",
      start: "2022-01-01",
      end: "2022-12-31",
      dataD3: "",
      stationsD3: "",
      inseeD3: "",
      dataD3Copy: "",
    };
  },
  //Fonctions du component

  methods: {
    handleRangeChangedEvent(range) {
      this.dataD3 = this.filterDataByDateRange(range[0], range[1]);

      this.createLineChart();
    },

    filterDataByDateRange(startDate, endDate) {
      return this.dataD3Copy.filter((item) => {
        const itemDate = new Date(item.date.value);
        return itemDate >= new Date(startDate) && itemDate <= new Date(endDate);
      });
    },
    getMinMaxTemperatures(datas) {
      const temperatureValues = datas.map((item) => item.temp_avg.value);
      const minTemp = Math.min(...temperatureValues);
      const maxTemp = Math.max(...temperatureValues);
      return [minTemp, maxTemp];
    },
    createLineChart() {
      var remv = d3.select("#evol");
      remv.selectAll("*").remove();
      const width = 900;
      const height = 200;
      const svg = d3
        .select("#evol")
        .attr("width", width)
        .attr("height", height);
      //2. Parse the dates

      //3. Creating the Chart Axes
      const x = d3
        .scaleTime()
        .domain(
          d3.extent(this.dataD3, function (d) {
            return new Date(d.date.value);
          })
        )
        .range([0, width]);
      const y = d3.scaleLinear().range([height, 0]);

      y.domain(this.getMinMaxTemperatures(this.dataD3));

      //4. Creating a Line
      const line = d3
        .line()
        .curve(d3.curveCardinal)
        .x((d) => x(new Date(d.date.value)))
        .y((d) => y(d.temp_avg.value));
      //5. Appending the Axes to the Chart
      var paddingheight = height - 23;
      svg
        .append("g")
        .attr("transform", "translate(30," + paddingheight + ")")
        .call(d3.axisBottom(x));

      // Append the y axis
      svg
        .append("g")
        .attr("transform", "translate(30,-23)")
        .call(d3.axisLeft(y));
      const colorScale = scaleOrdinal(schemeCategory10);

      for (const station of this.stationsD3) {
        const color = rgb(colorScale(station));
        svg
          .append("path")
          .datum(this.dataD3.filter((d) => d.Nstation.value === station))
          .attr("fill", "none")
          .attr("transform", "translate(30,-23)")

          .attr("stroke", color.toString())
          .attr("stroke-width", 1)
          .attr("d", line);
      }
    },
    //Création du diagramme D3
    async d3Test(insee, year) {
      this.yearSlide = year;
      var remv = d3.select("#evol");
      remv.selectAll("*").remove();
      remv = d3.select("#evol_Legend");
      remv.selectAll("*").remove();
      //1. Get Values
      this.dataD3 = await axios.post(
        import.meta.env.VITE_API_URL,
        {
          query: buildQuery_slices(insee, year),
        },
        {
          headers: {
            "Content-Type": "application/x-www-form-urlencoded",
          },
          responseType: "json",
        }
      );
      this.dataD3 = this.dataD3.data.results.bindings;
      this.dataD3Copy = this.dataD3;
      this.stationsD3 = this.dataD3.map((element) => element.Nstation.value);
      this.stationsD3 = Array.from(new Set(this.stationsD3));

      const width = 900;
      const height = 200;
      const svg = d3
        .select("#evol")
        .attr("width", width)
        .attr("height", height);
      //2. Parse the dates

      //3. Creating the Chart Axes
      const x = d3
        .scaleTime()
        .domain(
          d3.extent(this.dataD3, function (d) {
            return new Date(d.date.value);
          })
        )
        .range([0, width]);
      const y = d3.scaleLinear().range([height, 0]);

      y.domain(this.getMinMaxTemperatures(this.dataD3));

      //4. Creating a Line
      const line = d3
        .line()
        .curve(d3.curveCardinal)
        .x((d) => x(new Date(d.date.value)))
        .y((d) => y(d.temp_avg.value));
      //5. Appending the Axes to the Chart
      var paddingheight = height - 23;

      svg
        .append("g")
        .attr("transform", "translate(30," + paddingheight + ")")
        .attr("class", "fade-in")

        .call(d3.axisBottom(x));

      // Append the y axis
      svg
        .append("g")
        .attr("transform", "translate(30,-23)")
        .attr("class", "fade-in")
        .call(d3.axisLeft(y));
      const colorScale = scaleOrdinal(schemeCategory10);

      for (const station of this.stationsD3) {
        const color = rgb(colorScale(station));
        svg
          .append("path")
          .datum(this.dataD3.filter((d) => d.Nstation.value === station))
          .attr("fill", "none")
          .attr("transform", "translate(30,-23)")
          .attr("class", "fade-in")

          .attr("stroke", color.toString())
          .attr("stroke-width", 1)
          .attr("d", line);
      }
      this.addLegend();
    },
    addLegend() {
      const svg = d3.select("#evol_Legend");
      const legendGroup = svg
        .append("div")
        .style("fill", "white")

        .attr("class", "legend");

      const legendItems = legendGroup
        .selectAll(".legend-item")
        .data(this.stationsD3)
        .enter()
        .append("div")
        .attr("class", "legend-item");
      legendItems
        .append("svg")
        .attr("width", 10)
        .attr("height", 10)
        .append("g")
        .append("rect")
        .attr("width", 10)
        .attr("height", 10)
        .attr("fill", (d) =>
          rgb(schemeCategory10[this.stationsD3.indexOf(d)]).toString()
        );
      legendItems
        .append("text")
        .style("font-size", "0.70em")
        .attr("x", 17)
        .attr("y", 5)
        .attr("dy", "0.3em")
        .text((d) => d);
    },
    //Icones des stations
    iconTest(e) {
      const icnon = new icon({ iconUrl: img, className: e });
      return icnon;
    },
    //Icones des stations avec une flèche pour le vent
    iconTest2(e) {
      const icnon = new icon({ iconUrl: img2, className: e });
      return icnon;
    },
    //Fonctions appliqués sur chaque layer (région)
    onEachFeature(feature, layer) {
      this.colorTemp(layer);
      layer.on({
        mouseup: () => {
          this.inseeD3 = layer.feature.properties.code;

          this.d3Test(this.inseeD3, this.start.split("-")[0]);
          this.showTime = true;
          this.region = layer.feature.properties.nom;
        },
      });
    },
    //Coloration de chaque région par température
    async colorTemp(layer) {
      var finalAvg = await this.getavg;
      var arr = [];
      finalAvg.forEach((element) => {
        arr.push(Number(element[1]));
      });
      var min = Math.min(...arr);
      var max = Math.max(...arr);
      finalAvg.forEach((element) => {
        if (layer.feature.properties.code == element[0]) {
          layer.bindTooltip(
            layer.feature.properties.nom +
              " " +
              Number(element[1]).toFixed(2) +
              " °"
          );
          var hue = (1 - (element[1] - min) / (max - min)) * 40;
          layer.setStyle({
            fillColor: hsl(hue, 100, 50),
            weight: 1,
            color: "black",
            fillOpacity: 0.6,
            className: "fade-in",
          });
        }
      });
    },
    //Requête du avg des températures par région
    async AvgTemp() {
      const response = await axios.post(
        import.meta.env.VITE_API_URL,
        {
          query: getAvgTempPerRegion(this.start, this.end),
        },
        {
          headers: {
            "Content-Type": "application/x-www-form-urlencoded",
          },
          responseType: "json",
        }
      );
      var feature = regionsJson.features;
      var finalArr = [];
      response.data.results.bindings.forEach((element) => {
        feature.forEach((el) => {
          if (el.properties.code == element.insee.value) {
            finalArr.push([el.properties.code, element.temp_avg.value]);
          }
        });
      });
      this.loading = true;
      return finalArr;
    },
    //Requête du avg des températures par station

    async AvgStationTemp(query) {
      const response = await axios.post(
        import.meta.env.VITE_API_URL,
        {
          query: query(this.start, this.end),
        },
        {
          headers: {
            "Content-Type": "application/x-www-form-urlencoded",
          },
          responseType: "json",
        }
      );
      var feature = toRaw(this.stations);
      var testArr = [];
      feature.forEach((element) => {
        testArr.push(element.stationName.value);
      });
      var finalArr = response.data.results.bindings.filter((element) => {
        return testArr.includes(element.StationName.value);
      });
      return finalArr;
    },
    //Coloration de chaque station par température
    async colorStationTemp() {
      var finalAvg = await this.getstationAvg;
      var arr = [];
      finalAvg.forEach((element) => {
        arr.push(Number(element.temp_avg.value));
      });
      var min = Math.min(...arr);
      var max = Math.max(...arr);
      var finalArr = [];
      finalAvg.forEach((element) => {
        var hue = (1 - (element.temp_avg.value - min) / (max - min)) * 40;
        var hslp = hsl(hue, 100, 50);
        finalArr.push([
          element.StationName.value,
          element.temp_avg.value,
          hslp,
        ]);
      });
      var AllMark = document.getElementsByClassName("leaflet-marker-icon");
      finalArr.forEach((el) => {
        for (let i = 0; i < AllMark.length; i++) {
          if (
            this.$refs[el[0]][0].$el.className ==
            AllMark[i].className.split(" ")[1]
          ) {
            AllMark[i].style.backgroundColor = el[2];
          }
        }
      });
      for (let i = 0; i < AllMark.length; i++) {
        if (
          this.$refs["CAP CEPET"][0].$el.className ==
          AllMark[i].className.split(" ")[1]
        ) {
          AllMark[i].style.backgroundColor = "black";
        }
      }
      this.infoPopup();
    },
    //Coloration de chaque station par précipitation
    async colorStationRain() {
      var finalAvg = await this.getstationRain;
      var arr = [];
      finalAvg.forEach((element) => {
        arr.push(Number(element.rain.value));
      });
      var min = Math.min(...arr);
      var max = Math.max(...arr);
      var finalArr = [];
      finalAvg.forEach((element) => {
        var hue = (1 - (element.rain.value - min) / (max - min)) * 40 + 180;
        var hslp = hsl(hue, 100, 50);
        finalArr.push([element.StationName.value, element.rain.value, hslp]);
      });
      var AllMark = document.getElementsByClassName("leaflet-marker-icon");
      finalArr.forEach((el) => {
        for (let i = 0; i < AllMark.length; i++) {
          if (
            this.$refs[el[0]][0].$el.className ==
            AllMark[i].className.split(" ")[1]
          ) {
            AllMark[i].style.backgroundColor = el[2];
          }
        }
      });
      for (let i = 0; i < AllMark.length; i++) {
        if (
          this.$refs["CAP CEPET"][0].$el.className ==
          AllMark[i].className.split(" ")[1]
        ) {
          AllMark[i].style.backgroundColor = "black";
        }
      }
      this.infoPopup();
    },
    //Coloration de chaque station par humidité
    async colorStationHumidity() {
      var finalAvg = await this.getstationHumidity;
      var arr = [];
      finalAvg.forEach((element) => {
        arr.push(Number(element.humidity.value));
      });
      var min = Math.min(...arr);
      var max = Math.max(...arr);
      var finalArr = [];
      finalAvg.forEach((element) => {
        var hue = (1 - (element.humidity.value - min) / (max - min)) * 40 + 275;
        var hslp = hsl(hue, 100, 50);
        finalArr.push([
          element.StationName.value,
          element.humidity.value,
          hslp,
        ]);
      });
      var AllMark = document.getElementsByClassName("leaflet-marker-icon");
      finalArr.forEach((el) => {
        for (let i = 0; i < AllMark.length; i++) {
          if (
            this.$refs[el[0]][0].$el.className ==
            AllMark[i].className.split(" ")[1]
          ) {
            AllMark[i].style.backgroundColor = el[2];
          }
        }
      });
      for (let i = 0; i < AllMark.length; i++) {
        if (
          this.$refs["CAP CEPET"][0].$el.className ==
          AllMark[i].className.split(" ")[1]
        ) {
          AllMark[i].style.backgroundColor = "black";
        }
      }
      this.infoPopup();
    },
    //Coloration de chaque station par vitesse du vent et représentation de la direction du vent
    async colorStationWindSpeed() {
      var finalAvg = await this.getstationWindSpeed;
      var finalDir = await this.getstationWindDirection;
      var arr = [];
      finalAvg.forEach((element) => {
        arr.push(Number(element.speed.value));
      });
      var min = Math.min(...arr);
      var max = Math.max(...arr);
      var finalArr = [];
      finalAvg.forEach((element) => {
        var hue = (1 - (element.speed.value - min) / (max - min)) * 40 + 120;
        var hslp = hsl(hue, 100, 50);
        finalDir.forEach((em) => {
          if (em.StationName.value == element.StationName.value) {
            finalArr.push([
              element.StationName.value,
              element.speed.value,
              hslp,
              em.angle.value,
            ]);
          }
        });
      });
      var AllMark = document.getElementsByClassName("leaflet-marker-icon");
      finalArr.forEach((el) => {
        for (let i = 0; i < AllMark.length; i++) {
          if (
            this.$refs[el[0]][0].$el.className ==
            AllMark[i].className.split(" ")[1]
          ) {
            AllMark[i].style.backgroundColor = el[2];
            AllMark[i].style.transform +=
              "rotate(" + Number(el[3]).toFixed(0) + "deg)";
          }
        }
      });
      this.infoPopup();
    },
    //Selection des stations pour plus de manipulation dans la partie "Meteorological Parameters"
    selectStation(marker) {
      this.$store.dispatch("updateSelectedStations", marker.stationName.value);
    },
    //Pour enlever la coloration des boutons du ui des stations
    unsetEvery() {
      this.showTemp = false;
      this.showRain = false;
      this.showHumidity = false;
      this.showWind = false;
    },
    //Pour enlever l'affichage des stations
    unsetStation() {
      this.show = true;
      this.showMark = false;
      this.showTime = false;
      this.unsetEvery();
    },
    //Pour enlever l'affichage des régions
    unsetRegion() {
      this.show = false;
      this.showMark = true;
      this.showTime = false;
    },
    //Affichage de la valeur contenu dans chaque station dépendant de la mesure sélectionnée
    async infoPopup() {
      var mesure;
      if (this.showTemp) {
        var station = await this.getstationAvg;
        mesure = " °";
        station.forEach((element) => {
          if (
            this.$refs[
              "marker" + element.StationName.value
            ][0].$el.innerHTML.includes(element.StationName.value)
          ) {
            this.$refs["marker" + element.StationName.value][0].$el.innerHTML =
              element.StationName.value +
              " " +
              Number(element.temp_avg.value).toFixed(2) +
              mesure;
          }
        });
      } else if (this.showHumidity) {
        var station = await this.getstationHumidity;
        mesure = " g/m3";
        station.forEach((element) => {
          if (
            this.$refs[
              "marker" + element.StationName.value
            ][0].$el.innerHTML.includes(element.StationName.value)
          ) {
            this.$refs["marker" + element.StationName.value][0].$el.innerHTML =
              element.StationName.value +
              " " +
              Number(element.humidity.value).toFixed(2) +
              mesure;
          }
        });
        this.$refs["marker" + "CAP CEPET"][0].$el.innerHTML = "CAP CEPET";
      } else if (this.showRain) {
        var station = await this.getstationRain;
        mesure = " mm";
        station.forEach((element) => {
          if (
            this.$refs[
              "marker" + element.StationName.value
            ][0].$el.innerHTML.includes(element.StationName.value)
          ) {
            this.$refs["marker" + element.StationName.value][0].$el.innerHTML =
              element.StationName.value +
              " " +
              Number(element.rain.value).toFixed(2) +
              mesure;
          }
        });
        this.$refs["marker" + "CAP CEPET"][0].$el.innerHTML = "CAP CEPET";
      } else {
        var station = await this.getstationWindSpeed;
        mesure = " m/s";
        station.forEach((element) => {
          if (
            this.$refs[
              "marker" + element.StationName.value
            ][0].$el.innerHTML.includes(element.StationName.value)
          ) {
            this.$refs["marker" + element.StationName.value][0].$el.innerHTML =
              element.StationName.value +
              " " +
              Number(element.speed.value).toFixed(2) +
              mesure;
          }
        });
      }
    },
    //Changer l'année et relancer les requêtes
    async changeYear(year) {
      this.start = year + "-01-01";
      this.end = year + "-12-31";
      this.getavg = this.AvgTemp();
      this.getstationAvg = this.AvgStationTemp(
        buildQuery_getAllStationsAvgTemp
      );
      this.getstationRain = this.AvgStationTemp(
        buildQuery_avgRainQtyPerStation
      );
      this.getstationWindSpeed = this.AvgStationTemp(
        buildQuery_getAllStationsAvgWindSpeed
      );
      this.getstationWindDirection = this.AvgStationTemp(
        buildQuery_getAllStationsAvgWindDirection
      );
      this.getstationHumidity = this.AvgStationTemp(
        buildQuery_getAllStationsAvgHumidity
      );
      this.reload = false;
      this.showTime = false;
      this.showRain = false;
      this.showHumidity = false;
      this.showWind = false;
      this.loading = false;

      setTimeout(() => {
        this.reload = true;
      }, 100);
    },

    //Changer l'année et relancer les requêtes dans le timebrush
    changeYearD3(year) {
      this.yearSlide = year;
      this.d3Test(this.inseeD3, year);
    },
    test(e) {
      var arr = [
        new Date(
          new Date(String(this.yearSlide) + "-01-01").getTime() +
            e[0] * 24 * 60 * 60 * 1000
        )
          .toISOString()
          .slice(0, 10),
        new Date(
          new Date(String(this.yearSlide) + "-01-01").getTime() +
            e[1] * 24 * 60 * 60 * 1000
        )
          .toISOString()
          .slice(0, 10),
      ];
      this.handleRangeChangedEvent(arr);
    },
  },
};
</script>
<template>
  <!-- UI Buttons -->
  <div class="choose">
    <div
      @click="
        () => {
          unsetStation();
        }
      "
      :class="{ orange: show }"
      class="btn_div"
    >
      Région
    </div>
    <div @click="unsetRegion()" :class="{ orange: showMark }" class="btn_div">
      Station
    </div>
    <div class="metrics fade-in" v-if="this.showMark">
      <span
        @click="
          () => {
            unsetEvery();
            this.showTemp = true;
            this.colorStationTemp();
          }
        "
        :class="{ orange: this.showTemp }"
        ><i class="fa-solid fa-temperature-high"></i
      ></span>
      <span
        @click="
          () => {
            unsetEvery();
            this.showRain = true;
            this.colorStationRain();
          }
        "
        :class="{ blue: this.showRain }"
        ><i class="fa-solid fa-cloud-rain"></i
      ></span>
      <span
        @click="
          () => {
            unsetEvery();
            this.showHumidity = true;
            this.colorStationHumidity();
          }
        "
        :class="{ purple: this.showHumidity }"
        ><i class="fa-solid fa-droplet"></i
      ></span>
      <span
        @click="
          () => {
            unsetEvery();
            this.showWind = true;
            this.colorStationWindSpeed();
          }
        "
        :class="{ green: this.showWind }"
        ><i class="fa-solid fa-wind"></i
      ></span>
    </div>
    <v-select
      label="Year"
      v-model="defaultYear"
      :items="[
        2022, 2021, 2020, 2019, 2018, 2017, 2016, 2015, 2014, 2013, 2012,
      ]"
      @update:modelValue="changeYear"
      class="selectosStation fade-in"
    ></v-select>
  </div>

  <!---->

  <!-- Map -->
  <div class="blackDrop" v-if="showTime" @click="unsetStation()"></div>
  <div class="map">
    <l-map
      ref="map"
      v-model:zoom="zoom"
      :center="[46.41322, 2]"
      :min-zoom="4"
      v-if="reload"
      class="fade-in"
    >
      <l-tile-layer
        url="https://{s}.tile.openstreetmap.org/{z}/{x}/{y}.png"
        layer-type="base"
        name="OpenStreetMap"
      ></l-tile-layer>
      <l-geo-json
        v-if="show && loading"
        :geojson="geoJson"
        :options="geoJsonOptions"
        :optionsStyle="() => geoJsonStyle"
      >
      </l-geo-json>
      <l-marker
        v-if="showMark"
        v-for="(marker, i) in stations"
        :lat-lng="[
          parseFloat(marker.lat.value) * 1.005,
          parseFloat(marker.long.value),
        ]"
        :name="marker.name"
        :draggable="false"
        :ref="marker.stationName.value"
        @click="
          () => {
            this.selectStation(marker);
          }
        "
        :class="'marker' + i"
        :icon="
          this.showWind
            ? this.iconTest('marker' + i + ' MarkerTest fade-in')
            : this.iconTest2('marker' + i + ' MarkerTest fade-in')
        "
      >
        <l-tooltip :ref="'marker' + marker.stationName.value">{{
          marker.stationName.value
        }}</l-tooltip>
      </l-marker>
    </l-map>
  </div>
  <!---->

  <!-- TimeBrush -->

  <div class="TimeBrush scale-up-center" v-if="showTime">
    <div class="Time-select">
      <h2>Time Brush</h2>
      <v-select
        label="Year"
        v-model="defaultYear"
        :items="[
          2022, 2021, 2020, 2019, 2018, 2017, 2016, 2015, 2014, 2013, 2012,
        ]"
        class="selectos"
        @update:modelValue="changeYearD3"
      ></v-select>
    </div>
    <v-range-slider
      class="slidos"
      :min="0"
      :max="360"
      :step="10"
      @update:modelValue="test"
      thumb-color="orange"
      thumb-label
      strict
    >
      <template v-slot:thumb-label="{ modelValue }">
        {{
          new Date(
            new Date(String(this.yearSlide) + "-01-01").getTime() +
              modelValue * 24 * 60 * 60 * 1000
          )
            .toISOString()
            .slice(0, 10)
        }}
      </template>
    </v-range-slider>
    <div
      style="position: relative; height: 223px; width: 100%"
      id="evol_parent"
    >
      <svg id="evol"></svg>
      <div id="evol_Legend"></div>
    </div>

    <h4>{{ this.region }}</h4>
    <div></div>
  </div>
  <!---->
</template>
<style>
.fade-in {
  -webkit-animation: fade-in 1.2s cubic-bezier(0.39, 0.575, 0.565, 1) both;
  animation: fade-in 1.2s cubic-bezier(0.39, 0.575, 0.565, 1) both;
}
.scale-up-center {
  -webkit-animation: scale-up-center 0.4s cubic-bezier(0.39, 0.575, 0.565, 1)
    both;
  animation: scale-up-center 0.4s cubic-bezier(0.39, 0.575, 0.565, 1) both;
}
@-webkit-keyframes fade-in {
  0% {
    opacity: 0;
  }
  100% {
    opacity: 1;
  }
}
@keyframes fade-in {
  0% {
    opacity: 0;
  }
  100% {
    opacity: 1;
  }
}

@-webkit-keyframes scale-up-center {
  0% {
    -webkit-transform: scale(0.5);
    transform: scale(0.5);
  }
  100% {
    -webkit-transform: scale(1);
    transform: scale(1);
  }
}
@keyframes scale-up-center {
  0% {
    -webkit-transform: scale(0.5);
    transform: scale(0.5);
  }
  100% {
    -webkit-transform: scale(1);
    transform: scale(1);
  }
}

.leaflet-zoom-animated path {
  -webkit-animation: fade-in 1.2s cubic-bezier(0.39, 0.575, 0.565, 1) both;
  animation: fade-in 1.2s cubic-bezier(0.39, 0.575, 0.565, 1) both;
}
.map {
  height: calc(100% + 50px);

  width: auto;
  z-index: 9;
  position: relative;
}
.MarkerTest {
  transform-origin: center;
  width: 28px !important;
  background-color: black;
  border-radius: 10px;
  transition: 0.5s;
}

.leaflet-tile-pane {
  filter: brightness(80%);
}
.choose {
  display: flex;
  position: absolute;
  left: 4%;
  top: 13%;
  z-index: 10;
  gap: 10px;
  flex-wrap: wrap;
  width: 20%;
  justify-content: center;
}
.TimeBrush {
  display: flex;
  flex-wrap: wrap;
  background: rgba(0, 0, 0, 0.705);
  width: 90%;
  position: absolute;
  height: 73%;
  left: 5%;
  top: 35%;
  z-index: 12;
  gap: 10px;
  color: white;
  justify-content: center;
  border-radius: 10px;
  font-size: 1.5em;
  padding-bottom: 20px;
  -webkit-backdrop-filter: blur(5px);
  backdrop-filter: blur(5px);
}
.TimeBrush * {
  text-align: center;
}
.Time-select {
  width: 60%;
  display: flex;
  gap: 20px;
  align-items: center;
}
.TimeBrush h2 {
  padding-top: 20px;
  text-align: left;
}
.slidos {
  padding-inline: 20px;
  width: 80%;
}
.selectos {
  height: 40%;
}
.selectosStation {
  position: relative;
  width: 500px !important;
  height: 55px;
  background-color: white;
}
.TimeBrush h4 {
  width: 50%;
}
.choose {
  z-index: 13;
}
.choose .btn_div {
  line-height: 36px;
  padding: 0 20px;
  color: #fff;
  border-radius: 30px;
  cursor: pointer;
  border: transparent;
  background-color: #0003;
  font-size: 14px;
  width: -moz-fit-content;
  width: fit-content;
  height: -moz-fit-content;
  height: fit-content;
  transition: 0.3s;
}
.orange {
  background-color: #fd8d3c !important;
  box-shadow: 0px 2px 2px black;
}
.green {
  background-color: #3cfd4c !important;
  box-shadow: 0px 2px 2px black;
}
.purple {
  background-color: #cd3cfd !important;
  box-shadow: 0px 2px 2px black;
}
.blue {
  background-color: #3c6cfd !important;
  box-shadow: 0px 2px 2px black;
}
.metrics {
  border: solid;
  display: flex;
  flex-direction: column;
  position: relative;

  width: fit-content;
  height: -moz-fit-content;
  height: fit-content;
  border-radius: 30px;
  background: rgba(0, 0, 0, 0.75);
  padding: 10px;
}
.metrics span {
  background: rgba(0, 0, 0, 0.3);
  border: none;
  color: #fff;
  cursor: pointer;
  border-radius: 20px;
  padding: 10px;
  padding-inline: 13px;
  margin-bottom: 10px;
  align-self: center;
}
.legend {
  -webkit-animation: fade-in 1.2s cubic-bezier(0.39, 0.575, 0.565, 1) both;
  animation: fade-in 1.2s cubic-bezier(0.39, 0.575, 0.565, 1) both;
}
.legend-item {
  font-size: 14px;
  text-align: left;
  display: flex;
  column-gap: 10px;
}
#evol_Legend {
  position: relative;
  left: 85%;
  top: -110%;
}

.blackDrop {
  z-index: 11;
  background: rgba(0, 0, 0, 0.444);

  width: 110%;
  position: absolute;
  height: 120%;
  -webkit-backdrop-filter: blur(1px);
  backdrop-filter: blur(1px);
  -webkit-animation: fade-in 0.4s cubic-bezier(0.39, 0.575, 0.565, 1) both;
  animation: fade-in 0.4s cubic-bezier(0.39, 0.575, 0.565, 1) both;
}
.v-slider-thumb__label {
  padding: 20px;
}
.v-slider-thumb__label > div {
  width: 120px;
  padding: 20px;
}
</style>
