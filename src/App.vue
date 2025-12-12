<script setup>
import { reactive, ref, onMounted, computed } from 'vue'

// ======= STATE =======
const crime_url = ref('')
const apiBaseUrl = ref('')
const dialog_err = ref(false)

const isLoading = ref(false)
const loadError = ref('')

const locationInput = ref('')
const geoError = ref('')

const crimes = ref([]) // normalized incidents
const codesMap = ref({})
const neighborhoodsMap = ref({})
const visibleNeighborhoodIds = ref([])

const newIncident = reactive({
  case_number: '',
  date: '',
  time: '',
  code: '',
  incident: '',
  police_grid: '',
  neighborhood_number: '',
  block: ''
})
const newIncidentError = ref('')
const newIncidentSuccess = ref('')

const map = reactive({
  leaflet: null,
  center: {
    lat: 44.955139,
    lng: -93.102222,
    address: ''
  },
  zoom: 12,
  bounds: {
    nw: { lat: 45.008206, lng: -93.217977 },
    se: { lat: 44.883658, lng: -92.993787 }
  },
  incident_marker: null, 
  neighborhood_markers: [
    { id: 1, location: [44.942068, -93.020521], marker: null },
    { id: 2, location: [44.977413, -93.025156], marker: null },
    { id: 3, location: [44.931244, -93.079578], marker: null },
    { id: 4, location: [44.956192, -93.060189], marker: null },
    { id: 5, location: [44.978883, -93.068163], marker: null },
    { id: 6, location: [44.975766, -93.113887], marker: null },
    { id: 7, location: [44.959639, -93.121271], marker: null },
    { id: 8, location: [44.9477, -93.128505], marker: null },
    { id: 9, location: [44.930276, -93.119911], marker: null },
    { id: 10, location: [44.982752, -93.14791], marker: null },
    { id: 11, location: [44.963631, -93.167548], marker: null },
    { id: 12, location: [44.973971, -93.197965], marker: null },
    { id: 13, location: [44.949043, -93.178261], marker: null },
    { id: 14, location: [44.934848, -93.176736], marker: null },
    { id: 15, location: [44.913106, -93.170779], marker: null },
    { id: 16, location: [44.937705, -93.136997], marker: null },
    { id: 17, location: [44.949203, -93.093739], marker: null }
  ]
})

// Derived list of crimes just in neighborhoods visible on the map
const visibleCrimes = computed(() => {
  if (!visibleNeighborhoodIds.value || visibleNeighborhoodIds.value.length === 0) {
    return crimes.value
  }
  return crimes.value.filter(c =>
    visibleNeighborhoodIds.value.includes(c.neighborhood_number)
  )
})

// ======= MAP SETUP =======
onMounted(() => {
  // Create Leaflet map
  map.leaflet = L.map('leafletmap').setView([map.center.lat, map.center.lng], map.zoom)
  L.tileLayer('https://{s}.tile.openstreetmap.org/{z}/{x}/{y}.png', {
    attribution:
      '&copy; <a href="https://www.openstreetmap.org/copyright">OpenStreetMap</a> contributors',
    minZoom: 11,
    maxZoom: 18
  }).addTo(map.leaflet)

  // Limit panning to St. Paul bounding box
  map.leaflet.setMaxBounds([
    [map.bounds.se.lat, map.bounds.nw.lng],
    [map.bounds.nw.lat, map.bounds.se.lng]
  ])

  // Show St. Paul district boundaries
  const district_boundary = new L.geoJson()
  district_boundary.addTo(map.leaflet)
  fetch('data/StPaulDistrictCouncil.geojson')
    .then(response => response.json())
    .then(result => {
      result.features.forEach(value => {
        district_boundary.addData(value)
      })
    })
    .catch(error => {
      console.log('Error:', error)
    })

  // Update visible neighborhoods and input whenever map stops moving
  map.leaflet.on('moveend', () => {
    const center = map.leaflet.getCenter()
    map.center.lat = center.lat
    map.center.lng = center.lng
    locationInput.value = `${center.lat.toFixed(5)}, ${center.lng.toFixed(5)}`
    updateVisibleNeighborhoods()
  })

  // Initial visible neighborhoods
  updateVisibleNeighborhoods()
})

