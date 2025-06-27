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
const isLoading = ref(true); // Loading state while getting location

let userMarker = null;
let watchId = null;
let movementPolyline = null;
let positionsHistory = [];
let destinationMarker = null;
let directionLine = null;
let isFirstPosition = true;

function updatePosition(position) {
  lat.value = position.coords.latitude;
  lng.value = position.coords.longitude;
  const currentPosition = [lat.value, lng.value];
  
  // Add to movement history
  positionsHistory.push(currentPosition);
  
  // Center the map on first position only
  if (isFirstPosition) {
    map.value.setView(currentPosition, 15);
    isFirstPosition = false;
    isLoading.value = false;
  }

  // Update or create user marker
  if (!userMarker) {
    userMarker = L.marker(currentPosition, {
      draggable: false,
      rotationAngle: 0,
      rotationOrigin: 'center'
    }).addTo(map.value)
      .bindPopup("Your current location");
  } else {
    userMarker.setLatLng(currentPosition);
  }

  // Update direction if destination exists
  if (destinationMarker) {
    updateDirectionLine();
  }

  // Draw movement path
  if (movementPolyline) {
    map.value.removeLayer(movementPolyline);
  }
  movementPolyline = L.polyline(positionsHistory, {color: 'blue'}).addTo(map.value);
}

function updateDirectionLine() {
  if (!userMarker || !destinationMarker) return;
  
  const userPos = userMarker.getLatLng();
  const destPos = destinationMarker.getLatLng();
  
  // Calculate bearing angle
  const bearing = calculateBearing(userPos, destPos);
  
  // Update user marker rotation
  userMarker.setRotationAngle(bearing);
  
  // Update or create direction line
  if (directionLine) {
    directionLine.setLatLngs([userPos, destPos]);
  } else {
    directionLine = L.polyline([userPos, destPos], {
      color: 'red',
      dashArray: '5, 10',
      weight: 2
    }).addTo(map.value);
  }
}

function calculateBearing(start, end) {
  const startLat = start.lat * Math.PI / 180;
  const startLng = start.lng * Math.PI / 180;
  const endLat = end.lat * Math.PI / 180;
  const endLng = end.lng * Math.PI / 180;
  
  const y = Math.sin(endLng - startLng) * Math.cos(endLat);
  const x = Math.cos(startLat) * Math.sin(endLat) - 
           Math.sin(startLat) * Math.cos(endLat) * Math.cos(endLng - startLng);
  
  return (Math.atan2(y, x) * 180 / Math.PI + 360) % 360;
}

function handleMapClick(e) {
  if (!isTracking.value) return;
  
  // Remove previous destination marker
  if (destinationMarker) {
    map.value.removeLayer(destinationMarker);
    if (directionLine) {
      map.value.removeLayer(directionLine);
      directionLine = null;
    }
  }
  
  // Add new destination marker
  destinationMarker = L.marker(e.latlng, {
    draggable: true,
    icon: L.icon({
      iconUrl: 'https://cdn0.iconfinder.com/data/icons/small-n-flat/24/678111-map-marker-512.png',
      iconSize: [32, 32]
    })
  }).addTo(map.value)
    .bindPopup("Destination")
    .openPopup();
  
  // Update direction line
  if (userMarker) {
    updateDirectionLine();
  }
  
  // When destination is dragged
  destinationMarker.on('drag', () => {
    updateDirectionLine();
  });
}

function startTracking() {
  if (navigator.geolocation) {
    isTracking.value = true;
    positionsHistory = [];
    
    // Set up map click handler
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
  
  // Show finish tracking alert with stats
  if (positionsHistory.length > 0) {
    const distance = calculateTotalDistance();
    alert(`Tracking finished!\nTotal distance: ${distance.toFixed(2)} meters`);
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
  // Initialize map with minimal view
  map.value = L.map(mapContainer.value).setView([0, 0], 1);
  L.tileLayer("https://tile.openstreetmap.org/{z}/{x}/{y}.png", {
    maxZoom: 19,
    attribution: '&copy; <a href="http://www.openstreetmap.org/copyright">OpenStreetMap</a>',
  }).addTo(map.value);

  // Start tracking immediately
  startTracking();
});

onUnmounted(() => {
  stopTracking();
});
</script>

<template>
  <div>
    <p>Latitude: {{ lat.toFixed(6) }}, Longitude: {{ lng.toFixed(6) }}</p>
    <div v-if="isLoading" class="loading">Getting your location...</div>
    <button @click="startTracking" v-if="!isTracking && !isLoading">Start Tracking</button>
    <button @click="stopTracking" v-else-if="isTracking">Stop Tracking</button>
    <div ref="mapContainer" style="width: 100%; height: 500px;"></div>
  </div>
</template>

<style>
.loading {
  padding: 10px;
  background: #f0f0f0;
  margin-bottom: 10px;
  text-align: center;
}

button {
  padding: 8px 16px;
  margin-right: 10px;
  background: #42b983;
  color: white;
  border: none;
  border-radius: 4px;
  cursor: pointer;
}

button:hover {
  background: #369f6b;
}
</style>