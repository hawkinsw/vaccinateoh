<template>
    <div class="w-full h-full flex">

        <div id="location-sidebar" class="h-screen w-72 flex-none overflow-y-auto">
            <h1 class="text-center text-2xl">Vaccinate OH</h1>
            <ul class="location-list">
                <li class="location p-2 py-4 flex cursor-pointer" v-for="loc in (search_locations)" @mouseover="showLocationMarker(loc)">
                    <div class="location-details flex-grow">
                        <h3 class="font-bold">{{ loc.name }}</h3>
                        <div class="address text-sm text-gray-700" v-html="addressHtml(loc.address)"></div>
                        <div class="phone text-sm text-gray-500"><a :href="'tel:' + loc.phone">{{ loc.phone }}</a></div>
                        <div class="appt-link"><a :href="loc.bookinglink" target="_blank" v-if="loc.bookinglink">Search Appointments</a></div>
                    </div>
                    <div class="location-distance w-8 px-1 pt-4 text-center flex-none">
                        <div class="text-xs text-gray-500" v-if="loc.distance">{{ round(loc.distance) }} mi</div>
                        <div class="location-icon">&raquo; </div>
                    </div>
                </li>
            </ul>
        </div>
        <div id="map" class="h-screen flex-grow"></div>

        <div class="hidden fixed top-0 right-0 px-6 py-4 sm:block">
            <inertia-link v-if="$page.props.user" href="/dashboard" class="text-sm text-gray-700 underline">
                Dashboard
            </inertia-link>

            <template v-else>
                <inertia-link :href="route('login')" class="text-sm text-gray-700 underline">
                    Log in
                </inertia-link>

                <inertia-link :href="route('register')" class="ml-4 text-sm text-gray-700 underline">
                    Register
                </inertia-link>
            </template>
        </div>

        <div class="w-96 absolute top-0 inset-x-0 opacity-80 ml-1/2 z-50 mx-auto sm:px-6 lg:px-8">
            <div class="p-8">
                <form class="bg-white flex items-center rounded-full shadow-xl" @submit.prevent="searchLocations(null)">
                    <input class="rounded-l-full w-full py-4 px-6 text-gray-700 leading-tight focus:outline-none active:outline-none border-0" id="search" type="text" placeholder="ZIP Code Search" v-model="search_q">

                    <div class="p-4">
                        <button class="bg-blue-500 text-white rounded-full p-2 hover:bg-blue-400 focus:outline-none w-12 h-12 flex items-center justify-center" type="submit">
                            <svg width="24" height="24" xmlns="http://www.w3.org/2000/svg" fill-rule="evenodd" clip-rule="evenodd" class="fill-current text-white"><path d="M15.853 16.56c-1.683 1.517-3.911 2.44-6.353 2.44-5.243 0-9.5-4.257-9.5-9.5s4.257-9.5 9.5-9.5 9.5 4.257 9.5 9.5c0 2.442-.923 4.67-2.44 6.353l7.44 7.44-.707.707-7.44-7.44zm-6.353-15.56c4.691 0 8.5 3.809 8.5 8.5s-3.809 8.5-8.5 8.5-8.5-3.809-8.5-8.5 3.809-8.5 8.5-8.5z"/></svg>
                        </button>
                    </div>
                </form>
            </div>
        </div>
    </div>
</template>

<style scoped>
#map {
}
</style>

<script>
import { Loader } from '@googlemaps/js-api-loader';
import toastr from 'toastr';

