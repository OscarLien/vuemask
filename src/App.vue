<template>
  <div id="app">
    <div class="sidebar" :class="{ hide: sidebar === false }">
      <div class="sideHeader">
        <div class="todayInfo">
          <h2>
            星期<span>{{ dayList[today] }}</span>
          </h2>
          <div class="buyInfo">
            <p>{{ date }}</p>
            <p>
              身分證末尾
              <span class="idColor">{{ buyDay }}</span>
              可購買
            </p>
          </div>
        </div>
        <div class="selectArea">
          <label for="">
            <select
              name=""
              id=""
              class="citySelect"
              v-model="select.city"
              @change="select.area = ''"
            >
              <option value="">--請選擇縣市--</option>
              <option
                :value="city.CityName"
                v-for="city in cityName"
                :key="city.CityName"
              >
                {{ city.CityName }}
              </option>
            </select>
          </label>
          <label for="">
            <select
              name=""
              id=""
              class="citySelect"
              v-model="select.area"
              @change="updateSelect"
            >
              <option value="">--請選擇行政區--</option>
              <option
                :value="town.AreaName"
                v-for="town in cityName.find(
                  (city) => city.CityName === select.city
                ).AreaList"
                :key="town.AreaName"
              >
                {{ town.AreaName }}
              </option>
            </select>
          </label>
        </div>
      </div>
      <div class="sideSwitch-sm d-md-none"  @click="sidebar = !sidebar">
        <i class="fas fa-chevron-left" v-if="sidebar === true"></i>
      </div>
      <div class="sideSwitch"   @click="sidebar = !sidebar">
        <i class="fas fa-chevron-left" v-if="sidebar === true"></i>
        <i class="fas fa-chevron-right" v-else></i>
      </div>
      <div class="sideBody">
          <template v-for="item in filterData">
            <div
              class="sideContentBox"
              :key="item.properties.id"
              @click="penTo(item)"
            >
              <p class="pharmacyName">{{ item.properties.name }}</p>
              <div class="pharmacyInfo">
                <p>{{ item.properties.address }}</p>
                <p>{{ item.properties.phone }}</p>
                <p>{{ item.properties.note }}</p>
              </div>
              <div class="maskLeft">
                <div class="countbox adult">
                  <span>成人口罩</span>
                  <span>{{ item.properties.mask_adult }}</span>
                </div>
                <div class="countbox child">
                  <span>兒童口罩</span>
                  <span>{{ item.properties.mask_child }}</span>
                </div>
              </div>
            </div>
          </template>
      </div>
    </div>
    <div id="map">
      <div class="sideSwitch-sm-back" @click="sidebar = true">
        <i class="fas fa-chevron-right"></i>
      </div>
    </div>
  </div>
</template>
<script>
import L from 'leaflet';
import cityName from './assets/cityName.json';

let osmMap = {};

const osm = {
  addMapMarker(x, y, item) {
    L.marker([y, x]).addTo(osmMap).bindPopup(`
    <div class="popOutBox">
    <div class="pharmacyName">${item.name}</div>
    <div class="pharmacyInfo">
    <span>${item.address}</span><br>
    <span>${item.phone}</span><br>
    <span>${item.note}</span><br>
    </div>
    <div class="maskLeft">
    <div class="countbox adult">
    <span>成人口罩</span> <span>${item.mask_adult}</span>
    </div>
    <div class="countbox child">
    <span>兒童口罩</span> <span>${item.mask_child}</span>
    </div>
    </div>
    </div>`);
  },
  removeMapMarker() {
    osmMap.eachLayer((layer) => {
      if (layer instanceof L.Marker) {
        osmMap.removeLayer(layer);
      }
    });
  },
  penTo(x, y, item) {
    osmMap.panTo(new L.LatLng(y, x));
    L.marker([y, x], {})
      .addTo(osmMap)
      .bindPopup(
        `
        <div class="popOutBox">
        <div class="pharmacyName">${item.name}</div>
        <div class="pharmacyInfo">
        <span>${item.address}</span><br>
        <span>${item.phone}</span><br>
        <span>${item.note}</span><br>
        </div>
        <div class="maskLeft">
        <div class="countbox adult">
        <span>成人口罩</span> <span>${item.mask_adult}</span>
        </div>
        <div class="countbox child">
        <span>兒童口罩</span> <span>${item.mask_child}</span>
        </div>
        </div>
        </div>
        `,
      )
      .openPopup();
  },
};

export default {
  name: 'App',
  data: () => ({
    data: [],
    osmMap: {},
    cityName,
    select: {
      city: '臺北市',
      area: '中正區',
    },
    dayList: ['日', '一', '二', '三', '四', '五', '六'],
    sidebar: true,
  }),
  mounted() {
    const vm = this;
    const url = 'https://raw.githubusercontent.com/kiang/pharmacies/master/json/points.json';
    vm.$http.get(url).then((response) => {
      vm.data = response.data.features;
      vm.updateMap();
    });
    osmMap = L.map('map', {
      center: [23.03, 121.55],
      zoom: 15,
    });
    L.tileLayer('https://{s}.tile.openstreetmap.org/{z}/{x}/{y}.png?{foo}', {
      foo: 'bar',
      attribution:
        '&copy; <a href="https://www.openstreetmap.org/copyright">OpenStreetMap</a> contributors',
      maxZoom: 18,
    }).addTo(osmMap);
  },
  methods: {
    updateMap() {
      const pharmacies = this.filterData;
      pharmacies.forEach((pharmacy) => {
        const { properties, geometry } = pharmacy;
        osm.addMapMarker(
          geometry.coordinates[0],
          geometry.coordinates[1],
          properties,
        );
      });
      this.penTo(pharmacies[0]);
    },
    removeMarker() {
      osm.removeMapMarker();
    },
    penTo(item) {
      const { properties, geometry } = item;
      osm.penTo(geometry.coordinates[0], geometry.coordinates[1], properties);
    },
    updateSelect() {
      this.removeMarker();
      this.updateMap();
      const pharmacy = this.data.find(
        (item) => item.properties.town === this.select.area,
      );
      const { geometry, properties } = pharmacy;
      osm.penTo(geometry.coordinates[0], geometry.coordinates[1], properties);
    },
  },
  computed: {
    today() {
      const today = new Date().getDay();
      return today;
    },
    date() {
      const date = new Date();
      const year = date.getFullYear();
      const month = date.getMonth() + 1;
      const day = date.getDate();
      const today = `${year}-${month}-${day}`;
      return today;
    },
    buyDay() {
      const today = new Date().getDay();
      if (today === 0) {
        return '都可以購買';
      }
      return today % 2 === 0 ? '2,4,6,8,0' : '1,3,5,7,9';
    },
    filterData() {
      const vm = this;
      const data = vm.data.filter((item) => (item.properties.county === vm.select.city
      && item.properties.town === vm.select.area));
      return data;
    },
  },
};
</script>
<style lang="scss">
@import "./assets/all";
</style>
