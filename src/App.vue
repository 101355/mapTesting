<script setup>
import { ref, onMounted } from "vue";
import L from "leaflet";
import "leaflet/dist/leaflet.css";

// Fix for missing Leaflet marker icons in production
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

function getLocation() {
  if (navigator.geolocation) {
    navigator.geolocation.getCurrentPosition(
      (position) => {
        lat.value = position.coords.latitude;
        lng.value = position.coords.longitude;
        const currentPosition = [lat.value, lng.value];
        console.log("Your current location:", currentPosition);

        // Center the map
        map.value.setView(currentPosition, 15);

        // Add or update marker
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
        alert("Failed to get your location. Defaulting to London.");
        // Fallback to London coordinates
        map.value.setView([51.505, -0.09], 13);
      },
      {
        enableHighAccuracy: true,
        timeout: 10000,
        maximumAge: 0,
      }
    );
  } else {
    alert("Geolocation is not supported by this browser.");
  }
}

onMounted(() => {
  map.value = L.map(mapContainer.value).setView([51.505, -0.09], 13);
  L.tileLayer("https://tile.openstreetmap.org/{z}/{x}/{y}.png", {
    maxZoom: 19,
    attribution:
      '&copy; <a href="http://www.openstreetmap.org/copyright">OpenStreetMap</a>',
  }).addTo(map.value);

  getLocation();
});
</script>

<template>
  <p>Latitude: {{ lat }}, Longitude: {{ lng }}</p>
  <div ref="mapContainer" style="width: 100%; height: 400px;"></div>
</template>