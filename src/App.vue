<script setup>
import { ref, onMounted, onUnmounted, nextTick } from "vue";
import L from "leaflet";
import "leaflet/dist/leaflet.css";
import "leaflet-routing-machine";
import "leaflet-routing-machine/dist/leaflet-routing-machine.css";

// Fix marker icons
delete L.Icon.Default.prototype._getIconUrl;
L.Icon.Default.mergeOptions({
  iconRetinaUrl: 'https://cdnjs.cloudflare.com/ajax/libs/leaflet/1.7.1/images/marker-icon-2x.png',
  iconUrl: 'https://cdnjs.cloudflare.com/ajax/libs/leaflet/1.7.1/images/marker-icon.png',
  shadowUrl: 'https://cdnjs.cloudflare.com/ajax/libs/leaflet/1.7.1/images/marker-shadow.png',
});

const redIcon = new L.Icon({
  iconUrl: 'https://raw.githubusercontent.com/pointhi/leaflet-color-markers/master/img/marker-icon-2x-red.png',
  shadowUrl: 'https://cdnjs.cloudflare.com/ajax/libs/leaflet/1.7.1/images/marker-shadow.png',
  iconSize: [25, 41],
  iconAnchor: [12, 41],
  popupAnchor: [1, -34],
  shadowSize: [41, 41],
  className: 'leaflet-marker-icon'
});

const greenIcon = new L.Icon({
  iconUrl: 'https://raw.githubusercontent.com/pointhi/leaflet-color-markers/master/img/marker-icon-2x-green.png',
  shadowUrl: 'https://cdnjs.cloudflare.com/ajax/libs/leaflet/1.7.1/images/marker-shadow.png',
  iconSize: [25, 41],
  iconAnchor: [12, 41],
  popupAnchor: [1, -34],
  shadowSize: [41, 41],
  className: 'leaflet-marker-icon'
});

const map = ref(null);
const mapContainer = ref(null);
const userMarker = ref(null);
const destinationMarker = ref(null);
const routingControl = ref(null);
const routeInstructions = ref([]);
const travelMode = ref('driving');
const currentDuration = ref(0);
const totalDistance = ref(0);
const currentLocation = ref(null);
const isLoading = ref(false);
const waypoints = ref([]);
const destination = ref(null);

onMounted(() => {
  nextTick(() => {
    if (!mapContainer.value) {
      console.error("Map container not found");
      return;
    }

    // Initialize map with default view (New York coordinates)
    map.value = L.map(mapContainer.value, {
      zoomControl: true,
      preferCanvas: true
    }).setView([40.7128, -74.0060], 13);

    L.tileLayer('https://{s}.tile.openstreetmap.org/{z}/{x}/{y}.png', {
      attribution: '&copy; <a href="https://www.openstreetmap.org/copyright">OpenStreetMap</a> contributors'
    }).addTo(map.value);

    // Handle map clicks to add destination
    map.value.on('click', async (e) => {
      if (!currentLocation.value) {
        alert("Please wait for your location to load first");
        return;
      }
      
      destination.value = e.latlng;
      
      if (destinationMarker.value) {
        map.value.removeLayer(destinationMarker.value);
      }
      
      destinationMarker.value = L.marker(e.latlng, {
        icon: redIcon,
        zIndexOffset: 900
      }).addTo(map.value)
        .bindPopup("Destination")
        .openPopup();
      
      waypoints.value = [
        L.latLng(currentLocation.value.lat, currentLocation.value.lng),
        e.latlng
      ];
      
      isLoading.value = true;
      try {
        await updateRoute();
      } catch (error) {
        console.error("Routing error:", error);
        alert("Failed to calculate route");
      }
      isLoading.value = false;
    });

    // Ensure markers stay correct after zoom
    map.value.on('zoomend', () => {
      if (userMarker.value) userMarker.value.update();
      if (destinationMarker.value) destinationMarker.value.update();
      map.value.invalidateSize();
    });

    locateUser();
  });
});

