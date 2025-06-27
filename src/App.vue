<script setup>
import { ref, onMounted, onUnmounted } from "vue";
import L from "leaflet";
import "leaflet/dist/leaflet.css";
import "leaflet-rotatedmarker";

// Fix for missing Leaflet marker icons
delete L.Icon.Default.prototype._getIconUrl;
L.Icon.Default.mergeOptions({
  iconRetinaUrl: 'https://cdnjs.cloudflare.com/ajax/libs/leaflet/1.7.1/images/marker-icon-2x.png',
  iconUrl: 'https://cdnjs.cloudflare.com/ajax/libs/leaflet/1.7.1/images/marker-icon.png',
  shadowUrl: 'https://cdnjs.cloudflare.com/ajax/libs/leaflet/1.7.1/images/marker-shadow.png',
});

const lat = ref(0);
const lng = ref(0);
const map = ref();
const mapContainer = ref();
const isTracking = ref(false);
const isLoading = ref(true);
const routeInstructions = ref([]);
const totalDistance = ref(0);
const totalDuration = ref(0);
const currentSpeed = ref(0);
const eta = ref('Calculating...');
const routeMode = ref('fastest'); // 'fastest' or 'direct'

let userMarker = null;
let watchId = null;
let movementPolyline = null;
let positionsHistory = [];
let destinationMarker = null;
let routeLine = null;
let isFirstPosition = true;
let lastPosition = null;
let lastPositionTime = null;

function updatePosition(position) {
  if (!map.value) return;
  
  lat.value = position.coords.latitude;
  lng.value = position.coords.longitude;
  const currentPosition = [lat.value, lng.value];
  
  // Calculate current speed
  if (lastPosition && lastPositionTime) {
    const distance = map.value.distance(lastPosition, currentPosition);
    const timeDiff = (Date.now() - lastPositionTime) / 1000;
    if (timeDiff > 0) {
      currentSpeed.value = (distance / timeDiff * 3.6).toFixed(1); // km/h
    }
  }
  
  lastPosition = currentPosition;
  lastPositionTime = Date.now();
  positionsHistory.push(currentPosition);
  
  if (isFirstPosition) {
    try {
      map.value.setView(currentPosition, 15, { animate: false });
      isFirstPosition = false;
      isLoading.value = false;
    } catch (e) {
      console.warn("Map setView error:", e);
    }
  }

  // Update user marker
  if (!userMarker) {
    userMarker = L.marker(currentPosition, {
      draggable: false,
      icon: L.icon({
        iconUrl: 'https://cdnjs.cloudflare.com/ajax/libs/leaflet/1.7.1/images/marker-icon.png',
        iconSize: [25, 41],
        iconAnchor: [12, 41]
      })
    }).addTo(map.value)
      .bindPopup("Your current location");
  } else {
    userMarker.setLatLng(currentPosition);
  }

  // Update route if destination exists
  if (destinationMarker) {
    updateRoute();
  }

  // Update movement path
  if (movementPolyline) {
    try {
      map.value.removeLayer(movementPolyline);
    } catch (e) {
      console.warn("Polyline removal error:", e);
    }
  }
  movementPolyline = L.polyline(positionsHistory, {color: 'blue'}).addTo(map.value);
}

async function updateRoute() {
  if (!map.value || !userMarker || !destinationMarker) return;
  
  const start = userMarker.getLatLng();
  const end = destinationMarker.getLatLng();
  
  // Clear previous route
  if (routeLine) {
    try {
      map.value.removeLayer(routeLine);
    } catch (e) {
      console.warn("Route removal error:", e);
    }
  }
  
  if (routeMode.value === 'direct') {
    // Show direct beeline
    routeLine = L.polyline([start, end], {
      color: '#93c5fd',
      weight: 3,
      dashArray: '5, 5'
    }).addTo(map.value);
    
    const distance = map.value.distance(start, end);
    totalDistance.value = (distance / 1000).toFixed(2);
    totalDuration.value = 'N/A';
    eta.value = 'N/A';
    
    routeInstructions.value = [{
      instruction: "Direct path to destination",
      distance: totalDistance.value + ' km',
      duration: 'N/A',
      type: 'direct'
    }];
    
    return;
  }
  
  try {
    // Show temporary beeline while loading
    const tempLine = L.polyline([start, end], {
      color: '#93c5fd',
      weight: 3,
      dashArray: '5, 5'
    }).addTo(map.value);
    
    // Get route from OSRM
    const response = await fetch(
      `https://router.project-osrm.org/route/v1/driving/${start.lng},${start.lat};${end.lng},${end.lat}?overview=full&geometries=geojson&steps=true`
    );
    const data = await response.json();
    
    // Remove temporary line
    try {
      map.value.removeLayer(tempLine);
    } catch (e) {
      console.warn("Temp line removal error:", e);
    }
    
    if (data.routes?.[0]) {
      const route = data.routes[0];
      totalDistance.value = (route.distance / 1000).toFixed(2);
      totalDuration.value = (route.duration / 60).toFixed(2);
      
      // Calculate ETA
      if (currentSpeed.value > 0) {
        const speedKmh = parseFloat(currentSpeed.value);
        const distanceKm = parseFloat(totalDistance.value);
        const estimatedHours = distanceKm / speedKmh;
        const hours = Math.floor(estimatedHours);
        const minutes = Math.round((estimatedHours - hours) * 60);
        eta.value = `${hours > 0 ? hours + 'h ' : ''}${minutes}m`;
      } else {
        eta.value = `${totalDuration.value} min (estimate)`;
      }
      
      // Process instructions
      routeInstructions.value = (route.legs[0]?.steps || []).map(step => ({
        instruction: cleanInstruction(step.maneuver?.instruction),
        distance: (step.distance / 1000).toFixed(2) + ' km',
        duration: (step.duration / 60).toFixed(2) + ' min',
        type: step.maneuver?.type || 'turn'
      }));
      
      // Draw route
      routeLine = L.geoJSON(route.geometry, {
        style: {
          color: '#3b82f6',
          weight: 5,
          opacity: 0.7
        }
      }).addTo(map.value);
      
      // Fit bounds safely
      try {
        const bounds = L.latLngBounds([start, end]);
        map.value.fitBounds(bounds, { 
          padding: [50, 50],
          animate: false 
        });
      } catch (e) {
        console.warn("FitBounds error:", e);
        map.value.setView(start, 15, { animate: false });
      }
    }
  } catch (error) {
    console.error("Routing error:", error);
    // Fallback to direct line
    routeLine = L.polyline([start, end], {
      color: '#93c5fd',
      weight: 3,
      dashArray: '5, 5'
    }).addTo(map.value);
    
    routeInstructions.value = [{
      instruction: "Could not load detailed directions - showing direct path",
      distance: map.value.distance(start, end).toFixed(0) + ' m',
      duration: 'N/A',
      type: 'alert'
    }];
  }
}