export default {
    props: {
        locations: Array,
    },
    data() {
        return {
            map: {
                gmap: null,
                options: {
                    center: {lat: 39.9612, lng: -82.9988},
                    zoom: 8,
                    mapTypeControl: false,
                    fullscreenControl: false,
                    /* Add the below after google loads
                    mapTypeControlOptions: {
                        style: google.maps.MapTypeControlStyle.DROPDOWN_MENU,
                        position: google.maps.ControlPosition.BOTTOM_LEFT,
                    }
                    */
                },
                infoWindow: null,
                markers: [
                ],
            },
            search_q: '43210',
            search_locations: [],
        };
    },
    methods: {
        searchLocations(q) {
            axios.get('/api/locations?q=' + (q || this.search_q))
                .then(resp => {
                    // If no locations are found, show a warning but don't clear the results;
                    if(!resp.data.total) {
                        toastr.warning('No locations found. Try a different search.');
                        return;
                    }
                    let this_page_count = resp.data.to - resp.data.from + 1;
                    if(resp.data.total > this_page_count) {
                        toastr.success('We found ' + resp.data.total + ' locations. Showing the ' + this_page_count + ' closest.');
                    } else {
                        toastr.success('We found ' + this_page_count + ' locations.');
                    }
                    this.search_locations = resp.data.data;
                    document.querySelector('#location-sidebar').scrollTop = 0;
                    this.resetMarkers(this.search_locations);
                });
        },
        resetMarkers(locations) {
            let bounds = new google.maps.LatLngBounds();
            window.markers = window.markers || [];
            window.markers.forEach((marker) => {
                marker.setMap(null);
            });

            // From http://kml4earth.appspot.com/icons.html
            let iconBase = 'https://maps.google.com/mapfiles/kml/paddle/';
            locations.forEach((loc, index) => {
                if(loc.latitude && loc.longitude) {
                    let latLng = { lat: parseFloat(loc.latitude), lng: parseFloat(loc.longitude)};

                    loc.marker = new google.maps.Marker({
                        position: latLng,
                        map: this.map.gmap,
                        title: loc.name,
                        location_idx: index,
                        icon: iconBase + (loc.available ? 'grn-stars.png' : 'wht-blank.png'),
                    });

                    loc.marker.addListener("click", () => {
                        let content = '<h3 class="font-bold">' + loc.name + '</h3>'
                            + '<div class="address text-sm text-gray-700">' + this.addressHtml(loc.address) + '</div>'
                            + '<div class="phone text-sm text-gray-500"><a href="tel:' + loc.phone + '">' + loc.phone + '</a></div>'
                            + (!loc.distance ? '' : '<div class="text-xs text-gray-500">' + this.round(loc.distance) + ' miles</div>')
                            + (!loc.bookinglink ? '' : '<div class="appt-link"><a href="' + loc.bookinglink + '" target="_blank">Search Appointments</a></div>');
                        this.map.infoWindow.setContent(content);
                        this.map.infoWindow.open(this.map.gmap, loc.marker);
                    });

                    window.markers.push(loc.marker);

                    bounds.extend(latLng);
                }
            });
            this.map.gmap.fitBounds(bounds);
        },
        showLocationMarker(loc) {
            console.log('Clicking marker');
            console.log(loc);
            new google.maps.event.trigger( loc.marker, 'click' );
        },
        updateCurrentLocation() {
            if (navigator.geolocation) {
                navigator.geolocation.getCurrentPosition(
                    (position) => {
                        const pos = {
                            lat: position.coords.latitude,
                            lng: position.coords.longitude,
                        };
                        console.log({currentPosition: position});

                        if(this.map.current_location_marker) {
                            this.map.current_location_marker.setMap(null);
                        }

                        this.map.current_location_marker = new google.maps.Marker({
                            position: pos,
                            map: this.map.gmap,
                            title: 'Current Location',
                            icon: 'https://maps.google.com/mapfiles/kml/shapes/placemark_circle_highlight.png',
                        });

                        this.search_q = '';
                        let q = pos.lat + ',' + pos.lng;
                        toastr.info('Thanks for sharing your location. We\'re searching near you.');
                        this.searchLocations(q);
                    }
                );
            }
        },
        round(num, digits) {
            if(digits == null) {
                digits = 1;
            }
            return Math.round(num * Math.pow(10,digits)) / Math.pow(10, digits);
        },
        addressHtml(address) {
            return address.replace(/\||\r/,'<br>');
        },
    },
    mounted() {
        const loader = new Loader({
            apiKey: process.env.MIX_GOOGLE_MAPS_KEY,
            version: "weekly",
            libraries: []
        });

        this.search_locations = this.locations;

        loader
            .load()
            .then(() => {

                this.map.options.mapTypeControl = true;
                this.map.options.mapTypeControlOptions = {
                    style: google.maps.MapTypeControlStyle.DROPDOWN_MENU,
                    position: google.maps.ControlPosition.BOTTOM_LEFT,
                };
                this.map.gmap = new google.maps.Map(document.getElementById('map'), this.map.options)
                this.map.infoWindow = new google.maps.InfoWindow();
                window.map = this.map.gmap;
                window.infoWindow = this.map.infoWindow;

                this.resetMarkers(this.locations);
                this.updateCurrentLocation();
            });
    },
};
</script>
