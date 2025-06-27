<script setup>
import { ref, onMounted, onUnmounted } from "vue";
import L from "leaflet";
import "leaflet/dist/leaflet.css";
import "leaflet-rotatedmarker"; // Import the rotation plugin

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
let userMarker = null;
let watchId = null;
let movementPolyline = null;
let positionsHistory = [];

function updatePosition(position) {
  lat.value = position.coords.latitude;
  lng.value = position.coords.longitude;
  const currentPosition = [lat.value, lng.value];
  
  // Add to movement history
  positionsHistory.push(currentPosition);
  
  // Center the map on new position
  map.value.setView(currentPosition, 15);

  // Calculate bearing angle
  const bearing = positionsHistory.length > 1 ? getBearingAngle() : 0;

  // Update or create marker with rotation
  if (!userMarker) {
    userMarker = L.marker(currentPosition, {
      draggable: false,
      rotationAngle: bearing,
      rotationOrigin: 'center'
    }).addTo(map.value)
      .bindPopup("Your current location");
  } else {
    userMarker.setLatLng(currentPosition);
    userMarker.setRotationAngle(bearing); // Now this will work
  }

  // Draw movement path
  if (movementPolyline) {
    map.value.removeLayer(movementPolyline);
  }
  movementPolyline = L.polyline(positionsHistory, {color: 'blue'}).addTo(map.value);
}

// Calculate direction angle between points
function getBearingAngle() {
  if (positionsHistory.length < 2) return 0;
  
  const [prevLat, prevLng] = positionsHistory[positionsHistory.length - 2];
  const [currLat, currLng] = positionsHistory[positionsHistory.length - 1];
  
  const y = Math.sin(currLng - prevLng) * Math.cos(currLat);
  const x = Math.cos(prevLat) * Math.sin(currLat) - 
           Math.sin(prevLat) * Math.cos(currLat) * Math.cos(currLng - prevLng);
  
  return (Math.atan2(y, x) * 180 / Math.PI + 360) % 360;
}

function startTracking() {
  if (navigator.geolocation) {
    watchId = navigator.geolocation.watchPosition(
      updatePosition,
      (error) => {
        console.error("Geolocation error:", error.message);
        alert("Location tracking failed.");
      },
      {
        enableHighAccuracy: true,
        timeout: 5000,
        maximumAge: 0
      }
    );
  } else {
    alert("Geolocation is not supported by this browser.");
  }
}

function stopTracking() {
  if (watchId) {
    navigator.geolocation.clearWatch(watchId);
    watchId = null;
  }
}

onMounted(() => {
  map.value = L.map(mapContainer.value).setView([51.505, -0.09], 13);
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
    <p>Latitude: {{ lat.toFixed(6) }}, Longitude: {{ lng.toFixed(6) }}</p>
    <button @click="startTracking" v-if="!watchId">Start Tracking</button>
    <button @click="stopTracking" v-else>Stop Tracking</button>
    <div ref="mapContainer" style="width: 100%; height: 500px;"></div>
  </div>
</template>