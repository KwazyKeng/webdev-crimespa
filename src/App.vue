<script setup>
import { reactive, ref, onMounted, computed } from 'vue'

// state
let crime_url = ref('');
let apiBaseUrl = ref('');
let dialog_err = ref(false);

let isLoading = ref(false);
let loadError = ref('');

let locationInput = ref('');
let geoError = ref('');

let crimes = ref([]);
let codesMap = ref({});
let neighborhoodsMap = ref({});
let visibleNeighborhoodIds = ref([]);

let filters = reactive({
  selectedIncidentGroups: [],
  selectedNeighborhoods: [],
  startDate: '',
  endDate: '',
  maxIncidents: 1000
});

let allIncidentTypes = computed(() => {
  let set = new Set()
  crimes.value.forEach(c => {
    if (c.incident_type) set.add(c.incident_type)
  })
  return Array.from(set).sort()
});

let allNeighborhoodNames = computed(() => {
  let set = new Set()
  crimes.value.forEach(c => {
    if (c.neighborhood_name) set.add(c.neighborhood_name)
  })
  return Array.from(set).sort()
});

//all incidents aggregated
//category is either violent, property, or other for background color for table
let incidentGroups = [
  {
    id: "homicide",
    label: "Homicide",
    category: "violent",
    codes: [100, 110, 120]
  },
  {
    id: "rape_criminal_sexual_conduct",
    label: "Rape / Criminal Sexual Conduct",
    category: "violent",
    codes: [210, 220]
  },
  {
    id: "robbery",
    label: "Robbery",
    category: "violent",
    codes: [
      300, 311, 312, 313, 314,
      321, 322, 323, 324,
      331, 333, 334,
      341, 342, 343, 344,
      351, 352, 353, 354,
      361, 362, 363, 364,
      371, 372, 373, 374,
      401
    ]
  },
  {
    id: "assault",
    label: "Assault",
    category: "violent",
    codes: [
      400, 410, 411, 412,
      420, 421, 422,
      430, 431, 432,
      440, 441, 442,
      450, 451, 452, 453, 454, 455, 456,
      863
    ]
  },
  {
    id: "burglary",
    label: "Burglary",
    category: "property",
    codes: [
      500,
      510, 511, 513, 515, 516,
      520, 521, 523, 525, 526,
      530, 531, 533, 535, 536,
      540, 541, 543, 545, 546,
      550, 551, 553, 555, 556,
      560, 561, 563, 565, 566
    ]
  },
  {
    id: "motor_vehicle_theft",
    label: "Motor Vehicle Theft",
    category: "property",
    codes: [
      700,
      710, 711, 712,
      720, 721, 722,
      730, 731, 732
    ]
  },
  {
    id: "theft_non_vehicle",
    label: "Theft (non-vehicle)",
    category: "property",
    codes: [
      600, 603,
      611, 612, 613,
      621, 622, 623,
      630, 631, 632, 633,
      641, 642, 643,
      651, 652, 653,
      661, 662, 663,
      671, 672, 673,
      681, 682, 683,
      691, 692, 693
    ]
  },
  {
    id: "property_damage_vandalism",
    label: "Property Damage / Vandalism",
    category: "property",
    codes: [
      601,
      1400, 1401,
      1410, 1415, 1416,
      1420, 1425, 1426,
      1430, 1435,
      1440
    ]
  },
  {
    id: "arson",
    label: "Arson",
    category: "violent",
    codes: [
      900, 901, 903, 905,
      911, 913, 915,
      921, 922, 923, 925,
      931, 933,
      941, 942,
      951,
      961, 963,
      971, 972, 973,
      981
    ]
  },
  {
    id: "narcotics",
    label: "Narcotics",
    category: "other",
    codes: [
      1800,
      1810, 1811, 1812, 1813, 1814, 1815,
      1820, 1822, 1823, 1824, 1825,
      1830, 1835,
      1840, 1841, 1842, 1843, 1844, 1845,
      1850, 1855,
      1860, 1865,
      1870,
      1880,
      1885
    ]
  },
  {
    id: "weapons",
    label: "Weapons",
    category: "violent",
    codes: [2619]
  },
  {
    id: "death_investigation",
    label: "Death Investigation",
    category: "other",
    codes: [3100]
  },
  {
    id: "proactive_community_policing",
    label: "Proactive / Community Policing",
    category: "other",
    codes: [9954, 9959, 9986]
  },
  {
    id: "other",
    label: "Other",
    category: "other",
    codes: [614]
  }
];