function locateUser() {
  if (!navigator.geolocation) {
    alert("Geolocation is not supported by your browser");
    return;
  }

  navigator.geolocation.watchPosition(
    (pos) => {
      const latlng = L.latLng(pos.coords.latitude, pos.coords.longitude);
      currentLocation.value = latlng;

      if (!userMarker.value) {
        userMarker.value = L.marker(latlng, { 
          icon: greenIcon,
          zIndexOffset: 1000
        }).addTo(map.value)
          .bindPopup("Your location")
          .openPopup();
        map.value.setView(latlng, 15);
      } else {
        userMarker.value.setLatLng(latlng);
      }

      if (destination.value) {
        waypoints.value = [latlng, destination.value];
        updateRoute();
      }
    },
    (err) => {
      console.error("Geolocation error:", err);
      alert("Error getting your location: " + err.message);
    },
    { enableHighAccuracy: true, timeout: 10000 }
  );
}

async function updateRoute() {
  if (waypoints.value.length < 2) return;

  if (routingControl.value) {
    map.value.removeControl(routingControl.value);
    routingControl.value = null;
  }

  routingControl.value = L.Routing.control({
    waypoints: waypoints.value,
    routeWhileDragging: true,
    showAlternatives: false,
    fitSelectedRoutes: 'smart',
    collapsible: true,
    addWaypoints: false,
    draggableWaypoints: true,
    router: new L.Routing.osrmv1({
      serviceUrl: 'https://router.project-osrm.org/route/v1',
      profile: travelMode.value
    }),
    lineOptions: {
      styles: [{ color: getRouteColor(), opacity: 0.8, weight: 6 }],
      extendToWaypoints: true,
      missingRouteTolerance: 0
    },
    createMarker: () => null // We manage markers separately
  }).addTo(map.value);

  routingControl.value.on("routesfound", (e) => {
    const route = e.routes[0];
    totalDistance.value = formatDistance(route.summary.totalDistance);
    currentDuration.value = route.summary.totalTime;
    routeInstructions.value = processInstructions(route.instructions);
    
    setTimeout(() => {
      if (userMarker.value) userMarker.value.update();
      if (destinationMarker.value) destinationMarker.value.update();
      map.value.invalidateSize();
    }, 100);
  });

  routingControl.value.on("routingerror", (e) => {
    console.error("Routing error:", e);
    alert("Could not find a route. Please try a different location.");
  });
}

function processInstructions(instructions) {
  return instructions.map((i) => ({
    instruction: i.text,
    distance: formatDistance(i.distance),
    time: formatDuration(i.time),
    type: i.type
  }));
}

function formatDistance(meters) {
  return meters > 1000 ? `${(meters / 1000).toFixed(1)} km` : `${Math.round(meters)} m`;
}

function formatDuration(seconds) {
  const hours = Math.floor(seconds / 3600);
  const minutes = Math.round((seconds % 3600) / 60);
  return hours ? `${hours}h ${minutes}min` : `${minutes} min`;
}

function getRouteColor() {
  return travelMode.value === 'walking' ? '#34a853' : 
         travelMode.value === 'cycling' ? '#fbbc05' : '#4285F4';
}

function setTravelMode(mode) {
  travelMode.value = mode;
  if (waypoints.value.length >= 2) updateRoute();
}

function clearRoute() {
  if (routingControl.value) {
    map.value.removeControl(routingControl.value);
    routingControl.value = null;
  }
  if (destinationMarker.value) {
    map.value.removeLayer(destinationMarker.value);
    destinationMarker.value = null;
  }
  destination.value = null;
  waypoints.value = [];
  routeInstructions.value = [];
  totalDistance.value = 0;
  currentDuration.value = 0;
}

onUnmounted(() => {
  if (map.value) {
    map.value.remove();
    map.value = null;
  }
});
</script>

