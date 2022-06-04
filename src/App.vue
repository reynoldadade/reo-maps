<script setup>
import { watch, ref, computed, onMounted } from "vue";
import mapboxgl from "mapbox-gl";
import getDistance from "@turf/distance";

import {
  uniqueNamesGenerator,
  adjectives,
  animals,
} from "unique-names-generator";
import { polygon } from "@turf/helpers";

const map = ref(null);
const coordinates = ref([]);
const maxDistance = ref(0);
const groupColors = ref([]);

function createCoordinates(coords) {
  const { lng, lat } = coords;
  const el = document.createElement("div");
  el.className = "marker";

  const marker = new mapboxgl.Marker(el).setLngLat([lng, lat]).addTo(map.value);
  coordinates.value.push({
    lng,
    lat,
    name: uniqueNamesGenerator({
      dictionaries: [adjectives, animals],
      separator: " ",
    }),
    marker,
  });

  calculateMaxDistance(coords);
}

// function threeDp(item) {
//   //3 decimal places
//   return item.value.toFixed(3);
// }

function calculateMaxDistance(point) {
  //calculate distance between coord and each existing point for max distance
  const { lng, lat } = point;
  let max = 0;
  coordinates.value.forEach((coord) => {
    const distance = getDistance([lng, lat], [coord.lng, coord.lat]);

    if (distance > max) {
      max = distance;
    }
  });
  maxDistance.value = max * 0.25;
}

function randomColor() {
  return "#" + Math.floor(Math.random() * 16777215).toString(16);
}

function generateColors(group) {
  if (groupColors.value.length > group.length) {
    groupColors.value.splice(
      group.length,
      groupColors.value.length - group.length
    );
  } else {
    const diff = group.length - groupColors.value.length;
    for (let i = 0; i < diff; i++) {
      groupColors.value.push(randomColor());
    }
  }
}

function changeColor(index) {
  const randomColor = randomColor();
  if (groupColors.value.includes(randomColor)) {
    return changeColor(index);
  }
  groupColors.value[index] = randomColor;
}

function generateMapLines(groupCordinates) {
  const mapLines = [];
  groupCordinates.forEach((coords) => {
    const { group } = coords;
    mapLines.push([
      ...group.map((coord) => [coord.lng, coord.lat]),
      [group[0].lng, group[0].lat],
    ]);
  });

  return mapLines
    .filter((item) => item.length > 3)
    .map((item) => polygon([item]));
}

function drawLines(groupCordinates) {
  const mapLines = generateMapLines(groupCordinates);

  const source = {
    type: "FeatureCollection",
    features: mapLines,
  };

  if (map.value.getSource("lines")) {
    return map.value.getSource("lines").setData(source);
  }
  map.value.addSource("lines", {
    type: "geojson",
    data: source,
  });
  map.value.addLayer({
    id: "lines",
    type: "line",
    source: "lines",
    paint: {
      "line-width": 2,
      "line-color": "#000",
    },
  });
}

function createSourceAndLayer(mapLine){

},

const groupCordinates = computed(() => {
  let coordinateCopy = [...coordinates.value];
  if (coordinateCopy.length === 0) {
    return [];
  }
  if ((coordinateCopy.length > 0) & (coordinateCopy.length < 9)) {
    return [
      {
        name: "",
        group: coordinateCopy,
      },
    ];
  }
  let groups = [];

  // loop through the coordinates and group them by distance
  for (let i = 0; i < coordinateCopy.length; i++) {
    const group = [];
    for (let j = 0; j < coordinateCopy.length; j++) {
      const distance = getDistance(
        [coordinateCopy[i].lng, coordinateCopy[i].lat],
        [coordinateCopy[j].lng, coordinateCopy[j].lat]
      );
      // if less than max distance then add to a group
      if (distance < maxDistance.value) {
        group.push(coordinateCopy[j]);
      }
    }
    // if not empty add to groups
    if (group.length > 0) {
      // issue with what i have now is that there will be duplicates
      group.sort((a, b) => a.name - b.name);
      const groupObject = {
        name: group.map((item) => item.name).toString(),
        group,
      };
      //before pushing check if array already exists in groups
      const exists = groups.find((item) => item.name === groupObject.name);
      if (!exists) {
        groups.push(groupObject);
      }
    }
  }

  return groups;
});

function threeDp(value) {
  return value.toFixed(3);
}

onMounted(() => {
  mapboxgl.accessToken =
    "pk.eyJ1Ijoid2FsdWxlbGFkbWluIiwiYSI6ImNrMm5samZ3bzBya2QzY29maDh6czJ6bGcifQ.LRMREa0M12e8Cjdk7TCS6w";

  map.value = new mapboxgl.Map({
    container: "map",
    style: "mapbox://styles/mapbox/streets-v11",
    center: [-96, 37.8],
    zoom: 9,
  });

  map.value.on("click", (e) => {
    const coords = e.lngLat;
    createCoordinates(coords);
  });
});

//watchers

watch(
  () => groupCordinates.value,
  (newCoordinates) => {
    // if coordinate become more than 9 generate group colors and draw lines
    if (newCoordinates.length > 1) {
      generateColors(newCoordinates);
      drawLines(newCoordinates);
    }
  }
);
</script>

<script>
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
};
</script>

<template>
  <div class="rounded-[32px] mapCard p-6 flex justify-between">
    <div class="w-[210px] flex flex-col justify-between">
      <div class="flex-grow flex flex-col">
        <div class="mb-[14px] text-[20px] leading-[24px]">Groups</div>
        <div class="h-[579px] overflow-y-auto">
          <div
            v-for="(coord, index) in groupCordinates"
            :key="index + coord.name"
            class="mb-4"
          >
            <button class="flex items-center gap-1" @click="changeColor(index)">
              <div
                class="h-3 w-3 rounded-[2px]"
                :style="`background-color: ${groupColors[index]}`"
                v-if="coord.name"
              ></div>
              <div class="flex h-full items-center pt-1">
                {{ coord.name ? "Group" : "Ungrouped" }}
              </div>
            </button>
            <div v-for="group in coord.group" :key="group.name">
              <span>
                {{ group.name }}
              </span>
              <span class="text-[#767676] pl-2"
                >{{ threeDp(group.lng) }} {{ threeDp(group.lat) }}</span
              >
            </div>
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