function cleanInstruction(text) {
  if (!text) return "Continue";
  try {
    return text.replace(/<\/?[^>]+(>|$)/g, "");
  } catch {
    return text;
  }
}

function handleMapClick(e) {
  if (!isTracking.value || !map.value) return;
  
  // Clear previous destination
  if (destinationMarker) {
    try {
      map.value.removeLayer(destinationMarker);
    } catch (e) {
      console.warn("Marker removal error:", e);
    }
  }
  
  if (routeLine) {
    try {
      map.value.removeLayer(routeLine);
    } catch (e) {
      console.warn("Route removal error:", e);
    }
    routeLine = null;
  }
  
  routeInstructions.value = [];
  totalDistance.value = 0;
  totalDuration.value = 0;
  eta.value = 'Calculating...';
  
  // Create new destination
  destinationMarker = L.marker(e.latlng, {
    draggable: true,
    icon: L.icon({
      iconUrl: 'https://cdn0.iconfinder.com/data/icons/small-n-flat/24/678111-map-marker-512.png',
      iconSize: [32, 32]
    })
  }).addTo(map.value)
    .bindPopup("Destination")
    .openPopup();
  
  // Update route immediately
  updateRoute();
  
  // Update on drag with debounce
  let dragTimeout;
  destinationMarker.on('drag', () => {
    clearTimeout(dragTimeout);
    dragTimeout = setTimeout(() => {
      if (map.value && userMarker) updateRoute();
    }, 200);
  });
}

function toggleRouteMode() {
  routeMode.value = routeMode.value === 'fastest' ? 'direct' : 'fastest';
  if (destinationMarker) updateRoute();
}

function startTracking() {
  if (navigator.geolocation) {
    isTracking.value = true;
    positionsHistory = [];
    routeInstructions.value = [];
    totalDistance.value = 0;
    totalDuration.value = 0;
    eta.value = 'Calculating...';
    
    map.value?.off('click');
    map.value?.on('click', handleMapClick);
    
    watchId = navigator.geolocation.watchPosition(
      updatePosition,
      (error) => {
        console.error("Geolocation error:", error.message);
        alert("Location tracking failed.");
        isLoading.value = false;
      },
      {
        enableHighAccuracy: true,
        timeout: 10000,
        maximumAge: 0
      }
    );
  } else {
    alert("Geolocation is not supported by this browser.");
    isLoading.value = false;
  }
}

function stopTracking() {
  if (watchId) {
    navigator.geolocation.clearWatch(watchId);
    watchId = null;
  }
  
  isTracking.value = false;
  map.value?.off('click');
  
  if (positionsHistory.length > 0) {
    const distance = calculateTotalDistance();
    alert(`Tracking finished!\nTotal distance: ${(distance / 1000).toFixed(2)} km`);
  }
}

function calculateTotalDistance() {
  let total = 0;
  for (let i = 1; i < positionsHistory.length; i++) {
    total += map.value.distance(
      positionsHistory[i-1],
      positionsHistory[i]
    );
  }
  return total;
}

