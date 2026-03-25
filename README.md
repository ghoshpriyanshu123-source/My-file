<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>Location Tracer</title>
    <style>
        * {
            margin: 0;
            padding: 0;
            box-sizing: border-box;
        }

        body {
            font-family: 'Segoe UI', Tahoma, Geneva, Verdana, sans-serif;
            background: linear-gradient(135deg, #667eea 0%, #764ba2 100%);
            min-height: 100vh;
            display: flex;
            justify-content: center;
            align-items: center;
            color: #333;
        }

        .container {
            background: white;
            padding: 2rem;
            border-radius: 20px;
            box-shadow: 0 20px 40px rgba(0,0,0,0.1);
            max-width: 500px;
            width: 90%;
            text-align: center;
        }

        h1 {
            color: #4a5568;
            margin-bottom: 1.5rem;
            font-size: 2rem;
        }

        .btn {
            background: linear-gradient(45deg, #667eea, #764ba2);
            color: white;
            border: none;
            padding: 12px 30px;
            border-radius: 25px;
            font-size: 1.1rem;
            cursor: pointer;
            transition: transform 0.2s, box-shadow 0.2s;
            margin: 10px;
        }

        .btn:hover {
            transform: translateY(-2px);
            box-shadow: 0 10px 20px rgba(0,0,0,0.2);
        }

        .btn:disabled {
            opacity: 0.6;
            cursor: not-allowed;
            transform: none;
        }

        .location-info {
            margin-top: 2rem;
            text-align: left;
            background: #f8f9fa;
            padding: 1.5rem;
            border-radius: 10px;
            display: none;
        }

        .info-row {
            display: flex;
            justify-content: space-between;
            margin-bottom: 0.8rem;
            padding: 0.5rem;
            background: white;
            border-radius: 8px;
            box-shadow: 0 2px 4px rgba(0,0,0,0.05);
        }

        .label {
            font-weight: 600;
            color: #4a5568;
        }

        .value {
            color: #667eea;
            font-family: monospace;
        }

        .map-container {
            margin-top: 1.5rem;
            height: 300px;
            border-radius: 10px;
            overflow: hidden;
            display: none;
        }

        #map {
            width: 100%;
            height: 100%;
        }

        .status {
            margin: 1rem 0;
            padding: 0.8rem;
            border-radius: 8px;
            font-weight: 500;
        }

        .status.success {
            background: #d4edda;
            color: #155724;
            border: 1px solid #c3e6cb;
        }

        .status.error {
            background: #f8d7da;
            color: #721c24;
            border: 1px solid #f5c6cb;
        }

        .watch-btn {
            background: linear-gradient(45deg, #48bb78, #38a169);
        }

        .stop-btn {
            background: linear-gradient(45deg, #f56565, #e53e3e);
        }
    </style>
</head>
<body>
    <div class="container">
        <h1>📍 Location Tracer</h1>
        
        <button id="getLocationBtn" class="btn">Get My Location</button>
        <button id="watchLocationBtn" class="btn watch-btn">Start Tracking</button>
        <button id="stopTrackingBtn" class="btn stop-btn" style="display: none;">Stop Tracking</button>

        <div id="status"></div>

        <div id="locationInfo" class="location-info"></div>
        
        <div id="mapContainer" class="map-container">
            <div id="map"></div>
        </div>
    </div>

    <!-- Leaflet CSS and JS for maps -->
    <link rel="stylesheet" href="https://unpkg.com/leaflet@1.9.4/dist/leaflet.css" />
    <script src="https://unpkg.com/leaflet@1.9.4/dist/leaflet.js"></script>

    <script>
        class LocationTracer {
            constructor() {
                this.watchId = null;
                this.map = null;
                this.currentMarker = null;
                this.initElements();
                this.bindEvents();
            }

            initElements() {
                this.getLocationBtn = document.getElementById('getLocationBtn');
                this.watchLocationBtn = document.getElementById('watchLocationBtn');
                this.stopTrackingBtn = document.getElementById('stopTrackingBtn');
                this.statusDiv = document.getElementById('status');
                this.locationInfo = document.getElementById('locationInfo');
                this.mapContainer = document.getElementById('mapContainer');
            }

            bindEvents() {
                this.getLocationBtn.addEventListener('click', () => this.getCurrentPosition());
                this.watchLocationBtn.addEventListener('click', () => this.startWatching());
                this.stopTrackingBtn.addEventListener('click', () => this.stopWatching());
            }

            showStatus(message, type = 'success') {
                this.statusDiv.textContent = message;
                this.statusDiv.className = `status ${type}`;
                this.statusDiv.style.display = 'block';
                
                setTimeout(() => {
                    this.statusDiv.style.display = 'none';
                }, 5000);
            }

            getCurrentPosition() {
                if (!navigator.geolocation) {
                    this.showStatus('Geolocation is not supported by this browser.', 'error');
                    return;
                }

                this.getLocationBtn.disabled = true;
                this.getLocationBtn.textContent = 'Getting location...';

                navigator.geolocation.getCurrentPosition(
                    (position) => {
                        this.handlePosition(position);
                        this.getLocationBtn.disabled = false;
                        this.getLocationBtn.textContent = 'Get My Location';
                    },
                    (error) => {
                        this.handleLocationError(error);
                        this.getLocationBtn.disabled = false;
                        this.getLocationBtn.textContent = 'Get My Location';
                    },
                    {
                        enableHighAccuracy: true,
                        timeout: 10000,
                        maximumAge: 60000
                    }
                );
            }

            startWatching() {
                if (!navigator.geolocation) {
                    this.showStatus('Geolocation is not supported by this browser.', 'error');
                    return;
                }

                this.watchId = navigator.geolocation.watchPosition(
                    (position) => {
                        this.handlePosition(position);
                    },
                    (error) => {
                        this.handleLocationError(error);
                    },
                    {
                        enableHighAccuracy: true,
                        timeout: 5000,
                        maximumAge: 0
                    }
                );

                this.watchLocationBtn.style.display = 'none';
                this.stopTrackingBtn.style.display = 'inline-block';
                this.showStatus('Started tracking your location...');
            }

            stopWatching() {
                if (this.watchId) {
                    navigator.geolocation.clearWatch(this.watchId);
                    this.watchId = null;
                }
                this.watchLocationBtn.style.display = 'inline-block';
                this.stopTrackingBtn.style.display = 'none';
                this.showStatus('Stopped tracking.');
            }

            handlePosition(position) {
                const { latitude, longitude, accuracy } = position.coords;
                const timestamp = new Date(position.timestamp).toLocaleString();

                this.displayLocationInfo({
                    latitude,
                    longitude,
                    accuracy: accuracy.toFixed(0),
                    altitude: position.coords.altitude?.toFixed(0) || 'N/A',
                    speed: position.coords.speed ? (position.coords.speed * 3.6).toFixed(1) + ' km/h' : 'N/A',
                    timestamp
                });

                this.updateMap(latitude, longitude);
                this.showStatus('Location updated successfully!');
            }

            handleLocationError(error) {
                let message = '';
                switch(error.code) {
                    case error.PERMISSION_DENIED:
                        message = "Location access denied. Please enable location permissions.";
                        break;
                    case error.POSITION_UNAVAILABLE:
                        message = "Location information is unavailable.";
                        break;
                    case error.TIMEOUT:
                        message = "Location request timed out.";
                        break;
                    default:
                        message = "An unknown error occurred.";
                        break;
                }
                this.showStatus(message, 'error');
            }

            displayLocationInfo(data) {
                this.locationInfo.innerHTML = `
                    <div class="info-row">
                        <span class="label">Latitude:</span>
                        <span class="value">${data.latitude.toFixed(6)}</span>
                    </div>
                    <div class="info-row">
                        <span class="label">Longitude:</span>
                        <span class="value">${data.longitude.toFixed(6)}</span>
                    </div>
                    <div class="info-row">
                        <span class="label">Accuracy:</span>
                        <span class="value">${data.accuracy} m</span>
                    </div>
                    <div class="info-row">
                        <span class="label">Altitude:</span>
                        <span class="value">${data.altitude}</span>
                    </div>
                    <div class="info-row">
                        <span class="label">Speed:</span>
                        <span class="value">${data.speed}</span>
                    </div>
                    <div class="info-row">
                        <span class="label">Time:</span>
                        <span class="value">${data.timestamp}</span>
                    </div>
                `;
                this.locationInfo.style.display = 'block';
            }

            updateMap(latitude, longitude) {
                if (!this.map) {
                    this.map = L.map('map').setView([latitude, longitude], 15);
                    L.tileLayer('https://{s}.tile.openstreetmap.org/{z}/{x}/{y}.png', {
                        attribution: '© OpenStreetMap contributors'
                    }).addTo(this.map);
                }

                this.mapContainer.style.display = 'block';

                if (this.currentMarker) {
                    this.map.removeLayer(this.currentMarker);
                }

                this.currentMarker = L.marker([latitude, longitude])
                    .addTo(this.map)
                    .bindPopup(`Your location<br>Lat: ${latitude.toFixed(4)}<br>Lng: ${longitude.toFixed(4)}`)
                    .openPopup();

                this.map.setView([latitude, longitude], 15);
            }
        }

        // Initialize the app when page loads
        document.addEventListener('DOMContentLoaded', () => {
            new LocationTracer();
        });
    </script>
</body>
</html>
