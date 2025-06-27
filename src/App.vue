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
  lat.value = position.coords.latitude;
  lng.value = position.coords.longitude;
  const currentPosition = [lat.value, lng.value];
  
  // Calculate current speed if we have previous position
  if (lastPosition && lastPositionTime) {
    const distance = map.value.distance(lastPosition, currentPosition);
    const timeDiff = (Date.now() - lastPositionTime) / 1000; // seconds
    if (timeDiff > 0) {
      currentSpeed.value = distance / timeDiff; // meters per second
      currentSpeed.value = (currentSpeed.value * 3.6).toFixed(1); // convert to km/h
    }
  }
  
  lastPosition = currentPosition;
  lastPositionTime = Date.now();
  
  positionsHistory.push(currentPosition);
  
  if (isFirstPosition) {
    map.value.setView(currentPosition, 15);
    isFirstPosition = false;
    isLoading.value = false;
  }

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

  if (destinationMarker) {
    updateRoute();
  }

  if (movementPolyline) {
    map.value.removeLayer(movementPolyline);
  }
  movementPolyline = L.polyline(positionsHistory, {color: 'blue'}).addTo(map.value);
}

async function updateRoute() {
  if (!userMarker || !destinationMarker) return;
  
  const start = userMarker.getLatLng();
  const end = destinationMarker.getLatLng();
  
  try {
    const response = await fetch(
      `https://router.project-osrm.org/route/v1/driving/${start.lng},${start.lat};${end.lng},${end.lat}?overview=full&geometries=geojson&steps=true`
    );
    const data = await response.json();
    
    if (data.routes && data.routes.length > 0) {
      const route = data.routes[0];
      totalDistance.value = (route.distance / 1000).toFixed(2); // km
      totalDuration.value = (route.duration / 60).toFixed(2); // minutes
      
      // Calculate ETA based on current speed if available
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
      
      // Update route instructions
      routeInstructions.value = route.legs[0].steps.map(step => ({
        instruction: step.maneuver.instruction.replace(/<\/?[^>]+(>|$)/g, ""), // Remove HTML tags
        distance: (step.distance / 1000).toFixed(2) + ' km',
        duration: (step.duration / 60).toFixed(2) + ' min',
        type: step.maneuver.type
      }));
      
      // Draw the route
      if (routeLine) {
        map.value.removeLayer(routeLine);
      }
      
      routeLine = L.geoJSON(data.routes[0].geometry, {
        style: {
          color: '#3b82f6',
          weight: 5,
          opacity: 0.7
        }
      }).addTo(map.value);
      
      // Fit map to route bounds with padding
      const bounds = L.latLngBounds([start, end]);
      map.value.fitBounds(bounds, { padding: [50, 50] });
      
      // Add distance markers every 1km
      addDistanceMarkers(data.routes[0].geometry);
    }
  } catch (error) {
    console.error("Routing error:", error);
  }
}

function addDistanceMarkers(geometry) {
  // Remove existing distance markers
  map.value.eachLayer(layer => {
    if (layer.options && layer.options.isDistanceMarker) {
      map.value.removeLayer(layer);
    }
  });
  
  // Calculate cumulative distance and add markers every 1km
  let cumulativeDistance = 0;
  const coordinates = geometry.coordinates;
  
  for (let i = 1; i < coordinates.length; i++) {
    const prev = coordinates[i-1];
    const curr = coordinates[i];
    const segmentDistance = map.value.distance(
      [prev[1], prev[0]],
      [curr[1], curr[0]]
    );
    
    cumulativeDistance += segmentDistance;
    
    if (cumulativeDistance >= 1000) { // Every 1km
      const marker = L.marker([curr[1], curr[0]], {
        icon: L.divIcon({
          html: `${(cumulativeDistance / 1000).toFixed(1)}km`,
          className: 'distance-marker',
          iconSize: [60, 20]
        }),
        isDistanceMarker: true
      }).addTo(map.value);
      
      cumulativeDistance = 0; // Reset for next marker
    }
  }
}