function getCategoryForCode(code) {
  for (let group of incidentGroups) {
    if (group.codes.includes(code)) {
      return group.category || 'other'
    }
  }
  return 'other'
};

let selectedCodes = computed(() => {
  if (!filters.selectedIncidentGroups || filters.selectedIncidentGroups.length === 0) {
    return []
  }

  return incidentGroups
    .filter(g => filters.selectedIncidentGroups.includes(g.id))
    .flatMap(g => g.codes)
});


let newIncident = reactive({
  case_number: '',
  date: '',
  time: '',
  code: '',
  incident: '',
  police_grid: '',
  neighborhood_number: '',
  block: ''
});
let newIncidentError = ref('');
let newIncidentSuccess = ref('');

let map = reactive({
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
});

let filteredCrimes = computed(() => {
  let list = crimes.value;

  if (selectedCodes.value.length > 0) {
    list = list.filter(c => selectedCodes.value.includes(c.code))
  }

  // neighborhood_name filter
  if (filters.selectedNeighborhoods.length > 0) {
    list = list.filter(c =>
      filters.selectedNeighborhoods.includes(c.neighborhood_name)
    )
  }

  // date range filter (dates are 'YYYY-MM-DD', so string compare is fine)
  if (filters.startDate) {
    list = list.filter(c => c.date >= filters.startDate)
  }
  if (filters.endDate) {
    list = list.filter(c => c.date <= filters.endDate)
  }

  return list
});

let visibleCrimes = computed(() => {
  let baseList = filteredCrimes.value;

  // filter to visible neighborhoods (based on map)
  if (visibleNeighborhoodIds.value && visibleNeighborhoodIds.value.length > 0) {
    baseList = baseList.filter(c =>
      visibleNeighborhoodIds.value.includes(c.neighborhood_number)
    )
  }

  // apply max incidents *after* all filters
  let max = Number(filters.maxIncidents)
  if (max > 0 && baseList.length > max) {
    return baseList.slice(0, max);
  }

  return baseList;
});

// Vue callback for once <template> HTML has been added to web page
onMounted(() => {
  // Create Leaflet map (set bounds and valied zoom levels)
  map.leaflet = L.map('leafletmap').setView([map.center.lat, map.center.lng], map.zoom);
  L.tileLayer('https://{s}.tile.openstreetmap.org/{z}/{x}/{y}.png', {
    attribution:
      '&copy; <a href="https://www.openstreetmap.org/copyright">OpenStreetMap</a> contributors',
    minZoom: 11,
    maxZoom: 18
  }).addTo(map.leaflet);
  map.leaflet.setMaxBounds([[44.883658, -93.217977], [45.008206, -92.993787]]);

  // Get boundaries for St. Paul neighborhoods
  let district_boundary = new L.geoJson();
  district_boundary.addTo(map.leaflet);
  fetch('data/StPaulDistrictCouncil.geojson')
    .then(response => response.json())
    .then(result => {
      result.features.forEach(value => {
        district_boundary.addData(value);
      })
    })
    .catch(error => {
      console.log('Error:', error);
    })

  // update neighborhoods and update search box
  map.leaflet.on('moveend', () => {
    let center = map.leaflet.getCenter();
    map.center.lat = center.lat;
    map.center.lng = center.lng;
    map.zoom = map.leaflet.getZoom();
    locationInput.value = `${center.lat.toFixed(5)}, ${center.lng.toFixed(5)}`;
    updateVisibleNeighborhoods();
  });

  // initial visible neighborhoods
  updateVisibleNeighborhoods();
})