<template>
  <div class="map-app">
    <div class="map-controls">
      <button @click="locateUser" :disabled="isLoading">📍 Locate Me</button>
      <div class="travel-modes">
        <button @click="setTravelMode('driving')" :class="{ active: travelMode === 'driving' }" :disabled="isLoading">
          🚗 Drive
        </button>
        <button @click="setTravelMode('walking')" :class="{ active: travelMode === 'walking' }" :disabled="isLoading">
          🚶 Walk
        </button>
        <button @click="setTravelMode('cycling')" :class="{ active: travelMode === 'cycling' }" :disabled="isLoading">
          🚲 Bike
        </button>
      </div>
      <button @click="clearRoute" :disabled="!destination || isLoading">Clear Route</button>
      <div v-if="totalDistance && !isLoading" class="route-summary">
        <div>📏 {{ totalDistance }}</div>
        <div>⏱ {{ formatDuration(currentDuration) }}</div>
      </div>
      <div v-else-if="isLoading" class="loading-indicator">Loading route...</div>
    </div>

    <div ref="mapContainer" class="map-container"></div>

    <div v-if="routeInstructions.length && !isLoading" class="directions">
      <h3>Directions ({{ travelMode }})</h3>
      <ol>
        <li v-for="(step, i) in routeInstructions" :key="i" :class="'step-' + step.type">
          <div class="step-instruction">{{ step.instruction }}</div>
          <div class="step-details">
            <span class="step-distance">{{ step.distance }}</span>
            <span class="step-time">{{ step.time }}</span>
          </div>
        </li>
      </ol>
    </div>
  </div>
</template>

<style>
/* Base styles */
html, body, #app {
  margin: 0;
  padding: 0;
  height: 100%;
  width: 100%;
  font-family: Arial, sans-serif;
}

.map-app {
  display: flex;
  flex-direction: column;
  height: 100vh;
  width: 100vw;
}

.map-container {
  flex: 1;
  min-height: 400px;
  width: 100%;
  background: #f0f0f0;
}

.leaflet-container {
  background: #d4d4d4;
}

/* Controls */
.map-controls {
  padding: 12px;
  background: #f8f9fa;
  display: flex;
  flex-wrap: wrap;
  gap: 10px;
  align-items: center;
  border-bottom: 1px solid #ddd;
}

button {
  padding: 8px 16px;
  background: #e0e0e0;
  border: none;
  border-radius: 4px;
  cursor: pointer;
  font-weight: bold;
  transition: background 0.2s;
}

button:hover:not(:disabled) {
  background: #d0d0d0;
}

button.active {
  background: #4285F4;
  color: white;
}

button:disabled {
  opacity: 0.6;
  cursor: not-allowed;
}

.travel-modes {
  display: flex;
  gap: 8px;
}

.route-summary {
  display: flex;
  gap: 15px;
  margin-left: auto;
  font-weight: 500;
  color: #333;
}

.loading-indicator {
  margin-left: auto;
  color: #666;
  font-style: italic;
}

/* Directions */
.directions {
  padding: 16px;
  background: #f8f9fa;
  border-top: 1px solid #ddd;
  max-height: 300px;
  overflow-y: auto;
}

.directions h3 {
  margin-top: 0;
  color: #333;
}

.directions ol {
  padding-left: 20px;
  margin: 0;
}

.directions li {
  margin: 10px 0;
  padding: 10px;
  background: white;
  border-radius: 4px;
  border-left: 4px solid #4285F4;
  box-shadow: 0 1px 3px rgba(0,0,0,0.1);
}

.step-instruction {
  font-weight: 500;
  margin-bottom: 4px;
}

.step-details {
  display: flex;
  justify-content: space-between;
  font-size: 0.9em;
  color: #666;
}

/* Marker fixes */
.leaflet-marker-icon, .leaflet-marker-shadow {
  transform-origin: center bottom !important;
}

.leaflet-zoom-animated {
  will-change: transform;
  transition: transform 0.25s cubic-bezier(0,0,0.25,1);
}

/* Responsive */
@media (max-width: 768px) {
  .map-controls {
    flex-direction: column;
    align-items: flex-start;
  }
  
  .route-summary {
    margin-left: 0;
    gap: 10px;
  }
  
  .map-container {
    min-height: 300px;
  }
  
  .directions {
    max-height: 200px;
  }
}
</style>