// ======= HELPERS =======
function clampToBounds(lat, lng) {
  const minLat = map.bounds.se.lat
  const maxLat = map.bounds.nw.lat
  const minLng = map.bounds.nw.lng
  const maxLng = map.bounds.se.lng

  const clampedLat = Math.min(maxLat, Math.max(minLat, lat))
  const clampedLng = Math.min(maxLng, Math.max(minLng, lng))
  return { lat: clampedLat, lng: clampedLng }
}

function address(block) {
  if (!block) return block;

  let parts = block.split(' ')
  let first = parts[0].split('')
  
  for (let i = 0; i < first.length; i++){
    if(first[i] === "X"){
      first[i] = "0";
    }
  }

  parts[0] = first.join('');

  return parts.join(' ')
}

function updateVisibleNeighborhoods() {
  if (!map.leaflet) return

  const bounds = map.leaflet.getBounds()
  const north = bounds.getNorth()
  const south = bounds.getSouth()
  const west = bounds.getWest()
  const east = bounds.getEast()

  const visibleIds = map.neighborhood_markers
    .filter(n => {
      const [lat, lng] = n.location
      return lat <= north && lat >= south && lng >= west && lng <= east
    })
    .map(n => n.id)

  visibleNeighborhoodIds.value = visibleIds
}

function updateNeighborhoodMarkers() {
  // Count crimes per neighborhood
  const counts = {}
  crimes.value.forEach(c => {
    const id = c.neighborhood_number
    counts[id] = (counts[id] || 0) + 1
  })

  map.neighborhood_markers.forEach(n => {
    const id = n.id
    const name = neighborhoodsMap.value[id] || `Neighborhood ${id}`
    const count = counts[id] || 0
    const popupHtml = `<strong>${name}</strong><br/>${count} crimes`

    if (!n.marker) {
      const marker = L.marker(n.location).addTo(map.leaflet)
      n.marker = marker
      marker.bindPopup(popupHtml)
    } else {
      n.marker.setPopupContent(popupHtml)
    }
  })
}

// ======= CRIME DATA LOADING =======
async function initializeCrimes() {
  try {
    if (!crime_url.value) {
      dialog_err.value = true
      return
    }
    dialog_err.value = false
    apiBaseUrl.value = crime_url.value.replace(/\/+$/, '')

    isLoading.value = true
    loadError.value = ''

    const [codesRes, neighborhoodsRes, incidentsRes] = await Promise.all([
      fetch(`${apiBaseUrl.value}/codes`),
      fetch(`${apiBaseUrl.value}/neighborhoods`),
      fetch(`${apiBaseUrl.value}/incidents?limit=1000`)
    ])

    if (!codesRes.ok || !neighborhoodsRes.ok || !incidentsRes.ok) {
      throw new Error('Error fetching data from REST API')
    }

    const codesData = await codesRes.json()
    const neighborhoodsData = await neighborhoodsRes.json()
    const incidentsData = await incidentsRes.json()

    const cMap = {}
    codesData.forEach(c => {
      cMap[c.code] = c.type
    })
    codesMap.value = cMap

    const nMap = {}
    neighborhoodsData.forEach(n => {
      nMap[n.id] = n.name
    })
    neighborhoodsMap.value = nMap

    crimes.value = incidentsData.map(inc => ({
      ...inc,
      incident_type: cMap[inc.code] || String(inc.code),
      neighborhood_name: nMap[inc.neighborhood_number] || String(inc.neighborhood_number)
    }))

    updateVisibleNeighborhoods()
    updateNeighborhoodMarkers()
  } catch (err) {
    console.error(err)
    loadError.value = 'Error loading data from REST API'
  } finally {
    isLoading.value = false
  }
}

