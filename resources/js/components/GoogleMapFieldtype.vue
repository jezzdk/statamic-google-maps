<script setup>
import { ref, computed, watch, onMounted } from 'vue'
import { Fieldtype } from '@statamic/cms'

const emit = defineEmits(Fieldtype.emits)
const props = defineProps(Fieldtype.props)
const { expose, update } = Fieldtype.use(emit, props)
defineExpose(expose)

// Reactive state
const lat = ref(null)
const lng = ref(null)
const markerLat = ref(null)
const markerLng = ref(null)
const zoom = ref(null)
const type = ref(null)
const style = ref(null)
const showControls = ref(false)
const map = ref(null)
const marker = ref(null)
const hasMarker = ref(false)
const stylesExpanded = ref(false)
const geocoder = ref(null)
const location = ref(null)
const mapRef = ref(null)

// Computed properties
const hasGeocoder = computed(() => props.config.geocoder)

const canReset = computed(() => props.meta.defaultLat && props.meta.defaultLng)

const mapHasChanged = computed(() => {
    return lat.value != props.meta.defaultLat
        || lng.value != props.meta.defaultLng
        || zoom.value != props.config.initial_zoom
        || type.value != props.config.initial_type
})

const hasGeolocation = computed(() => navigator.geolocation || false)

// Watchers
watch([lat, lng, markerLat, markerLng, zoom, type, style, showControls], () => {
    saveLocation()
})

// Methods
const addMapListeners = () => {
    if (props.config.markers) {
        map.value.addListener('click', (e) => {
            addMarker(e.latLng)

            markerLat.value = e.latLng.lat()
            markerLng.value = e.latLng.lng()
        })
    }

    map.value.addListener('center_changed', () => {
        lat.value = map.value.getCenter().lat()
        lng.value = map.value.getCenter().lng()
    })

    map.value.addListener('zoom_changed', () => {
        zoom.value = map.value.getZoom()
    })

    map.value.addListener('maptypeid_changed', () => {
        type.value = map.value.getMapTypeId()
    })
}

const addMarker = (position) => {
    marker.value.setMap(map.value)
    marker.value.setPosition(position)
    marker.value.setVisible(true)
    markerLat.value = position.lat()
    markerLng.value = position.lng()
    hasMarker.value = true
}

const removeMarker = () => {
    marker.value.setMap(null)
    marker.value.setPosition(null)
    marker.value.setVisible(false)
    markerLat.value = null
    markerLng.value = null
    hasMarker.value = false
}

const resetMap = () => {
    map.value.setCenter({
        lat: Number.parseFloat(props.meta.defaultLat),
        lng: Number.parseFloat(props.meta.defaultLng),
    })

    map.value.setZoom(Number(props.meta.defaultZoom) || 16)
    map.value.setMapTypeId(Number(props.meta.defaultType) || 'roadmap')

    removeMarker()
}

const saveLocation = () => {
    update({
        lat: lat.value,
        lng: lng.value,
        markerLat: markerLat.value,
        markerLng: markerLng.value,
        zoom: zoom.value,
        type: type.value,
        style: style.value,
        showControls: showControls.value,
    })
}

const findPosition = () => {
    geocoder.value.geocode({
        address: location.value
    }).then((response) => {
        if (response.results.length > 0) {
            console.log('Location found')

            let position = response.results[0].geometry.location
            map.value.setCenter(position)

            addMarker(position)
        } else {
            console.error('Location not found')
        }
    }).catch((error) => {
        console.error(error.message)
    })
}

const findUserPosition = () => {
    if (!navigator.geolocation) {
        return;
    }

    navigator.geolocation.getCurrentPosition((position) => {
        const pos = {
            lat: position.coords.latitude,
            lng: position.coords.longitude,
        };

        map.value.setCenter(pos)
    }, () => {
        console.debug('Error getting user position')
    })
}