// helper methods
function clampToBounds(lat, lng) {
  let minLat = map.bounds.se.lat;
  let maxLat = map.bounds.nw.lat;
  let minLng = map.bounds.nw.lng;
  let maxLng = map.bounds.se.lng;

  let clampedLat = Math.min(maxLat, Math.max(minLat, lat));
  let clampedLng = Math.min(maxLng, Math.max(minLng, lng));
  return { lat: clampedLat, lng: clampedLng };
}

function updateVisibleNeighborhoods() {
  if (!map.leaflet) return;

  let bounds = map.leaflet.getBounds();
  let north = bounds.getNorth();
  let south = bounds.getSouth();
  let west = bounds.getWest();
  let east = bounds.getEast();

  let visibleIds = map.neighborhood_markers
    .filter(n => {
      let [lat, lng] = n.location
      return lat <= north && lat >= south && lng >= west && lng <= east
    })
    .map(n => n.id);

  visibleNeighborhoodIds.value = visibleIds;
}

function updateNeighborhoodMarkers() {
  // count crimes per neighborhood
  let counts = {};
  filteredCrimes.value.forEach(c => {
  let id = c.neighborhood_number;
  counts[id] = (counts[id] || 0) + 1;
  });

  map.neighborhood_markers.forEach(n => {
    let id = n.id;
    let name = neighborhoodsMap.value[id] || `Neighborhood ${id}`;
    let count = counts[id] || 0;
    let popupHtml = `<strong>${name}</strong><br/>${count} crimes`;

    if (!n.marker) {
      let marker = L.marker(n.location).addTo(map.leaflet)
      n.marker = marker
      marker.bindPopup(popupHtml)
    } else {
      n.marker.setPopupContent(popupHtml)
    }
  })
}

// load crime data
function initializeCrimes() {
  try {
    if (!crime_url.value) {
      dialog_err.value = true;
      return;
    }
    dialog_err.value = false;
    apiBaseUrl.value = crime_url.value.replace(/\/+$/, '');

    isLoading.value = true;
    loadError.value = '';

    Promise.all([
      fetch(`${apiBaseUrl.value}/codes`),
      fetch(`${apiBaseUrl.value}/neighborhoods`),
      fetch(`${apiBaseUrl.value}/incidents?limit=1000`)
    ])
      .then(([codesRes, neighborhoodsRes, incidentsRes]) => {
        if (!codesRes.ok || !neighborhoodsRes.ok || !incidentsRes.ok) {
          throw new Error('Error fetching data from REST API');
        }
        return Promise.all([
          codesRes.json(),
          neighborhoodsRes.json(),
          incidentsRes.json()
        ]);
      })
      .then(([codesData, neighborhoodsData, incidentsData]) => {
        let cMap = {};
        codesData.forEach(c => {
          cMap[c.code] = c.type;
        });
        codesMap.value = cMap;

        let nMap = {};
        neighborhoodsData.forEach(n => {
          nMap[n.id] = n.name;
        });
        neighborhoodsMap.value = nMap;

        crimes.value = incidentsData.map(inc => ({
          ...inc,
          incident_type: cMap[inc.code] || String(inc.code),
          neighborhood_name: nMap[inc.neighborhood_number] || String(inc.neighborhood_number),
          category: getCategoryForCode(inc.code)
        }));

        updateVisibleNeighborhoods();
        updateNeighborhoodMarkers();
      })
      .catch(err => {
        console.error(err);
        loadError.value = 'Error loading data from REST API';
      })
      .finally(() => {
        isLoading.value = false;
      });
  } catch (err) {
    // this outer try/catch will almost never fire, but we'll keep it symmetrical
    console.error(err);
    loadError.value = 'Error loading data from REST API';
    isLoading.value = false;
  }
}

function refreshCrimes() {
  if (!apiBaseUrl.value) return;

  fetch(`${apiBaseUrl.value}/incidents?limit=1000`)
    .then(res => {
      if (!res.ok) {
        throw new Error('Error refreshing incidents');
      }
      return res.json();
    })
    .then(incidentsData => {
      crimes.value = incidentsData.map(inc => ({
        ...inc,
        incident_type: codesMap.value[inc.code] || String(inc.code),
        neighborhood_name:
          neighborhoodsMap.value[inc.neighborhood_number] ||
          String(inc.neighborhood_number),
          category: getCategoryForCode(inc.code)
      }));

      updateVisibleNeighborhoods();
      updateNeighborhoodMarkers();
    })
    .catch(err => {
      console.error(err);
    });
}

