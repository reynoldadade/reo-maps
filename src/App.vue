<script>
import mapboxgl from "mapbox-gl";
import distance from "@turf/distance";
import {
  uniqueNamesGenerator,
  adjectives,
  animals,
} from "unique-names-generator";
export default {
  head() {
    return {
      link: [
        {
          rel: "stylesheet",
          href: "https://api.tiles.mapbox.com/mapbox-gl-js/v2.8.2/mapbox-gl.css",
          type: "text/css",
        },
      ],
    };
  },

  data() {
    return {
      map: null,
      coordinates: [],
      maxDistance: 0,
    };
  },
  mounted() {
    mapboxgl.accessToken =
      "pk.eyJ1Ijoid2FsdWxlbGFkbWluIiwiYSI6ImNrMm5samZ3bzBya2QzY29maDh6czJ6bGcifQ.LRMREa0M12e8Cjdk7TCS6w";
    this.map = new mapboxgl.Map({
      container: "map", // container ID
      style: "mapbox://styles/mapbox/streets-v11", // style URL
      center: [-74.5, 40], // starting position [lng, lat]
      zoom: 10, // starting zoom
    });

    this.map.on("click", (e) => {
      this.createCoordinates(e.lngLat);
    });
  },
  computed: {
    methods: {
      createCoordinates(coords) {
        const { lng, lat } = coords;
        const el = document.createElement("div");
        el.className = "marker";

        const marker = new mapboxgl.Marker(el)
          .setLngLat([lng, lat])
          .addTo(this.map);
        this.coordinates.push({
          lng,
          lat,
          name: uniqueNamesGenerator({
            dictionaries: [adjectives, animals],
            separator: " ",
          }),
          marker,
        });
        this.calculateMaxDistance(coords);
      },
      threeDp(value) {
        //3 decimal places
        return value.toFixed(3);
      },
      calculateMaxDistance(point) {
        //calculate distance between coord and each existing point for max distance
        const { lng, lat } = point;
        let maxDistance = 0;
        this.coordinates.forEach((coord) => {
          const distance = distance([lng, lat], [coord.lng, coord.lat]);

          if (distance > maxDistance) {
            maxDistance = distance;
          }
        });
        this.maxDistance = maxDistance * 0.25;
      },
    },
  },
};
</script>

<template>
  <div class="rounded-[32px] mapCard p-6 flex justify-between">
    <div class="w-[210px] flex flex-col justify-between">
      <div class=" ">
        <div class="mb-[14px] text-[20px] leading-[24px]">Groups</div>
        <div>
          <div v-for="(coord, index) in coordinates" :key="index">
            {{ coord.name }}
            <span class="text-[#767676]"
              >{{ threeDp(coord.lng) }} {{ threeDp(coord.lat) }}</span
            >
          </div>
        </div>
      </div>

      <div>
        <button
          class="px-4 py-[6.5px] rounded text-[#2265F1]"
          id="clearPointsButton"
        >
          Clear points
        </button>
      </div>
    </div>
    <div
      class="w-[738px] h-[673px] rounded-[16px] overflow-hidden"
      id="map"
    ></div>
  </div>
</template>

<style>
.mapCard {
  background: #ffffff;
  position: absolute;
  left: 200px;
  top: 152px;
  /* Light/Elevation 16 */

  box-shadow: 0px 8px 16px rgba(0, 0, 0, 0.08),
    0px 16px 48px rgba(0, 0, 0, 0.12);
  width: 1040px;
  height: 721px;
}

#clearPointsButton {
  background: linear-gradient(
      0deg,
      rgba(34, 101, 241, 0.08),
      rgba(34, 101, 241, 0.08)
    ),
    #ffffff;
}

.marker {
  box-sizing: border-box;
  background: #ffffff;
  border-radius: 50%;
  border: 4px solid #767676;
  box-shadow: 0px 2px 4px rgba(0, 0, 0, 0.4);
  cursor: pointer;
  width: 12px;
  height: 12px;
}
</style>