onMounted(() => {
  // Initialize map with safety checks
  const initMap = () => {
    if (!mapContainer.value) {
      requestAnimationFrame(initMap);
      return;
    }
    
    map.value = L.map(mapContainer.value, {
      zoomControl: false,
      preferCanvas: true // Better for frequent updates
    }).setView([0, 0], 1);
    
    L.tileLayer("https://tile.openstreetmap.org/{z}/{x}/{y}.png", {
      maxZoom: 19,
      attribution: '&copy; <a href="http://www.openstreetmap.org/copyright">OpenStreetMap</a>',
    }).addTo(map.value);
    
    // Add zoom control after map is ready
    L.control.zoom({ position: 'topright' }).addTo(map.value);
    
    startTracking();
  };
  
  initMap();
});

onUnmounted(() => {
  stopTracking();
  if (map.value) {
    try {
      map.value.remove();
    } catch (e) {
      console.warn("Map removal error:", e);
    }
  }
});
</script>

<template>
  <div>
    <div class="controls">
      <p>Current Location: {{ lat.toFixed(6) }}, {{ lng.toFixed(6) }}</p>
      <p v-if="currentSpeed > 0">Speed: {{ currentSpeed }} km/h</p>
      <div v-if="isLoading" class="loading">Getting your location...</div>
      
      <div class="buttons">
        <button @click="startTracking" v-if="!isTracking && !isLoading">Start Tracking</button>
        <button @click="stopTracking" v-else-if="isTracking">Stop Tracking</button>
        <button @click="toggleRouteMode" v-if="isTracking">
          {{ routeMode === 'fastest' ? 'Show Direct Path' : 'Show Fastest Route' }}
        </button>
      </div>
      
      <div v-if="destinationMarker" class="route-info">
        <h3>Route Information ({{ routeMode }})</h3>
        <div class="route-summary">
          <div class="route-stat">
            <span class="stat-value">{{ totalDistance }} km</span>
            <span class="stat-label">Distance</span>
          </div>
          <div class="route-stat">
            <span class="stat-value">{{ totalDuration }} min</span>
            <span class="stat-label">Duration</span>
          </div>
          <div class="route-stat">
            <span class="stat-value">{{ eta }}</span>
            <span class="stat-label">ETA</span>
          </div>
        </div>
        
        <div class="instructions" v-if="routeInstructions.length">
          <h4>Directions:</h4>
          <ol>
            <li v-for="(step, index) in routeInstructions" :key="index" :class="step.type">
              <span class="instruction-text">{{ step.instruction }}</span>
              <span class="instruction-distance">{{ step.distance }} • {{ step.duration }}</span>
            </li>
          </ol>
        </div>
      </div>
    </div>
    
    <div ref="mapContainer" style="width: 100%; height: 500px;"></div>
  </div>
</template>

<style>
.controls {
  padding: 15px;
  background: #f8f9fa;
  border-bottom: 1px solid #dee2e6;
  font-family: Arial, sans-serif;
}

.loading {
  padding: 10px;
  background: #f0f0f0;
  margin: 10px 0;
  text-align: center;
  border-radius: 4px;
}

.buttons {
  margin: 10px 0;
  display: flex;
  gap: 10px;
  flex-wrap: wrap;
}

button {
  padding: 8px 16px;
  background: #3b82f6;
  color: white;
  border: none;
  border-radius: 4px;
  cursor: pointer;
  flex: 1;
  min-width: 120px;
  font-size: 14px;
}

button:hover {
  background: #2563eb;
}

.route-info {
  margin-top: 15px;
  padding: 15px;
  background: white;
  border-radius: 8px;
  box-shadow: 0 1px 3px rgba(0,0,0,0.1);
}

.route-summary {
  display: flex;
  justify-content: space-between;
  margin-bottom: 15px;
  padding-bottom: 15px;
  border-bottom: 1px solid #eee;
  gap: 10px;
}

.route-stat {
  text-align: center;
  flex: 1;
  min-width: 80px;
}

.stat-value {
  display: block;
  font-size: 1.1rem;
  font-weight: bold;
  color: #1e40af;
}

.stat-label {
  display: block;
  font-size: 0.8rem;
  color: #64748b;
  margin-top: 4px;
}

.instructions {
  max-height: 200px;
  overflow-y: auto;
  margin-top: 10px;
  padding: 10px;
  background: #f8f9fa;
  border-radius: 8px;
}

.instructions ol {
  padding-left: 20px;
  margin: 0;
}

.instructions li {
  margin: 8px 0;
  padding: 8px;
  border-radius: 4px;
  background: white;
  font-size: 14px;
}

.instruction-text {
  display: block;
  margin-bottom: 4px;
}

.instruction-distance {
  font-size: 0.8rem;
  color: #64748b;
}

.depart, .arrive, .alert {
  font-weight: bold;
  color: #1e40af;
}

.direct {
  font-style: italic;
  color: #4b5563;
}

.turn {
  padding-left: 20px;
  position: relative;
}

.turn:before {
  content: "↳";
  position: absolute;
  left: 5px;
  color: #3b82f6;
}

h3, h4 {
  margin: 5px 0;
  color: #1e40af;
  font-size: 1.1rem;
}

h4 {
  font-size: 1rem;
  margin-top: 10px;
}

/* Leaflet overrides */
.leaflet-control-zoom {
  margin-top: 60px !important;
}

.leaflet-popup-content {
  font-size: 14px;
}
</style>