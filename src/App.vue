<script setup>
import { ref, onMounted } from "vue";
import L from "leaflet";
import "leaflet/dist/leaflet.css";

const lat = ref(0);
const lng = ref(0);
const map = ref();
const mapContainer = ref();
let userMarker = null;

function initMap() {
  // Initialize map
  map.value = L.map(mapContainer.value).setView([51.505, -0.09], 13);

  // Add OpenStreetMap tile layer
  L.tileLayer("https://tile.openstreetmap.org/{z}/{x}/{y}.png", {
    maxZoom: 19,
    attribution:
      '&copy; <a href="http://www.openstreetmap.org/copyright">OpenStreetMap</a>',
  }).addTo(map.value);
}

function trackLocation() {
  if (navigator.geolocation) {
    navigator.geolocation.watchPosition(
      (position) => {
        lat.value = position.coords.latitude;
        lng.value = position.coords.longitude;

        const currentPosition = [lat.value, lng.value];

        console.log("User location:", currentPosition); // ✅ Log the position

        map.value.setView(currentPosition, 15);

        if (!userMarker) {
          userMarker = L.marker(currentPosition, { draggable: true })
            .addTo(map.value)
            .bindPopup("You are here.")
            .openPopup();

          userMarker.on("dragend", (event) => {
            const { lat: newLat, lng: newLng } = event.target.getLatLng();
            lat.value = newLat;
            lng.value = newLng;
            console.log("Dragged to:", newLat, newLng);
          });
        } else {
          userMarker.setLatLng(currentPosition);
        }
      },
      (error) => {
        console.error("Geolocation error:", error.message);
      },
      {
        enableHighAccuracy: true,
      }
    );
  } else {
    alert("Geolocation is not supported by this browser.");
  }
}

onMounted(() => {
  initMap();
  trackLocation(); // ✅ Automatically request GPS access on load
});
</script>

<template>
  <p>Latitude: {{ lat }}, Longitude: {{ lng }}</p>
  <div ref="mapContainer" style="width: 400px; height: 400px;"></div>
</template>