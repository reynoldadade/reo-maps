<script setup>
import { watch, ref, computed, onMounted } from "vue";
import mapboxgl from "mapbox-gl";
import getDistance from "@turf/distance";
import randomColor from "randomcolor";

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
const Layers = ref([]);

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

// calculate farthest distanct point

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

// generate a random color
// function randomColor() {
//   return "#" + Math.floor(Math.random() * 16777215).toString(16);
// }

// generate colors according to number of groups

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

// click to change the color of a group
function changeColor(index) {
  const color = randomColor();
  if (groupColors.value.includes(color)) {
    return changeColor(index);
  }
  groupColors.value[index] = color;

  //change fill color
  map.value.setPaintProperty(
    `${Layers.value[index].name}-fill`,
    "fill-color",
    color
  );

  //change line color
  map.value.setPaintProperty(
    `${Layers.value[index].name}-line`,
    "line-color",
    color
  );
}

// form array to be used for mapbox layers

function generateMapLines(groupCordinates) {
  const mapLines = [];
  groupCordinates.forEach((coords) => {
    const { group, name } = coords;
    // you append first coord to the other coords to close the polygon
    const polygonCoords = [
      ...group.map((coord) => [coord.lng, coord.lat]),
      [group[0].lng, group[0].lat],
    ];
    mapLines.push({ polygonCoords, name });
  });

  return (
    mapLines
      // we filter becuase you can form any shape without at least 4 points
      .filter((item) => item.polygonCoords.length > 3)
      .map((item) => ({
        name: item.name,
        polygonCoords: polygon([item.polygonCoords]),
      }))
  );
}

function drawLines(groupCordinates) {
  const mapLines = generateMapLines(groupCordinates);
  // loop though each group and draw the lines

  mapLines.forEach((line, index) => {
    // get line data
    const { polygonCoords, name } = line;
    // add polygoncoords as source
    const source = polygonCoords;
    // check if source exists then update data
    if (map.value.getSource(name)) {
      return map.value.getSource(name).setData(source);
    }
    // add source to map
    map.value.addSource(name, {
      type: "geojson",
      data: source,
    });

    //create layer
    const linelayer = {
      id: `${name}-line`,
      type: "line",
      source: name,
      paint: {
        "line-width": 2,
        "line-color": groupColors.value[index],
      },
    };

    const fillLayer = {
      id: `${name}-fill`,
      type: "fill",
      source: name,
      paint: {
        "fill-color": groupColors.value[index],
        "fill-opacity": 0.4,
      },
    };
    // add line layer
    map.value.addLayer(linelayer);

    //add fill layer to map
    map.value.addLayer(fillLayer);
    //push details into layers so you can track
    Layers.value.push({ name, linelayer, fillLayer });
  });
}

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

function clearPoint() {
  // clear markers
  coordinates.value.forEach((item) => {
    item.marker.remove();
  });
  //clear coordinates
  coordinates.value = [];

  //remove source
  Layers.value.forEach((layer) => {
    map.value.removeLayer(`${layer.name}-line`);
    map.value.removeLayer(`${layer.name}-fill`);
    map.value.removeSource(layer.name);
  });
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
            <button
              class="flex items-center gap-1 hover:bg-gray-200 p-1 rounded"
              @click="changeColor(index)"
            >
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
          @click="clearPoint"
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