const addUserPositionButton = () => {
    const locationButton = document.createElement("button")
    locationButton.innerHTML = `
<?xml version="1.0" encoding="iso-8859-1"?>
<svg version="1.1" id="Capa_1" xmlns="http://www.w3.org/2000/svg" xmlns:xlink="http://www.w3.org/1999/xlink" x="0px" y="0px" viewBox="0 0 512 512" style="enable-background:new 0 0 512 512; display: block;" xml:space="preserve">
<g><g><path d="M256,0c-48.551,0-95.818,13.675-136.693,39.545l16.044,25.35C171.419,42.066,213.139,30,256,30 c42.861,0,84.581,12.066,120.648,34.895l16.044-25.35C351.818,13.675,304.551,0,256,0z"/></g></g>
<g><g><path d="M376.649,447.105C340.581,469.934,298.861,482,256,482c-42.861,0-84.581-12.066-120.648-34.895l-16.044,25.35 C160.182,498.325,207.449,512,256,512c48.551,0,95.818-13.675,136.693-39.545L376.649,447.105z"/></g></g>
<g><g><path d="M472.455,119.307l-25.35,16.044C469.934,171.419,482,213.139,482,256c0,42.861-12.066,84.581-34.895,120.648l25.35,16.044 C498.325,351.818,512,304.551,512,256C512,207.449,498.325,160.182,472.455,119.307z"/></g></g>
<g><g><path d="M64.895,135.352l-25.35-16.045C13.675,160.182,0,207.449,0,256c0,48.551,13.675,95.818,39.545,136.693l25.35-16.044 C42.066,340.581,30,298.861,30,256C30,213.139,42.066,171.419,64.895,135.352z"/></g></g>
<g><g><path d="M256,204c-28.673,0-52,23.327-52,52c0,28.673,23.327,52,52,52c28.673,0,52-23.327,52-52C308,227.327,284.673,204,256,204z M256,278c-12.131,0-22-9.869-22-22s9.869-22,22-22c12.131,0,22,9.869,22,22S268.131,278,256,278z"/></g></g>
</svg>`
    locationButton.style.margin = '10px'
    locationButton.style.padding = '8px'
    locationButton.style.width = '40px'
    locationButton.style.boxShadow = 'rgba(0, 0, 0, 0.3) 0px 1px 4px -1px'
    locationButton.style.backgroundColor = '#FFFFFF'
    locationButton.addEventListener("mouseover", () => locationButton.style.backgroundColor = '#EFEFEF')
    locationButton.addEventListener("mouseout", () => locationButton.style.backgroundColor = '#FFFFFF')
    locationButton.addEventListener("click", () => findUserPosition())
    map.value.controls[google.maps.ControlPosition.RIGHT_TOP].push(locationButton)
}

onMounted(() => {
    lat.value = props.value.lat || props.meta.defaultLat
    lng.value = props.value.lng || props.meta.defaultLng
    markerLat.value = props.value.markerLat
    markerLng.value = props.value.markerLng
    zoom.value = props.value.zoom || props.config.initial_zoom || 16
    type.value = props.value.type || props.config.initial_type || 'roadmap'
    style.value = props.value.style
    stylesExpanded.value = Boolean(props.value.style)
    showControls.value = props.value.showControls

    map.value = new google.maps.Map(mapRef.value, {
        zoom: Number(zoom.value),
        center: {
            lat: Number.parseFloat(lat.value),
            lng: Number.parseFloat(lng.value),
        },
        mapTypeId: type.value,
        mapTypeControl: props.config.maptypes,
        mapTypeControlOptions: {
            mapTypeIds: [
                google.maps.MapTypeId.ROADMAP,
                google.maps.MapTypeId.SATELLITE,
                google.maps.MapTypeId.TERRAIN,
                google.maps.MapTypeId.HYBRID,
            ]
        },
        streetViewControl: false,
    })

    if (props.config.markers) {
        marker.value = new google.maps.Marker({
            clickable: false,
            draggable: true,
        })

        marker.value.addListener('dragend', () => {
            markerLat.value = marker.value.getPosition().lat()
            markerLng.value = marker.value.getPosition().lng()
        })

        if (markerLat.value && markerLng.value) {
            addMarker(
                new google.maps.LatLng(Number.parseFloat(markerLat.value), Number.parseFloat(markerLng.value))
            )
        }
    }

    if (props.config.geocoder) {
        geocoder.value = new google.maps.Geocoder()
    }

    addMapListeners()

    addUserPositionButton()
})
</script>

<template>
    <div class="space-y-2">
        <ui-input v-if="hasGeocoder" type="text" v-model="location" @keyup.enter="findPosition" placeholder="Search location" />
        <div class="w-full h-96 rounded google-maps-container" ref="mapRef"></div>
        <div class="flex justify-between">
            <div>
                <ui-button v-if="hasMarker" variant="danger" size="xs" text="Remove marker" @click.prevent="removeMarker()" />
                <ui-button v-else-if="config.markers" variant="primary" size="xs" text="Add marker" @click.prevent="addMarker(map.getCenter())" />
            </div>
            <div>
                <ui-button v-if="canReset && mapHasChanged" variant="ghost" size="xs" text="Reset map" @click.prevent="resetMap" />
            </div>
        </div>
        <div class="flex items-center gap-8 text-sm">
            <div class="flex items-center gap-2"><ui-switch v-model="showControls" size="sm" /> Show map controls</div>
            <div v-if="meta.pro && !config.hideStyles" class="flex items-center gap-2"><ui-switch v-model="stylesExpanded" size="sm" /> Use custom styling</div>
        </div>
        <div v-if="meta.pro && !config.hideStyles && stylesExpanded">
            <ui-textarea v-model="style" placeholder="Paste in the styles as JSON"></ui-textarea>
            <div class="text-gray-600 text-xs">Need help? Check out the <a href="https://mapstyle.withgoogle.com/" target="_blank">style tool</a> or <a href="https://snazzymaps.com/" target="_blank">Snazzy Maps</a>.</div>
        </div>
    </div>
</template>