async function refreshCrimes() {
  try {
    if (!apiBaseUrl.value) return
    const res = await fetch(`${apiBaseUrl.value}/incidents?limit=1000`)
    if (!res.ok) throw new Error('Error refreshing incidents')
    const incidentsData = await res.json()
    crimes.value = incidentsData.map(inc => ({
      ...inc,
      incident_type: codesMap.value[inc.code] || String(inc.code),
      neighborhood_name:
        neighborhoodsMap.value[inc.neighborhood_number] || String(inc.neighborhood_number)
    }))
    updateVisibleNeighborhoods()
    updateNeighborhoodMarkers()
  } catch (err) {
    console.error(err)
  }
}

// ======= GEOCODING (NOMINATIM) =======
async function goToLocation() {
  geoError.value = ''

  const raw = locationInput.value.trim()
  if (!raw) return

  // 1) Try to parse "lat,lng"
  const parts = raw.split(',')
  if (parts.length === 2) {
    const lat = Number(parts[0].trim())
    const lng = Number(parts[1].trim())
    if (!Number.isNaN(lat) && !Number.isNaN(lng)) {
      const clamped = clampToBounds(lat, lng)
      map.center.lat = clamped.lat
      map.center.lng = clamped.lng
      map.leaflet.setView([map.center.lat, map.center.lng], map.zoom)
      return
    }
  }

  // 2) Otherwise, treat as address and use Nominatim
  try {
    const url = `https://nominatim.openstreetmap.org/search?format=json&limit=1&q=${encodeURIComponent(
      raw
    )}`
    const res = await fetch(url, {
      headers: {
        Accept: 'application/json'
      }
    })
    if (!res.ok) throw new Error('Nominatim search failed')
    const results = await res.json()
    if (!results || results.length === 0) {
      geoError.value = 'Location not found'
      return
    }
    let lat = parseFloat(results[0].lat)
    let lng = parseFloat(results[0].lon)
    const clamped = clampToBounds(lat, lng)
    map.center.lat = clamped.lat
    map.center.lng = clamped.lng
    map.leaflet.setView([map.center.lat, map.center.lng], map.zoom)
    locationInput.value = `${map.center.lat.toFixed(5)}, ${map.center.lng.toFixed(5)}`
  } catch (err) {
    console.error(err)
    geoError.value = 'Error looking up location'
  }
}

function focusCrime(crime) {
  if (!crime || !map.leaflet) return;

  let query = address(crime.block);
  let url = `https://nominatim.openstreetmap.org/search?format=json&limit=1&q=${query}`;

  fetch(url, { 
      headers: { Accept: 'application/json' } })
    .then(res => {
      return res.json();
    })

    .then(results => {

      let lat = parseFloat(results[0].lat);
      let lng = parseFloat(results[0].lon);

      let clamped = clampToBounds(lat, lng);
      lat = clamped.lat;
      lng = clamped.lng;

      map.center.lat = lat;
      map.center.lng = lng;

      map.leaflet.setView([lat, lng], 16);

      if (!map.incident_marker) {
        map.incident_marker = L.circleMarker([lat, lng], {
          radius: 8,
          color: 'red',
          fillColor: 'red',
          fillOpacity: 0.9
        }).addTo(map.leaflet);
      } else {
        map.incident_marker.setLatLng([lat, lng]);
      }
      let popupHtml = `
        <div>
          <strong>${crime.incident_type}</strong><br/>
          ${crime.date} ${crime.time}<br/>
          ${address(crime.block)}<br/>
        </div>
      `;

      map.incident_marker.bindPopup(popupHtml).openPopup();
      map.incident_marker.off("popupopen");
      map.incident_marker.on("popupopen", () => {
        let btn = document.getElementById(`crime-delete-${crime.case_number}`);
        if (btn) {
          btn.onclick = function () {
            deleteIncident(crime.case_number);
          };
        }
      });
    })

    .catch(err => {
      console.error("500 Error:", err);
    });
}