function applyFilters() {
  if (!apiBaseUrl.value) return;

  isLoading.value = true;
  loadError.value = '';

  let params = new URLSearchParams();
  params.set('limit', '1000');

  if (filters.startDate) {
    params.set('start_date', filters.startDate);
  }
  if (filters.endDate) {
    params.set('end_date', filters.endDate);
  }

  fetch(`${apiBaseUrl.value}/incidents?` + params.toString())
    .then(res => {
      if (!res.ok) {
        throw new Error('Error fetching filtered incidents');
      }
      return res.json();
    })
    .then(incidentsData => {
      crimes.value = incidentsData.map(inc => ({
        ...inc,
        incident_type: codesMap.value[inc.code] || String(inc.code),
        neighborhood_name:
          neighborhoodsMap.value[inc.neighborhood_number] ||
          String(inc.neighborhood_number),
        category: getCategoryForCode(inc.code)
      }));

      updateVisibleNeighborhoods();
      updateNeighborhoodMarkers();
    })
    .catch(err => {
      console.error(err);
      loadError.value = 'Error loading filtered data from REST API';
    })
    .finally(() => {
      isLoading.value = false;
    });
}

// geocoding
function goToLocation() {
  geoError.value = '';

  let raw = locationInput.value.trim();
  if (!raw) return;

  // check if user entered "lat, lng"
  let parts = raw.split(',');
  if (parts.length === 2) {
    let lat = Number(parts[0].trim());
    let lng = Number(parts[1].trim());

    if (!Number.isNaN(lat) && !Number.isNaN(lng)) {
      let clamped = clampToBounds(lat, lng);
      map.center.lat = clamped.lat;
      map.center.lng = clamped.lng;
      map.leaflet.setView([map.center.lat, map.center.lng]);
      return;
    }
  }

  // otherwise use nominatim
  let url = `https://nominatim.openstreetmap.org/search?format=json&limit=1&q=${encodeURIComponent(raw)}`;

  fetch(url, { headers: { Accept: 'application/json' } })
    .then(res => {
      if (!res.ok) {
        throw new Error('Nominatim search failed');
      }
      return res.json();
    })
    .then(results => {
      if (!results || results.length === 0) {
        geoError.value = 'Location not found';
        return;
      }

      let lat = parseFloat(results[0].lat);
      let lng = parseFloat(results[0].lon);

      let clamped = clampToBounds(lat, lng);
      map.center.lat = clamped.lat;
      map.center.lng = clamped.lng;

      map.leaflet.setView([map.center.lat, map.center.lng]);

      locationInput.value = `${map.center.lat.toFixed(5)}, ${map.center.lng.toFixed(5)}`;
    })
    .catch(err => {
      console.error(err);
      geoError.value = 'Error looking up location';
    });
}

// add new incidient
function submitNewIncident() {
  newIncidentError.value = '';
  newIncidentSuccess.value = '';

  let requiredFields = [
    'case_number',
    'date',
    'time',
    'code',
    'incident',
    'police_grid',
    'neighborhood_number',
    'block'
  ];

  // validation: must fill all fields
  for (let field of requiredFields) {
    if (!newIncident[field]) {
      newIncidentError.value = 'Please fill out all fields before submitting.';
      return;
    }
  }

  if (!apiBaseUrl.value) {
    newIncidentError.value = 'REST API URL is not set.';
    return;
  }

  let payload = {
    ...newIncident,
    code: Number(newIncident.code),
    police_grid: Number(newIncident.police_grid),
    neighborhood_number: Number(newIncident.neighborhood_number)
  };

  fetch(`${apiBaseUrl.value}/new-incident`, {
    method: 'PUT',
    headers: {
      'Content-Type': 'application/json'
    },
    body: JSON.stringify(payload)
  })
    .then(res => {
      if (!res.ok) {
        return res.text().then(text => {
          throw new Error(text || 'Server error adding incident.');
        });
      }
      return res.text(); // server might return a simple message
    })
    .then(() => {
      newIncidentSuccess.value = 'Incident added successfully.';

      // clear the form
      requiredFields.forEach(f => {
        newIncident[f] = '';
      });

      // refresh crimes
      return refreshCrimes();
    })
    .catch(err => {
      console.error(err);
      newIncidentError.value = err.message || 'Network error adding incident.';
    });
}