function handleMapClick(e) {
  if (!isTracking.value) return;
  
  if (destinationMarker) {
    map.value.removeLayer(destinationMarker);
    if (routeLine) {
      map.value.removeLayer(routeLine);
      routeLine = null;
    }
    routeInstructions.value = [];
    totalDistance.value = 0;
    totalDuration.value = 0;
    eta.value = 'Calculating...';
  }
  
  destinationMarker = L.marker(e.latlng, {
    draggable: true,
    icon: L.icon({
      iconUrl: 'https://cdn0.iconfinder.com/data/icons/small-n-flat/24/678111-map-marker-512.png',
      iconSize: [32, 32]
    })
  }).addTo(map.value)
    .bindPopup("Destination")
    .openPopup();
  
  if (userMarker) {
    updateRoute();
  }
  
  destinationMarker.on('drag', () => {
    updateRoute();
  });
}

function startTracking() {
  if (navigator.geolocation) {
    isTracking.value = true;
    positionsHistory = [];
    routeInstructions.value = [];
    totalDistance.value = 0;
    totalDuration.value = 0;
    eta.value = 'Calculating...';
    
    map.value.off('click');
    map.value.on('click', handleMapClick);
    
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
  map.value.off('click');
  
  if (positionsHistory.length > 0) {
    const distance = calculateTotalDistance();
    alert(`Tracking finished!\nTotal distance traveled: ${(distance / 1000).toFixed(2)} km`);
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
  map.value = L.map(mapContainer.value).setView([0, 0], 1);
  L.tileLayer("https://tile.openstreetmap.org/{z}/{x}/{y}.png", {
    maxZoom: 19,
    attribution: '&copy; <a href="http://www.openstreetmap.org/copyright">OpenStreetMap</a>',
  }).addTo(map.value);

  startTracking();
});

onUnmounted(() => {
  stopTracking();
});
</script>

<template>
  <div>
    <div class="controls">
      <p>Current Location: {{ lat.toFixed(6) }}, {{ lng.toFixed(6) }}</p>
      <p v-if="currentSpeed > 0">Current Speed: {{ currentSpeed }} km/h</p>
      <div v-if="isLoading" class="loading">Getting your location...</div>
      <div class="buttons">
        <button @click="startTracking" v-if="!isTracking && !isLoading">Start Tracking</button>
        <button @click="stopTracking" v-else-if="isTracking">Stop Tracking</button>
      </div>
      
      <div v-if="destinationMarker" class="route-info">
        <h3>Route Information</h3>
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
        
        <div class="instructions">
          <h4>Turn-by-Turn Directions:</h4>
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
}

.loading {
  padding: 10px;
  background: #f0f0f0;
  margin: 10px 0;
  text-align: center;
}

.buttons {
  margin: 10px 0;
  display: flex;
  gap: 10px;
}

button {
  padding: 8px 16px;
  background: #3b82f6;
  color: white;
  border: none;
  border-radius: 4px;
  cursor: pointer;
  flex: 1;
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
}

.route-stat {
  text-align: center;
  flex: 1;
}

.stat-value {
  display: block;
  font-size: 1.2rem;
  font-weight: bold;
  color: #1e40af;
}

.stat-label {
  display: block;
  font-size: 0.8rem;
  color: #64748b;
}

.instructions {
  max-height: 300px;
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
}

.instruction-text {
  display: block;
  margin-bottom: 4px;
}

.instruction-distance {
  font-size: 0.8rem;
  color: #64748b;
}

.depart, .arrive {
  font-weight: bold;
  color: #1e40af;
}

.turn {
  padding-left: 20px;
  position: relative;
}

.turn:before {
  content: "↳";
  position: absolute;
  left: 5px;
}

.distance-marker {
  background: white;
  padding: 2px 5px;
  border-radius: 3px;
  border: 1px solid #ccc;
  font-size: 0.8rem;
  font-weight: bold;
  text-align: center;
}

h3, h4 {
  margin: 5px 0;
  color: #1e40af;
}
</style>