// ======= NEW INCIDENT FORM =======
async function submitNewIncident() {
  newIncidentError.value = ''
  newIncidentSuccess.value = ''

  const requiredFields = [
    'case_number',
    'date',
    'time',
    'code',
    'incident',
    'police_grid',
    'neighborhood_number',
    'block'
  ]
  for (const field of requiredFields) {
    if (!newIncident[field]) {
      newIncidentError.value = 'Please fill out all fields before submitting.'
      return
    }
  }

  if (!apiBaseUrl.value) {
    newIncidentError.value = 'REST API URL is not set.'
    return
  }

  try {
    const res = await fetch(`${apiBaseUrl.value}/new-incident`, {
      method: 'PUT',
      headers: {
        'Content-Type': 'application/json'
      },
      body: JSON.stringify({
        ...newIncident,
        code: Number(newIncident.code),
        police_grid: Number(newIncident.police_grid),
        neighborhood_number: Number(newIncident.neighborhood_number)
      })
    })

    if (!res.ok) {
      const text = await res.text()
      newIncidentError.value = text || 'Server error adding incident.'
      return
    }

    newIncidentSuccess.value = 'Incident added successfully.'
    // Clear form
    requiredFields.forEach(f => {
      newIncident[f] = ''
    })

    // Refresh crimes so new one shows up
    await refreshCrimes()
  } catch (err) {
    console.error(err)
    newIncidentError.value = 'Network error adding incident.'
  }
}
//delete

function deleteIncident(caseNumber) {

  fetch(`${apiBaseUrl.value}/remove-incident`, {
    method: "DELETE",
    headers: {
      "Content-Type": "application/json"
    },
    body: JSON.stringify({ case_number: caseNumber })
  })
  .then(res => res.text())
  .then(text => {

    console.log(text);

    let oldList = crimes.value;
    let newList = [];

    for (let i = 0; i < oldList.length; i++) {
      let crime = oldList[i];
      if (crime.case_number !== caseNumber) {
        newList.push(crime);
      }
    }
    crimes.value = newList;
  }).catch(err => {
    console.error("500 Error:", err);
  });
}

// ======= DIALOG HANDLER =======
function closeDialog() {
  const dialog = document.getElementById('rest-dialog')
  const url_input = document.getElementById('dialog-url')
  if (crime_url.value !== '' && url_input.checkValidity()) {
    dialog_err.value = false
    dialog.close()
    initializeCrimes()
  } else {
    dialog_err.value = true
  }
}
</script>