// dialog handler
function closeDialog() {
  let dialog = document.getElementById('rest-dialog');
  let url_input = document.getElementById('dialog-url');
  if (crime_url.value !== '' && url_input.checkValidity()) {
    dialog_err.value = false;
    dialog.close();
    initializeCrimes();
  } else {
    dialog_err.value = true;
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

        <!-- Filters -->
        <div class="filters-panel">
          <h4>Filters</h4>

          <label class="filter-label">
            Max incidents:
            <input
              type="number"
              min="1"
              max="1000"
              v-model.number="filters.maxIncidents"
              class="filter-input"
            />
          </label>

          <div class="filter-row">
            <label>Start date:</label>
            <input type="date" v-model="filters.startDate" class="filter-input" />
          </div>

          <div class="filter-row">
            <label>End date:</label>
            <input type="date" v-model="filters.endDate" class="filter-input" />
          </div>

          <div class="filter-group">
            <div class="filter-group-title">Incident type</div>

            <div class="filter-scroll">
              <label
                v-for="group in incidentGroups"
                :key="group.id"
                class="filter-checkbox"
              >
                <input
                  type="checkbox"
                  :value="group.id"
                  v-model="filters.selectedIncidentGroups"
                />
                {{ group.label }}
              </label>
            </div>
          </div>

          <div class="filter-group">
            <div class="filter-group-title">Neighborhood</div>
            <div class="filter-scroll">
              <label
                v-for="n in allNeighborhoodNames"
                :key="n"
                class="filter-checkbox"
              >
                <input
                  type="checkbox"
                  :value="n"
                  v-model="filters.selectedNeighborhoods"
                />
                {{ n }}
              </label>
            </div>
          </div>

          <button
            type="button"
            class="button tiny"
            @click="applyFilters"
          >
            Update
          </button>
        </div>

        <div class="crime-legend">
          <span class="legend-item">
            <span class="legend-swatch legend-violent"></span>
            Violent crime (crime against a person)
          </span>
          <span class="legend-item">
            <span class="legend-swatch legend-property"></span>
            Property crime (crime against property)
          </span>
          <span class="legend-item">
            <span class="legend-swatch legend-other"></span>
            Other crime
          </span>
        </div>

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
              <tr v-for="crime in visibleCrimes" :key="crime.case_number" :class="['crime-row', `crime-${crime.category || 'other'}`]">
                <td>{{ crime.date }}</td>
                <td>{{ crime.time }}</td>
                <td>{{ crime.neighborhood_name }}</td>
                <td>{{ crime.incident_type }}</td>
                <td>{{ crime.block }}</td>
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

.crime-row.crime-violent {
  background-color: #ffe0e0; /* light red/pink */
}

.crime-row.crime-property {
  background-color: #fff6cc; /* light yellow */
}

.crime-row.crime-other {
  background-color: #e6f0ff; /* light blue */
}

/* Legend styles */
.crime-legend {
  margin-bottom: 0.75rem;
  font-size: 0.9rem;
}

.crime-legend .legend-item {
  display: inline-flex;
  align-items: center;
  margin-right: 1rem;
  margin-bottom: 0.25rem;
}

.legend-swatch {
  display: inline-block;
  width: 14px;
  height: 14px;
  border-radius: 3px;
  margin-right: 0.4rem;
  border: 1px solid #ccc;
}

.legend-violent {
  background-color: #ffe0e0;
}

.legend-property {
  background-color: #fff6cc;
}

.legend-other {
  background-color: #e6f0ff;
}
</style>
