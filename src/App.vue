<script setup>
import { ref, onMounted, onUnmounted, nextTick } from "vue";
import L from "leaflet";
import "leaflet/dist/leaflet.css";
import "leaflet-routing-machine";
import "leaflet-routing-machine/dist/leaflet-routing-machine.css";
import axios from "axios";

// Route tracking variables
const currentPositionOnRoute = ref(0);
const routePolyline = ref(null);
const routeCoordinates = ref([]);
const remainingDistance = ref(0);
const remainingTime = ref(0);
let trackingInterval = null;

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

const blueIcon = new L.Icon({
  iconUrl: 'https://raw.githubusercontent.com/pointhi/leaflet-color-markers/master/img/marker-icon-2x-blue.png',
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
const showDirections = ref(false);

onMounted(() => {
  nextTick(() => {
    if (!mapContainer.value) {
      console.error("Map container not found");
      return;
    }

    try {
      // Initialize map with default view (New York coordinates)
      map.value = L.map(mapContainer.value, {
        zoomControl: true,
        preferCanvas: true,
        renderer: L.canvas()
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

      // Improved zoom handling
      map.value.on('zoomstart', () => {
        if (!map.value || map.value._removed) return;
      });

      map.value.on('zoomend', () => {
        try {
          if (userMarker.value && map.value.hasLayer(userMarker.value)) {
            userMarker.value.update();
          }
          if (destinationMarker.value && map.value.hasLayer(destinationMarker.value)) {
            destinationMarker.value.update();
          }
          map.value.invalidateSize();
        } catch (e) {
          console.warn("Zoom animation error:", e);
        }
      });

      locateUser();
    } catch (error) {
      console.error("Map initialization error:", error);
      alert("Failed to initialize map. Please try refreshing the page.");
    }
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

      if (!userMarker.value || !map.value.hasLayer(userMarker.value)) {
        if (userMarker.value) map.value.removeLayer(userMarker.value);
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
      
      // Update position along route if tracking
      if (routeCoordinates.value.length) {
        updatePositionOnRoute();
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
  if (!map.value || map.value._removed) return;
  if (waypoints.value.length < 2) return;

  isLoading.value = true;
  
  try {
    // Clear existing route if any
    if (routingControl.value) {
      map.value.removeControl(routingControl.value);
      routingControl.value = null;
    }
    
    if (routePolyline.value) {
      map.value.removeLayer(routePolyline.value);
    }

    // Use direct OSRM API call for more control
    const response = await axios.get(
      `https://router.project-osrm.org/route/v1/${travelMode.value}/` +
      `${waypoints.value[0].lng},${waypoints.value[0].lat};` +
      `${waypoints.value[1].lng},${waypoints.value[1].lat}?` +
      'overview=full&geometries=geojson&steps=true'
    );

    const route = response.data.routes[0];
    
    // Store route coordinates for tracking
    routeCoordinates.value = route.geometry.coordinates.map(coord => [coord[1], coord[0]]);
    
    // Create polyline for the route
    routePolyline.value = L.polyline(routeCoordinates.value, {
      color: getRouteColor(),
      weight: 6,
      opacity: 0.8
    }).addTo(map.value);

    // Update route summary
    totalDistance.value = route.distance;
    currentDuration.value = route.duration;
    remainingDistance.value = route.distance;
    remainingTime.value = route.duration;
    
    // Process instructions
    routeInstructions.value = processInstructions(route.legs[0].steps);
    
    // Fit bounds to route
    const bounds = L.latLngBounds(routeCoordinates.value);
    map.value.fitBounds(bounds, { padding: [50, 50] });
    
    // Start tracking position along route
    startRouteTracking();
    
  } catch (error) {
    console.error("Routing error:", error);
    alert("Could not find a route. Please try a different location.");
  } finally {
    isLoading.value = false;
  }
}

function startRouteTracking() {
  stopRouteTracking();
  
  // Only track if we have a route and user location
  if (!routeCoordinates.value.length || !currentLocation.value) return;
  
  // Find closest point on route to current location
  updatePositionOnRoute();
  
  // Update position periodically
  trackingInterval = setInterval(updatePositionOnRoute, 5000);
}

function stopRouteTracking() {
  if (trackingInterval) {
    clearInterval(trackingInterval);
    trackingInterval = null;
  }
}

function updatePositionOnRoute() {
  if (!routeCoordinates.value.length || !currentLocation.value) return;
  
  // Find the closest point on the route to current location
  let closestIndex = 0;
  let minDistance = Infinity;
  
  routeCoordinates.value.forEach((coord, index) => {
    const distance = map.value.distance(
      L.latLng(coord),
      L.latLng(currentLocation.value.lat, currentLocation.value.lng)
    );
    
    if (distance < minDistance) {
      minDistance = distance;
      closestIndex = index;
    }
  });
  
  // Calculate progress (0-1)
  currentPositionOnRoute.value = closestIndex / routeCoordinates.value.length;
  
  // Update remaining distance and time
  updateRouteProgress();
}

function updateRouteProgress() {
  if (!routeCoordinates.value.length) return;
  
  // Calculate remaining distance and time based on progress
  remainingDistance.value = totalDistance.value * (1 - currentPositionOnRoute.value);
  remainingTime.value = currentDuration.value * (1 - currentPositionOnRoute.value);
}

function processInstructions(steps) {
  return steps.flatMap(step => {
    // Skip very short steps
    if (step.distance < 10) return [];
    
    return {
      instruction: step.maneuver.instruction,
      distance: formatDistance(step.distance),
      time: formatDuration(step.duration),
      type: step.maneuver.type,
      modifier: step.maneuver.modifier
    };
  });
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
  stopRouteTracking();
  
  if (routingControl.value) {
    map.value.removeControl(routingControl.value);
    routingControl.value = null;
  }
  
  if (routePolyline.value) {
    map.value.removeLayer(routePolyline.value);
    routePolyline.value = null;
  }
  
  if (destinationMarker.value) {
    map.value.removeLayer(destinationMarker.value);
    destinationMarker.value = null;
  }
  
  destination.value = null;
  waypoints.value = [];
  routeInstructions.value = [];
  routeCoordinates.value = [];
  totalDistance.value = 0;
  currentDuration.value = 0;
  remainingDistance.value = 0;
  remainingTime.value = 0;
  currentPositionOnRoute.value = 0;
  showDirections.value = false;
}

onUnmounted(() => {
  stopRouteTracking();
  
  if (map.value) {
    try {
      // Remove all layers
      map.value.eachLayer(layer => {
        try {
          map.value.removeLayer(layer);
        } catch (e) {
          console.warn("Error removing layer:", e);
        }
      });
      
      // Remove the map instance
      map.value.remove();
    } catch (e) {
      console.error("Error during map cleanup:", e);
    } finally {
      map.value = null;
    }
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
      <button @click="showDirections = !showDirections" :disabled="!routeInstructions.length || isLoading">
        {{ showDirections ? 'Hide' : 'Show' }} Directions
      </button>
      <div v-if="totalDistance && !isLoading" class="route-summary">
        <div>📏 {{ formatDistance(totalDistance) }}</div>
        <div>⏱ {{ formatDuration(currentDuration) }}</div>
        <div v-if="currentPositionOnRoute > 0">
          🚀 {{ Math.round(currentPositionOnRoute * 100) }}% · 
          {{ formatDistance(remainingDistance) }} left · 
          {{ formatDuration(remainingTime) }}
        </div>
      </div>
      <div v-else-if="isLoading" class="loading-indicator">Loading route...</div>
    </div>

    <div ref="mapContainer" class="map-container"></div>

    <div v-if="showDirections && routeInstructions.length && !isLoading" class="directions">
      <h3>Directions ({{ travelMode }}) 
        <button @click="showDirections = false" class="close-btn">×</button>
      </h3>
      <div class="progress-container">
        <div class="progress-bar">
          <div class="progress-fill" :style="{ width: `${currentPositionOnRoute * 100}%` }"></div>
        </div>
        <div class="progress-text">
          {{ Math.round(currentPositionOnRoute * 100) }}% complete
        </div>
      </div>
      <ol>
        <li v-for="(step, i) in routeInstructions" :key="i" :class="['step', `step-${step.type}`, { 'step-current': i === Math.floor(currentPositionOnRoute * routeInstructions.length) }]">
          <div class="step-icon">
            <span v-if="step.type === 'depart'">🚦</span>
            <span v-else-if="step.type === 'arrive'">🏁</span>
            <span v-else-if="step.modifier === 'left'">⬅️</span>
            <span v-else-if="step.modifier === 'right'">➡️</span>
            <span v-else-if="step.modifier === 'sharp left'">↖️</span>
            <span v-else-if="step.modifier === 'sharp right'">↗️</span>
            <span v-else-if="step.modifier === 'slight left'">↙️</span>
            <span v-else-if="step.modifier === 'slight right'">↘️</span>
            <span v-else>🛣️</span>
          </div>
          <div class="step-content">
            <div class="step-instruction">{{ step.instruction }}</div>
            <div class="step-details">
              <span class="step-distance">{{ step.distance }}</span>
              <span class="step-time">{{ step.time }}</span>
            </div>
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
  flex-direction: column;
  gap: 4px;
  margin-left: auto;
  font-weight: 500;
  color: #333;
  text-align: right;
  font-size: 0.9em;
}

.route-summary > div {
  display: flex;
  align-items: center;
  gap: 6px;
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
  display: flex;
  justify-content: space-between;
  align-items: center;
}

.directions ol {
  padding-left: 0;
  margin: 10px 0 0;
  list-style: none;
}

.directions li {
  margin: 8px 0;
  padding: 8px;
  background: white;
  border-radius: 4px;
  display: flex;
  gap: 10px;
  align-items: center;
  transition: all 0.2s;
}

.step-current {
  border-left: 4px solid #4285F4;
  background-color: #f0f7ff !important;
}

.step-icon {
  font-size: 1.2em;
  min-width: 30px;
  text-align: center;
}

.step-content {
  flex: 1;
}

.step-instruction {
  font-weight: 500;
  margin-bottom: 4px;
}

.step-details {
  display: flex;
  justify-content: space-between;
  font-size: 0.85em;
  color: #666;
}

.close-btn {
  background: none;
  border: none;
  font-size: 1.2em;
  padding: 0 8px;
  margin-left: 10px;
  color: #666;
}

.close-btn:hover {
  color: #333;
  background: none;
}

/* Progress bar */
.progress-container {
  margin: 10px 0 15px;
}

.progress-bar {
  height: 8px;
  background: #e0e0e0;
  border-radius: 4px;
  overflow: hidden;
}

.progress-fill {
  height: 100%;
  background: #4285F4;
  transition: width 0.3s ease;
}

.progress-text {
  font-size: 0.8em;
  color: #666;
  text-align: center;
  margin-top: 4px;
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
    gap: 4px;
    text-align: left;
    width: 100%;
  }
  
  .map-container {
    min-height: 300px;
  }
  
  .directions {
    max-height: 200px;
  }
}
</style>