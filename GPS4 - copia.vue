<script>
/* eslint-disable no-undef */
import {computed, ref} from 'vue'
import {useGeolocation} from './useGeolocation'
import {Loader} from '@googlemaps/js-api-loader'
import Swal from 'sweetalert2'

//Poner su propia API KEY de Google
const GOOGLE_MAPS_API_KEY = 'loremIpsum1234'


export default {
    name: 'GPS4',
    //setup se inicia mucho antes
    created() {
        //saber la accion del navegador, reload, navigate, back_forward
        //console.log(performance.getEntriesByType('navigation')[0].type)
        if (performance.getEntriesByType('navigation')[0].type === 'reload') {
            console.log('se refresco')
            sessionStorage.clear();
        }
    },
    setup() {
        const {coords} = useGeolocation()
        const currPos = computed(() => ({
            lat: coords.value.latitude,
            lng: coords.value.longitude
        }))
        /*
        //forma directa y sencilla
        const success =(position) => {
            console.log(position)
        }
        const error = (error) => {
            console.log(error)
        }
        navigator.geolocation.getCurrentPosition(success, error);
        */

        const loader = new Loader({
            apiKey: GOOGLE_MAPS_API_KEY, libraries: ['places']
        });
        let mapDiv = ref(null)
        let map = ref(null)
        let marker = ref(null)

        let resul_lat = ref(null);
        let resul_lng = ref(null);

        let placeDet = ref({
            exterior: '',
            interior: '',
            calle: '',
            colonia: '',
            CP: '',
            localidad: '',
            municipio: '',
            estado: ''
        })


        //initMap

        loader.load().then(() => {
            let center;

            if (sessionStorage.getItem('center')) {
                console.log('session')
                center = JSON.parse(sessionStorage.getItem('center'))
                placeDet.value = JSON.parse(sessionStorage.getItem('placeDet'))
                document.getElementById('place-input').value = sessionStorage.getItem('placeInput')
                resul_lng.value = center.lng;
                resul_lat.value = center.lat;
            } else {
                center = currPos.value;
                resul_lng.value = center.lng;
                resul_lat.value = center.lat;
            }

            map.value = new google.maps.Map(mapDiv.value, {
                //centrado en posicion actual
                center: center,
                zoom: 18
            })

            marker = new google.maps.Marker({
                map: map.value,
                draggable: true,
                animation: google.maps.Animation.DROP,
                position: center
            })

            google.maps.event.addListener(marker, 'dragend', function () {
                //alert(marker.getPosition())
                resul_lat.value = marker.getPosition().lat()
                resul_lng.value = marker.getPosition().lng()
                center = {
                    lat: resul_lat.value,
                    lng: resul_lng.value
                }
                //guardar center por arrastre
                sessionStorage.setItem('center', JSON.stringify(center))
            })

            //autocompletar
            const placeInput = document.getElementById("place-input");
            const options = {
                componentRestrictions: {country: "mx"},
                types: ['geocode']
            }
            const autocomplete = new google.maps.places.Autocomplete(placeInput, options);
            google.maps.event.addListener(autocomplete, 'place_changed', function () {
                //obtener el lugar
                let place = autocomplete.getPlace();

                sessionStorage.setItem('placeInput', placeInput.value)

                center = place.geometry.location;
                //guardar center por busqueda plces
                sessionStorage.setItem('center', JSON.stringify(center))
                //centrar el mapa en el lugar
                map.value.setCenter(center)
                //poner marcador en ese lugar
                marker.setPosition(center)

                sessionStorage.setItem('map', map.value);
                sessionStorage.setItem('marker', marker.value);


                resul_lat.value = center.lat();
                resul_lng.value = center.lng();

                //limpiar todo
                exterior.value = ''
                calle.value = ''
                colonia.value = ''
                CP.value = ''
                municipio.value = ''
                estado.value = ''

                placeDet.value = {
                    exterior: '',
                    interior: '',
                    calle: '',
                    colonia: '',
                    CP: '',
                    localidad: '',
                    municipio: '',
                    estado: '',
                }

                //guardar detalles
                let detalles = place.address_components
                for (let i = 0; i < detalles.length; i++) {
                    var d = detalles[i]
                    switch (d.types[0]) {
                        case "street_number":
                            placeDet.value.exterior = d.long_name
                            //alert('numero exterior '+d.long_name)
                            break
                        case "route":
                            placeDet.value.calle = d.long_name
                            //alert('numero exterior '+d.long_name)
                            break
                        case "sublocality_level_1":
                            placeDet.value.colonia = d.long_name
                            //alert('la colonia es ' + d.long_name)
                            break
                        case "postal_code":
                            placeDet.value.CP = d.long_name
                            //alert('la colonia es ' + d.long_name)
                            break
                        case "locality":
                            placeDet.value.municipio = d.long_name
                            //alert('municipio es ' + d.long_name)
                            break
                        case "administrative_area_level_1":
                            placeDet.value.estado = d.long_name
                            //alert('estado es ' + d.long_name)
                            break
                        case 'country':
                            //alert('el pais es ' + d.long_name)
                            break
                    }
                }
                sessionStorage.setItem('placeDet', JSON.stringify(placeDet.value));
            });

        })


        return {
            //mapa
            mapDiv,
            //coordenadas
            resul_lat, resul_lng,
            placeDet
        }
    }

}
</script>

<template>
    <div class="py-12 mx-auto">
        <!-- Autocomplete location search input -->
        <div class="form-group">
            <input class="w-full" id="place-input" type="text" placeholder="Type address..."/>
        </div>
        <div class="m-6" ref="mapDiv" style="width: 95%; height: 400px"/>
        <p><b>Latitude:</b>{{ resul_lat }}</p>
        <p><b>Longitude:</b>{{ resul_lng }}</p>
        <a v-if="resul_lng!=null" class="text-blue-600 no-underline hover:underline"
           :href="`https://www.google.com/maps/search/${resul_lat},${resul_lng}`">
            Ver en Google Maps</a>

        <form action="">
            <div class="form-group">
                <label>Calle: </label>
                <input type="text" v-model="placeDet.calle" placeholder="Calle">
            </div>
            <div class="form-group">
                <label>Número Exterior: </label>
                <input type="text" v-model="placeDet.exterior" placeholder="Número Exterior">
            </div>
            <div class="form-group">
                <label>Número Interior: </label>
                <input type="text" v-model="placeDet.interior" placeholder="Número Interior">
            </div>
            <div class="form-group">
                <label>Colonia: </label>
                <input type="text" v-model="placeDet.colonia" placeholder="Colonia">
            </div>
            <div class="form-group">
                <label>CP: </label>
                <input type="text" v-model="placeDet.CP" placeholder="CP">
            </div>
            <div class="form-group">
                <label>Localidad: </label>
                <input type="text" v-model="placeDet.localidad" placeholder="Localidad">
            </div>
            <div class="form-group">
                <label>Municipio: </label>
                <input type="text" v-model="placeDet.municipio" placeholder="Municipio">
            </div>
            <div class="form-group">
                <label>Estado: </label>
                <!--<input type="text" v-model="placeDet.estado" placeholder="Estado">-->
                <select v-model="placeDet.estado" id="">
                    <option disabled hidden>Estado</option>
                    <option value="Michoacán">Michoacán</option>
                    <option value="Coahuila">Coahuila</option>
                    <option value="Veracruz">Veracruz</option>
                </select>
            </div>
        </form>

    </div>
</template>