<template>
  <dialog id="rest-dialog" open>
    <h1 class="dialog-header">St. Paul Crime REST API</h1>
    <label class="dialog-label">URL: </label>
    <input
      id="dialog-url"
      class="dialog-input"
      type="url"
      v-model="crime_url"
      placeholder="http://localhost:8000"
    />
    <p class="dialog-error" v-if="dialog_err">Error: must enter valid URL</p>
    <br />
    <button class="button" type="button" @click="closeDialog">OK</button>
  </dialog>

  <div class="grid-container main-container">
    <div class="grid-x grid-padding-x">
      <div class="cell medium-8 small-12">
        <div class="map-controls">
          <label>Location:</label>
          <input
            type="text"
            v-model="locationInput"
            placeholder="Address or 'lat,lng'"
            class="map-input"
          />
          <button class="button tiny" type="button" @click="goToLocation">Go</button>
        </div>
        <p v-if="geoError" class="geo-error">{{ geoError }}</p>
        <div id="leafletmap" class="map-view"></div>
      </div>

      <div class="cell medium-4 small-12">
        <h3>Crimes in visible neighborhoods</h3>
        <p v-if="isLoading">Loading crimes...</p>
        <p v-if="loadError" class="geo-error">{{ loadError }}</p>

        <div class="crime-table-wrapper" v-if="visibleCrimes.length > 0">
          <table class="crime-table">
            <thead>
              <tr>
                <th>Date</th>
                <th>Time</th>
                <th>Neighborhood</th>
                <th>Incident</th>
                <th>Block</th>
              </tr>
            </thead>
            <tbody>
              <tr v-for="crime in visibleCrimes" :key="crime.case_number">
                <td>{{ crime.date }}</td>
                <td>{{ crime.time }}</td>
                <td>{{ crime.neighborhood_name }}</td>
                <td>{{ crime.incident_type }}</td>
                <td>{{ address(crime.block) }}</td>
                <td><button class = "delete-button" type="button" @click="deleteIncident(crime.case_number)">Delete</button></td>
                <td><button class = "incident" type="button" @click="focusCrime(crime)">Incident</button></td>
              </tr>
            </tbody>
          </table>
        </div>
        <p v-else>No crimes to show for the current map view.</p>
      </div>
    </div>

    <div class="grid-x grid-padding-x">
      <div class="cell small-12">
        <h3>Add New Incident</h3>
        <form @submit.prevent="submitNewIncident" class="incident-form">
          <div class="grid-x grid-padding-x">
            <div class="cell medium-3 small-6">
              <label>
                Case #
                <input v-model="newIncident.case_number" type="text" />
              </label>
            </div>
            <div class="cell medium-3 small-6">
              <label>
                Date (YYYY-MM-DD)
                <input v-model="newIncident.date" type="text" />
              </label>
            </div>
            <div class="cell medium-3 small-6">
              <label>
                Time (HH:MM:SS)
                <input v-model="newIncident.time" type="text" />
              </label>
            </div>
            <div class="cell medium-3 small-6">
              <label>
                Code
                <input v-model="newIncident.code" type="number" />
              </label>
            </div>
          </div>

          <div class="grid-x grid-padding-x">
            <div class="cell medium-4 small-12">
              <label>
                Incident
                <input v-model="newIncident.incident" type="text" />
              </label>
            </div>
            <div class="cell medium-2 small-6">
              <label>
                Police Grid
                <input v-model="newIncident.police_grid" type="number" />
              </label>
            </div>
            <div class="cell medium-2 small-6">
              <label>
                Neighborhood #
                <input v-model="newIncident.neighborhood_number" type="number" />
              </label>
            </div>
            <div class="cell medium-4 small-12">
              <label>
                Block
                <input v-model="newIncident.block" type="text" />
              </label>
            </div>
          </div>

          <button class="button small" type="submit">Submit Incident</button>
          <p v-if="newIncidentError" class="geo-error">{{ newIncidentError }}</p>
          <p v-if="newIncidentSuccess" class="success-text">{{ newIncidentSuccess }}</p>
        </form>
      </div>
    </div>
  </div>
</template>

<style scoped>
#rest-dialog {
  width: 20rem;
  margin-top: 1rem;
  z-index: 1000;
}

.main-container {
  margin-top: 1rem;
}

.map-controls {
  margin-bottom: 0.5rem;
  display: flex;
  gap: 0.5rem;
  align-items: center;
}

.map-input {
  flex: 1;
}

.map-view {
  height: 500px;
  border: 1px solid #ccc;
}

.crime-table-wrapper {
  max-height: 500px;
  overflow-y: auto;
  border: 1px solid #ccc;
}

.crime-table {
  width: 100%;
  font-size: 0.8rem;
}

.crime-table th,
.crime-table td {
  padding: 0.25rem 0.5rem;
}

.geo-error {
  color: #d32323;
  font-size: 0.9rem;
}

.success-text {
  color: #2a7a2a;
  font-size: 0.9rem;
}

.dialog-header {
  font-size: 1.5rem;
}

.dialog-label {
  font-size: 1rem;
}

.dialog-input {
  font-size: 1rem;
  width: 100%;
}

.dialog-error {
  font-size: 1rem;
  color: #d32323;
}

.delete-button{
  background-color: rgb(211, 35, 35);
  color: rgb(255, 255, 255);
  padding: 10px 10px;
  cursor: pointer;
}

.incident{
  background-color: rgb(0, 170, 249);
  color: rgb(255, 255, 255);
  padding: 10px 10px;
  cursor: pointer;
}
</style>
