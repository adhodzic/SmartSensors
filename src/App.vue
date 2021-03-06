<template>
  <div id="app">
    <line-chart
      v-if="loaded"
      :chartData="chartData"
      :options="options"
      :change="change"
    />
    <sensor
      class="sensor-list"
      v-bind:key="sensorData.unix"
      v-for="sensorData in sensors"
      :sensorData="sensorData"
    />

    <h2>Profit Euro: {{ ether }} EUR</h2>
    <h2>Profit Kuna:{{ ether * 7.55 }} HRK</h2>
  </div>
</template>

<script>
import Sensor from "./components/Sensor";
import LineChart from "./components/Chart";
import axios from "axios";
import moment from "moment";
export default {
  name: "App",
  components: { LineChart, Sensor },
  data: () => ({
    ether: null,
    loaded: false,
    change: false,
    sensors: [],
    chartData: {
      labels: ["January", "February"],
      datasets: [
        {
          label: "Data One",
          backgroundColor: "#f87979",
          data: [40, 20],
        },
      ],
    },
    options: {
      responsive: true,
      maintainAspectRatio: false,
      scales: {
        yAxes: [
          {
            display: true,
            ticks: {
              suggestedMax: 25,
              suggestedMin: 10, // minimum will be 0, unless there is a lower value.
              // OR //
              //beginAtZero: true   // minimum value will be 0.
            },
          },
        ],
      },
    },
  }),
  methods: {
    avg(v) {
      return v.reduce((a, b) => a + b, 0) / v.length;
    },
    smoothOut(vector, variance) {
      let self = this;
      var t_avg = self.avg(vector) * variance;
      var ret = Array(vector.length);
      for (var i = 0; i < vector.length; i++) {
        (function () {
          var prev = i > 0 ? ret[i - 1] : vector[i];
          var next = i < vector.length ? vector[i] : vector[i - 1];
          ret[i] = self.avg([t_avg, self.avg([prev, vector[i], next])]);
        })();
      }
      return ret;
    },
    async getData() {
      try {
        const temp = await axios.get("http://raspberrypitemp.ddns.net:8080/api/daily");
        let data = this.transformData(temp.data);
        this.chartData.labels = data[0];
        this.chartData.datasets[0].data = data[1];
        this.loaded = true;
      } catch (e) {
        console.error(e);
      }
    },
    async getSensors() {
      const response = await axios.get("http://raspberrypitemp.ddns.net:8080/api/temp");
      let sensors = response.data;
      console.log(sensors);
      this.sensors = [];
      this.sensors.push(sensors);
    },
    transformData(data) {
      let lab = [];
      let dat = [];
      data.forEach((e) => {
        lab.push(moment(e.unix * 1000).format("DD-MM-YY[\n] HH:mm"));
        dat.push(e.temp);
      });

      return [lab, dat];
    },
    async getEther() {
      const price = await axios.get(
        "https://min-api.cryptocompare.com/data/price?fsym=ETH&tsyms=EUR"
      );
      let eur = price.data["EUR"];
      eur = eur * 0.21388604;
      this.ether = eur - 235;
      console.log(eur);
    },
  },
  async mounted() {
    this.loaded = false;
    await this.getEther();
    await this.getData();
    await this.getSensors();
    let self = this;
    setInterval(async function () {
      await self.getEther();
      await self.getData();
      await self.getSensors();
      self.change = !self.change;
    }, 3000);
  },
};
</script>

<style>
#app {
  font-family: Avenir, Helvetica, Arial, sans-serif;
  -webkit-font-smoothing: antialiased;
  -moz-osx-font-smoothing: grayscale;
  text-align: center;
  color: #2c3e50;
  margin-top: 60px;
}
</style>
