<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>Tourist Safety App</title>
    <script src="https://cdn.tailwindcss.com"></script>
    <link rel="preconnect" href="https://fonts.googleapis.com">
    <link rel="preconnect" href="https://fonts.gstatic.com" crossorigin>
    <link href="https://fonts.googleapis.com/css2?family=Inter:wght@400;500;600;700&display=swap" rel="stylesheet">
    <style>
        body {
            font-family: 'Inter', sans-serif;
            background-color: #f3f4f6;
        }
        .container {
            max-width: 90%;
        }
        .unsafe-zone {
            animation: pulse-red 1s infinite;
        }
        @keyframes pulse-red {
            0%, 100% {
                transform: scale(1);
                box-shadow: 0 0 0 0 rgba(239, 68, 68, 0.7);
            }
            70% {
                transform: scale(1.05);
                box-shadow: 0 0 0 10px rgba(239, 68, 68, 0);
            }
        }
        .safe-zone {
            animation: pulse-green 1s infinite;
        }
        @keyframes pulse-green {
            0%, 100% {
                transform: scale(1);
                box-shadow: 0 0 0 0 rgba(16, 185, 129, 0.7);
            }
            70% {
                transform: scale(1.05);
                box-shadow: 0 0 0 10px rgba(16, 185, 129, 0);
            }
        }
        #location-log {
            max-height: 400px;
            overflow-y: auto;
        }
        #map {
            height: 500px;
            width: 100%;
            border-radius: 0.75rem;
            box-shadow: 0 4px 6px rgba(0, 0, 0, 0.1);
        }
    </style>
    <!-- Leaflet CSS -->
    <link rel="stylesheet" href="https://unpkg.com/leaflet@1.9.4/dist/leaflet.css"
        integrity="sha256-p4NxAoJBhIIN85kRjmh0oB63I4b+POOqVLUHEaSbh+g="
        crossorigin=""/>
    <!-- Leaflet JS -->
    <script src="https://unpkg.com/leaflet@1.9.4/dist/leaflet.js"
        integrity="sha256-200S+OQ+H5j+h+1JvRj2G5A5rB7I7L5M5J4L2J+K+P4="
        crossorigin=""></script>
</head>
<body class="bg-gray-100 flex items-center justify-center min-h-screen p-4">

    <!-- Firebase SDKs -->
    <script type="module">
        import { initializeApp } from "https://www.gstatic.com/firebasejs/11.6.1/firebase-app.js";
        import { getAuth, signInAnonymously, signInWithCustomToken, onAuthStateChanged } from "https://www.gstatic.com/firebasejs/11.6.1/firebase-auth.js";
        import { getFirestore, doc, getDoc, setDoc, onSnapshot, collection, query, addDoc, serverTimestamp, orderBy } from "https://www.gstatic.com/firebasejs/11.6.1/firebase-firestore.js";
        import { setLogLevel } from "https://www.gstatic.com/firebasejs/11.6.1/firebase-firestore.js";

        // Global variables for Firebase setup
        const appId = typeof __app_id !== 'undefined' ? __app_id : 'default-app-id';
        const firebaseConfig = JSON.parse(typeof __firebase_config !== 'undefined' ? __firebase_config : '{}');
        const initialAuthToken = typeof __initial_auth_token !== 'undefined' ? __initial_auth_token : null;

        // Firestore and Auth instances
        let app, db, auth;
        let userId = null;
        let isAuthReady = false;
        let generatedOTP = null;
        
        let isInitialized = false;
        let geoWatchId = null;
        let escalationTimer = null;
        const escalationTimeout = 180000;
        const safetyStatus = {
            SAFE: 'safe',
            UNSAFE: 'unsafe',
            ESCALATED: 'escalated'
        };

        let currentStatus = safetyStatus.SAFE;
        let lastReportedLocation = null;
        let userLocationHistory = [];
        const maxHistory = 10;
        let isOtpVerified = false;
        
        let currentUnsafeZones = [];
        let simulatedLocationIndex = 0;

        // Admin credentials for different authorities
        const ADMIN_CREDENTIALS = {
            'police': 'password123',
            'hospital': 'password123',
            'touristauthority': 'password123'
        };
        let currentAdminRole = '';
        
        const famousPlacesData = {
            'hyderabad': {
                places: [
                    'Choose a place...',
                    'Charminar',
                    'Golconda Fort',
                    'Salar Jung Museum',
                    'Ramoji Film City',
                    'Hussain Sagar Lake'
                ],
                unsafeZones: [
                    { lat: 17.4124, lng: 78.4735, radius: 200, name: 'Charminar Unsafe Zone' },
                    { lat: 17.4239, lng: 78.4733, radius: 150, name: 'Salar Jung Museum Restricted Area' },
                    { lat: 17.4372, lng: 78.4480, radius: 250, name: 'Golconda Fort Unsafe Zone' }
                ],
                simulatedLocations: [
                    { lat: 17.4100, lng: 78.4700 },
                    { lat: 17.4120, lng: 78.4730 },
                    { lat: 17.4124, lng: 78.4735 },
                    { lat: 17.4126, lng: 78.4737 },
                    { lat: 17.4130, lng: 78.4740 },
                    { lat: 17.4150, lng: 78.4750 }
                ]
            },
            'pune': {
                places: [
                    'Choose a place...',
                    'Shaniwar Wada',
                    'Aga Khan Palace',
                    'Sinhagad Fort',
                    'Dagdusheth Halwai Ganpati Temple',
                    'Pataleshwar Cave Temple'
                ],
                unsafeZones: [
                    { lat: 18.5197, lng: 73.8567, radius: 250, name: 'Shaniwar Wada Unsafe Zone' },
                    { lat: 18.5209, lng: 73.8573, radius: 150, name: 'Pataleshwar Cave Temple Restricted Area' }
                ],
                simulatedLocations: [
                    { lat: 18.5170, lng: 73.8550 },
                    { lat: 18.5190, lng: 73.8560 },
                    { lat: 18.5197, lng: 73.8567 },
                    { lat: 18.5199, lng: 73.8569 },
                    { lat: 18.5200, lng: 73.8570 },
                    { lat: 18.5210, lng: 73.8580 }
                ]
            },
            'delhi': {
                places: [
                    'Choose a place...',
                    'Red Fort',
                    'Qutub Minar',
                    'Humayun\'s Tomb',
                    'India Gate',
                    'Lotus Temple'
                ],
                unsafeZones: [
                    { lat: 28.6562, lng: 77.2410, radius: 200, name: 'Red Fort Unsafe Zone' },
                    { lat: 28.5245, lng: 77.1855, radius: 150, name: 'Qutub Minar Restricted Area' }
                ],
                simulatedLocations: [
                    { lat: 28.6550, lng: 77.2400 },
                    { lat: 28.6560, lng: 77.2408 },
                    { lat: 28.6562, lng: 77.2410 },
                    { lat: 28.6564, lng: 77.2412 },
                    { lat: 28.6570, lng: 77.2415 },
                    { lat: 28.6580, lng: 77.2420 }
                ]
            }
        };

        // --- UI Helper Functions ---
        const getEl = id => document.getElementById(id);
        const hideElement = id => getEl(id).classList.add('hidden');
        const showElement = id => getEl(id).classList.remove('hidden');

        const showScreen = (screenId) => {
            hideElement('start-screen');
            hideElement('form-screen');
            hideElement('app-screen');
            hideElement('admin-login-screen');
            hideElement('admin-dashboard-screen');
            showElement(screenId);
        }

        // --- Firebase Functions ---
        const authenticateUser = async () => {
            if (auth) {
                try {
                    if (initialAuthToken) {
                        await signInWithCustomToken(auth, initialAuthToken);
                        console.log("Signed in with custom token.");
                    } else {
                        await signInAnonymously(auth);
                        console.log("Signed in anonymously.");
                    }
                    return true;
                } catch (error) {
                    console.error("Authentication failed:", error);
                    return false;
                }
            }
            return false;
        };

        const saveUserData = async (userData) => {
            if (db && userId) {
                try {
                    const docRef = doc(db, `artifacts/${appId}/users/${userId}/data`, 'profile');
                    await setDoc(docRef, userData, { merge: true });
                    console.log("User data saved successfully.");
                } catch (e) {
                    console.error("Error saving user data:", e);
                }
            } else {
                console.error("Firestore not initialized or userId not available.");
            }
        };

        const loadUserData = () => {
            if (db && userId) {
                const docRef = doc(db, `artifacts/${appId}/users/${userId}/data`, 'profile');
                onSnapshot(docRef, (docSnap) => {
                    if (docSnap.exists()) {
                        const data = docSnap.data();
                        getEl('name-input').value = data.name || '';
                        getEl('phone-input').value = data.phoneNumber || '';
                        getEl('emergency-name-input').value = data.emergencyContactName || '';
                        getEl('emergency-phone-input').value = data.emergencyContactPhone || '';
                        getEl('destination-input').value = data.destination || '';
                        getEl('famous-place-select').value = data.destinationPlace || '';
                        getEl('simulated-mode-checkbox').checked = data.isSimulated || false;
                        getEl('custom-location-checkbox').checked = data.isCustomLocation || false;
                        getEl('custom-location-input').value = data.customLocation || '';
                        
                        if (data.isCustomLocation) {
                            showElement('custom-destination-section');
                            hideElement('place-selection-section');
                        }
                    } else {
                        console.log("No user profile found. Starting fresh.");
                    }
                }, (error) => {
                    console.error("Error loading user data:", error);
                });
            }
        };

        const sendLiveLocationToDemoServer = async (location) => {
            if (db && userId) {
                try {
                    const collectionRef = collection(db, `artifacts/${appId}/users/${userId}/live_location`);
                    await addDoc(collectionRef, {
                        timestamp: serverTimestamp(),
                        latitude: location.lat,
                        longitude: location.lng
                    });
                } catch (e) {
                    console.error("Error saving live location data:", e);
                }
            }
        };
        
        // --- NEW: Function to create a public emergency incident for the admin dashboard
        const createEmergencyIncident = async (incidentType) => {
             if (db && userId && lastReportedLocation) {
                try {
                    const docRef = doc(db, `artifacts/${appId}/public/data/emergencyLocations`, userId);
                    await setDoc(docRef, {
                        timestamp: serverTimestamp(),
                        latitude: lastReportedLocation.lat,
                        longitude: lastReportedLocation.lng,
                        type: incidentType,
                        userId: userId
                    });
                    console.log(`Emergency incident of type '${incidentType}' created successfully.`);
                } catch (e) {
                    console.error("Error creating emergency incident:", e);
                }
            } else {
                console.error("Cannot create incident. Missing DB, userId, or location.");
            }
        };

        // --- Geolocation & Safety Functions ---
        const getDistance = (lat1, lon1, lat2, lon2) => {
            const R = 6371e3; // metres
            const φ1 = lat1 * Math.PI/180;
            const φ2 = lat2 * Math.PI/180;
            const Δφ = (lat2-lat1) * Math.PI/180;
            const Δλ = (lon2-lon1) * Math.PI/180;

            const a = Math.sin(Δφ/2) * Math.sin(Δφ/2) +
                    Math.cos(φ1) * Math.cos(φ2) *
                    Math.sin(Δλ/2) * Math.sin(Δλ/2);
            const c = 2 * Math.atan2(Math.sqrt(a), Math.sqrt(1-a));

            return R * c;
        }

        const startGeolocationWatch = (simulatedLocations = null) => {
            const isSimulatedMode = getEl('simulated-mode-checkbox').checked;
            const isCustomLocation = getEl('custom-location-checkbox').checked;

            if (isSimulatedMode && !isCustomLocation) {
                simulatedLocationIndex = 0;
                geoWatchId = setInterval(() => {
                    const position = simulatedLocations[simulatedLocationIndex];
                    simulatedLocationIndex = (simulatedLocationIndex + 1) % simulatedLocations.length;
                    
                    const lat = position.lat;
                    const lng = position.lng;
                    
                    lastReportedLocation = { lat, lng };
                    userLocationHistory.push(lastReportedLocation);
                    if (userLocationHistory.length > maxHistory) {
                        userLocationHistory.shift();
                    }
                    
                    getEl('user-location').textContent = `[SIMULATED] Latitude: ${lat.toFixed(4)}, Longitude: ${lng.toFixed(4)}`;
                    
                    const inUnsafeZone = currentUnsafeZones.find(zone => {
                        const distance = getDistance(lat, lng, zone.lat, zone.lng);
                        return distance <= zone.radius;
                    });

                    sendLiveLocationToDemoServer(lastReportedLocation);

                    if (inUnsafeZone) {
                        handleUnsafeZone(inUnsafeZone);
                    } else {
                        handleSafeZone();
                    }
                }, 3000);
            } else {
                if (navigator.geolocation && geoWatchId === null) {
                    geoWatchId = navigator.geolocation.watchPosition(
                        position => {
                            const lat = position.coords.latitude;
                            const lng = position.coords.longitude;
                            lastReportedLocation = { lat, lng };
                            userLocationHistory.push(lastReportedLocation);
                            if (userLocationHistory.length > maxHistory) {
                                userLocationHistory.shift();
                            }
                            
                            getEl('user-location').textContent = `Latitude: ${lat.toFixed(4)}, Longitude: ${lng.toFixed(4)}`;
                            
                            sendLiveLocationToDemoServer(lastReportedLocation);

                            if (!isCustomLocation) {
                                const inUnsafeZone = currentUnsafeZones.find(zone => {
                                    const distance = getDistance(lat, lng, zone.lat, zone.lng);
                                    return distance <= zone.radius;
                                });
                                if (inUnsafeZone) {
                                    handleUnsafeZone(inUnsafeZone);
                                } else {
                                    handleSafeZone();
                                }
                            } else {
                                handleSafeZone();
                            }
                        },
                        error => {
                            console.error("Geolocation error:", error);
                            let errorMessage = 'Location access denied or unavailable. Please enable permissions or use simulated mode for presentation.';
                            getEl('user-location').textContent = errorMessage;
                            updateSafetyStatus('status-indicator', 'bg-gray-500 text-white', 'Location Unavailable');
                        },
                        { enableHighAccuracy: true, timeout: 5000, maximumAge: 0 }
                    );
                } else {
                     console.error("Geolocation API not supported or already watching.");
                }
            }
            showScreen('app-screen');
        };

        const stopGeolocationWatch = () => {
            if (geoWatchId !== null) {
                const isSimulatedMode = getEl('simulated-mode-checkbox').checked;
                if (isSimulatedMode) {
                    clearInterval(geoWatchId);
                } else {
                    navigator.geolocation.clearWatch(geoWatchId);
                }
                geoWatchId = null;
                console.log("Geolocation watch stopped.");
            }
        };

        const updateSafetyStatus = (elementId, colorClass, text) => {
            const el = getEl(elementId);
            el.className = `p-4 rounded-xl text-center font-bold mb-4 ${colorClass}`;
            el.textContent = text;
        };
        
        const playAlarmSound = () => {
            const audioCtx = new (window.AudioContext || window.webkitAudioContext)();
            const oscillator = audioCtx.createOscillator();
            const gainNode = audioCtx.createGain();
            oscillator.connect(gainNode);
            gainNode.connect(audioCtx.destination);
            oscillator.type = 'square';
            oscillator.frequency.value = 440;
            gainNode.gain.setValueAtTime(0.5, audioCtx.currentTime);
            oscillator.start();
            setTimeout(() => { oscillator.stop(); }, 500);
        };

        const startEscalationTimer = () => {
            if (escalationTimer) clearTimeout(escalationTimer);
            getEl('timer-text').textContent = `Escalation in ${escalationTimeout / 60000} minutes...`;
            showElement('timer-display');
            
            escalationTimer = setTimeout(() => {
                currentStatus = safetyStatus.ESCALATED;
                console.log("Alert escalated to authorities!");
                const message = "ALERT: Your live location has been sent to local authorities and tourist department.";
                showNotificationModal('High-Risk Alert Escalated!', message, 'bg-red-500');
                updateSafetyStatus('status-indicator', 'bg-red-500 text-white unsafe-zone', 'High-Risk Alert Escalated!');
                playAlarmSound();
                // Create emergency incident for admin dashboard
                createEmergencyIncident('High-Risk Alert');
            }, escalationTimeout);
        };

        const stopEscalationTimer = () => {
            if (escalationTimer) {
                clearTimeout(escalationTimer);
                escalationTimer = null;
                console.log("Escalation timer stopped.");
            }
            hideElement('timer-display');
        };

        const handleUnsafeZone = (zone) => {
            if (currentStatus === safetyStatus.SAFE) {
                currentStatus = safetyStatus.UNSAFE;
                console.log(`User entered unsafe zone: ${zone.name}`);
                const message = `WARNING: You are entering a restricted or unsafe area: ${zone.name}. Please leave immediately.`;
                showNotificationModal('Warning!', message, 'bg-yellow-500');
                updateSafetyStatus('status-indicator', 'bg-yellow-500 text-white unsafe-zone', 'Unsafe Zone Detected!');
                playAlarmSound();
                startEscalationTimer();
            }
        };

        const handleSafeZone = () => {
            if (currentStatus !== safetyStatus.SAFE) {
                currentStatus = safetyStatus.SAFE;
                console.log("User is in a safe zone.");
                updateSafetyStatus('status-indicator', 'bg-green-500 text-white safe-zone', 'You are in a safe area.');
                stopEscalationTimer();
            }
        };

        const handleSOS = () => {
            stopGeolocationWatch();
            stopEscalationTimer();
            currentStatus = safetyStatus.ESCALATED;
            console.log("SOS activated. Alert escalated to authorities!");
            const message = "EMERGENCY: Your live location has been sent to local authorities and tourist department.";
            showNotificationModal('SOS Activated!', message, 'bg-red-500');
            updateSafetyStatus('status-indicator', 'bg-red-500 text-white unsafe-zone', 'SOS - Authorities Notified!');
            playAlarmSound();
            createEmergencyIncident('SOS');
        };
        
        // --- Modal/Notification Functions ---
        const showNotificationModal = (title, message, bgColor) => {
            const modal = getEl('notification-modal');
            getEl('notification-title').textContent = title;
            getEl('notification-message').textContent = message;
            getEl('notification-modal-content').className = `bg-white p-6 rounded-xl shadow-lg border-t-8 ${bgColor}`;
            modal.classList.remove('hidden');
        };

        const populateFamousPlaces = (cityName) => {
            const cityData = famousPlacesData[cityName.toLowerCase()];
            const selectEl = getEl('famous-place-select');
            selectEl.innerHTML = '';

            if (cityData) {
                cityData.places.forEach(place => {
                    const option = document.createElement('option');
                    option.value = place;
                    option.textContent = place;
                    selectEl.appendChild(option);
                });
                currentUnsafeZones = cityData.unsafeZones;
                showElement('place-selection-section');
            } else {
                hideElement('place-selection-section');
                showNotificationModal('City Not Found', 'Sorry, we do not have data for this city yet. Please try another.', 'bg-red-500');
            }
        };
        
        const resetForm = () => {
            getEl('user-form').reset();
            isOtpVerified = false;
            hideElement('otp-section');
            hideElement('emergency-contact-section');
            hideElement('destination-section');
            hideElement('place-selection-section');
            hideElement('custom-destination-section');
            hideElement('start-monitoring-btn');
            hideElement('send-otp-btn');
            getEl('otp-info').classList.remove('hidden');
            updateSafetyStatus('status-indicator', 'bg-green-500 text-white safe-zone', 'You are in a safe area.');
        };

        // --- Admin Dashboard Functions ---
        let map;
        let mapMarker;
        const incidentList = getEl('incidentList');
        const loadingMessage = getEl('loadingMessage');
        const noIncidentsMessage = getEl('noIncidents');
        const mapContainer = getEl('mapContainer');
        const mapPlaceholder = getEl('mapPlaceholder');
        
        function initMap(latitude, longitude) {
            if (map) {
                map.remove();
            }
            map = L.map('map').setView([latitude, longitude], 15);
            L.tileLayer('https://tile.openstreetmap.org/{z}/{x}/{y}.png', {
                maxZoom: 19,
                attribution: '&copy; <a href="http://www.openstreetmap.org/copyright">OpenStreetMap</a>'
            }).addTo(map);

            mapMarker = L.marker([latitude, longitude]).addTo(map)
                .bindPopup('Emergency Location').openPopup();

            mapContainer.classList.remove('hidden');
            mapPlaceholder.classList.add('hidden');
        }

        window.viewLocation = function(latitude, longitude) {
            initMap(latitude, longitude);
        };
        
        function renderIncidents(incidents) {
            incidentList.innerHTML = '';
            loadingMessage.classList.add('hidden');

            if (incidents.length === 0) {
                noIncidentsMessage.classList.remove('hidden');
            } else {
                noIncidentsMessage.classList.add('hidden');
                incidents.forEach(incident => {
                    const incidentItem = document.createElement('div');
                    incidentItem.className = 'bg-white p-4 rounded-lg shadow-sm border border-gray-100 flex justify-between items-center';
                    const timestamp = incident.timestamp ? new Date(incident.timestamp.toDate()).toLocaleString() : 'N/A';
                    
                    incidentItem.innerHTML = `
                        <div>
                            <p class="font-semibold text-gray-800">User ID: <span class="text-sm font-normal text-gray-500">${incident.id}</span></p>
                            <p class="text-sm text-gray-500 mt-1">Status: <span class="text-red-500 font-medium">Emergency</span></p>
                            <p class="text-xs text-gray-400 mt-1">Last Updated: ${timestamp}</p>
                        </div>
                        <button onclick="viewLocation(${incident.latitude}, ${incident.longitude})" class="bg-indigo-500 hover:bg-indigo-600 text-white font-medium py-2 px-4 rounded-full text-sm transition-colors duration-200">
                            View Location
                        </button>
                    `;
                    incidentList.appendChild(incidentItem);
                });
            }
        }

        const setupAdminDashboard = () => {
            const emergencyCollectionPath = `artifacts/${appId}/public/data/emergencyLocations`;
            const q = collection(db, emergencyCollectionPath);

            onSnapshot(q, (snapshot) => {
                const incidents = [];
                snapshot.forEach(doc => {
                    incidents.push({ id: doc.id, ...doc.data() });
                });
                renderIncidents(incidents);
            }, (error) => {
                console.error("Error fetching incidents:", error);
                loadingMessage.textContent = "Error loading incidents.";
                loadingMessage.classList.remove('hidden');
            });
        };

        // --- Main Logic & Event Listeners ---
        window.onload = async () => {
            if (isInitialized) return;
            isInitialized = true;
            
            if (Object.keys(firebaseConfig).length > 0) {
                app = initializeApp(firebaseConfig);
                db = getFirestore(app);
                auth = getAuth(app);
                setLogLevel('debug');
            } else {
                console.error("Firebase config is not available. Firestore will not be functional.");
            }

            if (auth) {
                onAuthStateChanged(auth, user => {
                    if (user) {
                        userId = user.uid;
                        isAuthReady = true;
                        getEl('user-id-display').textContent = `User ID: ${userId}`;
                        console.log(`User authenticated with ID: ${userId}`);
                        
                        // Setup live location listener for Tourist App
                        const q = query(collection(db, `artifacts/${appId}/users/${userId}/live_location`));
                        onSnapshot(q, (snapshot) => {
                            const logList = getEl('location-log');
                            logList.innerHTML = '';
                            const docs = [];
                            snapshot.forEach(doc => { docs.push(doc.data()); });
                            docs.sort((a, b) => {
                                const timeA = a.timestamp?.seconds || 0;
                                const timeB = b.timestamp?.seconds || 0;
                                return timeB - timeA;
                            });
                            docs.forEach(data => {
                                const li = document.createElement('li');
                                const date = data.timestamp ? new Date(data.timestamp.seconds * 1000).toLocaleString() : 'N/A';
                                li.textContent = `Time: ${date}, Lat: ${data.latitude.toFixed(4)}, Lng: ${data.longitude.toFixed(4)}`;
                                li.className = 'py-1 text-sm text-gray-700 border-b border-gray-200 last:border-0';
                                logList.appendChild(li);
                            });
                        });

                    } else {
                        console.log("User signed out or auth state changed.");
                    }
                });
                await authenticateUser();
            } else {
                console.log("Firebase Auth is not available. Running in non-persistent mode.");
                userId = crypto.randomUUID();
                getEl('user-id-display').textContent = `User ID: ${userId}`;
                isAuthReady = true;
            }
            
            hideElement('loading-spinner');
            showScreen('start-screen');
            
            // --- Event Listeners for Tourist App Flow ---
            getEl('phone-input').addEventListener('input', (e) => {
                if (e.target.value.length >= 10) {
                    showElement('send-otp-btn');
                    hideElement('otp-info');
                } else {
                    hideElement('send-otp-btn');
                    showElement('otp-info');
                }
            });
            getEl('destination-input').addEventListener('input', (e) => {
                const city = e.target.value;
                const isCustom = getEl('custom-location-checkbox').checked;
                if (!isCustom && city.length > 2) {
                    populateFamousPlaces(city);
                } else if (!isCustom) {
                    hideElement('place-selection-section');
                }
            });
            getEl('custom-location-checkbox').addEventListener('change', (e) => {
                if (e.target.checked) {
                    hideElement('place-selection-section');
                    hideElement('destination-input');
                    showElement('custom-destination-section');
                } else {
                    showElement('destination-input');
                    hideElement('custom-destination-section');
                    const city = getEl('destination-input').value;
                    if (city.length > 2) {
                        populateFamousPlaces(city);
                    }
                }
            });
            getEl('send-otp-btn').addEventListener('click', () => {
                const phoneNumber = getEl('phone-input').value;
                if (!phoneNumber) {
                    showNotificationModal('Error', 'Please enter your phone number.', 'bg-red-500');
                    return;
                }
                generatedOTP = Math.floor(100000 + Math.random() * 900000).toString();
                console.log(`[SIMULATED OTP] Your OTP is: ${generatedOTP}`);
                showNotificationModal('OTP Sent', `A 6-digit OTP has been sent to your console for demonstration. Your OTP is: ${generatedOTP}`, 'bg-blue-500');
                showElement('otp-section');
            });
            getEl('verify-otp-btn').addEventListener('click', () => {
                const otp = getEl('otp-input').value;
                if (otp === generatedOTP) {
                    showNotificationModal('Success', 'OTP verified successfully!', 'bg-green-500');
                    isOtpVerified = true;
                    hideElement('send-otp-btn');
                    hideElement('otp-section');
                    showElement('emergency-contact-section');
                    showElement('destination-section');
                    showElement('start-monitoring-btn');
                } else {
                    showNotificationModal('Error', 'Invalid OTP. Please try again.', 'bg-red-500');
                }
            });
            getEl('user-form').addEventListener('submit', async (e) => {
                e.preventDefault();
                const name = getEl('name-input').value;
                const phoneNumber = getEl('phone-input').value;
                const emergencyContactName = getEl('emergency-name-input').value;
                const emergencyContactPhone = getEl('emergency-phone-input').value;
                const isSimulated = getEl('simulated-mode-checkbox').checked;
                const isCustomLocation = getEl('custom-location-checkbox').checked;
                let destination = '';
                let destinationPlace = '';
                let simulatedLocations = null;

                if (isCustomLocation) {
                    destination = getEl('custom-location-input').value;
                    if (!destination) {
                        showNotificationModal('Error', 'Please enter your custom destination.', 'bg-red-500');
                        return;
                    }
                } else {
                    destination = getEl('destination-input').value;
                    destinationPlace = getEl('famous-place-select').value;
                    const cityData = famousPlacesData[destination.toLowerCase()];
                    if (!cityData || destinationPlace === "Choose a place...") {
                        showNotificationModal('Error', 'Please enter a valid city and select a place.', 'bg-red-500');
                        return;
                    }
                    simulatedLocations = cityData.simulatedLocations;
                }

                if (!name || !phoneNumber || !emergencyContactName || !emergencyContactPhone || !isOtpVerified) {
                    showNotificationModal('Error', 'Please fill in all required fields and verify your OTP.', 'bg-red-500');
                    return;
                }
                const userData = {
                    name,
                    phoneNumber,
                    emergencyContactName,
                    emergencyContactPhone,
                    destination,
                    destinationPlace,
                    isSimulated,
                    isCustomLocation,
                    customLocation: isCustomLocation ? destination : '',
                    appId: appId,
                    userId: userId,
                    createdAt: new Date().toISOString()
                };
                await saveUserData(userData);
                startGeolocationWatch(simulatedLocations);
            });
            getEl('stop-tracking-button').addEventListener('click', () => {
                stopGeolocationWatch();
                resetForm();
                showScreen('start-screen');
            });
            getEl('sos-button').addEventListener('click', handleSOS);
            getEl('view-log-button').addEventListener('click', () => {
                getEl('location-history-modal').classList.remove('hidden');
            });
            getEl('close-modal-button').addEventListener('click', () => {
                getEl('notification-modal').classList.add('hidden');
            });
            getEl('close-log-modal-button').addEventListener('click', () => {
                getEl('location-history-modal').classList.add('hidden');
            });
            
            // --- NEW: Event Listeners for Central Login and Admin Flow ---
            getEl('tourist-login-btn').addEventListener('click', () => {
                showScreen('form-screen');
                loadUserData(); // Load user data if it exists
            });
            getEl('admin-login-link').addEventListener('click', (e) => {
                e.preventDefault();
                showScreen('admin-login-screen');
            });
            getEl('admin-login-form').addEventListener('submit', (e) => {
                e.preventDefault();
                const username = getEl('admin-username').value.toLowerCase();
                const password = getEl('admin-password').value;
                if (ADMIN_CREDENTIALS[username] === password) {
                    currentAdminRole = username.charAt(0).toUpperCase() + username.slice(1);
                    getEl('admin-role-display').textContent = currentAdminRole;
                    showNotificationModal('Success', `${currentAdminRole} login successful!`, 'bg-green-500');
                    showScreen('admin-dashboard-screen');
                    setupAdminDashboard();
                } else {
                    showNotificationModal('Error', 'Invalid credentials.', 'bg-red-500');
                }
            });
            getEl('admin-back-btn').addEventListener('click', () => {
                showScreen('start-screen');
            });
            
            getEl('admin-logout-btn').addEventListener('click', () => {
                showScreen('start-screen');
            });
        };
    </script>
    
    <!-- Loading Spinner -->
    <div id="loading-spinner" class="flex flex-col items-center justify-center p-8 bg-white rounded-xl shadow-lg">
        <svg class="animate-spin -ml-1 mr-3 h-10 w-10 text-gray-500" xmlns="http://www.w3.org/2000/svg" fill="none" viewBox="0 0 24 24">
            <circle class="opacity-25" cx="12" cy="12" r="10" stroke="currentColor" stroke-width="4"></circle>
            <path class="opacity-75" fill="currentColor" d="M4 12a8 8 0 018-8V0C5.373 0 0 5.373 0 12h4zm2 5.291A7.962 7.962 0 014 12H0c0 3.042 1.135 5.824 3 7.938l3-2.647z"></path>
        </svg>
        <p class="mt-4 text-gray-700">Initializing app...</p>
    </div>

    <!-- Start Screen: User or Admin Login -->
    <div class="container bg-white rounded-3xl shadow-2xl p-6 md:p-10 flex flex-col items-center max-w-lg" id="start-screen">
        <h1 class="text-3xl md:text-4xl font-extrabold text-gray-800 mb-4 text-center">Welcome to Tourist Safety</h1>
        <p class="text-gray-500 text-center mb-8">Choose your role to get started.</p>
        <button id="tourist-login-btn" class="w-full bg-blue-600 text-white font-bold py-3 rounded-xl shadow-md hover:bg-blue-700 transition-all duration-200 transform hover:scale-105 mb-4">
            I'm a Tourist
        </button>
        <a href="#" id="admin-login-link" class="text-center text-blue-600 hover:underline transition-colors duration-200">Are you an Admin? Click here.</a>
        <div id="user-id-display" class="text-sm text-gray-400 mt-4 text-center break-all"></div>
    </div>
    
    <!-- Admin Login Screen -->
    <div class="container bg-white rounded-3xl shadow-2xl p-6 md:p-10 flex flex-col items-center max-w-lg hidden" id="admin-login-screen">
        <h1 class="text-3xl md:text-4xl font-extrabold text-gray-800 mb-4 text-center">Admin Login</h1>
        <p class="text-gray-500 text-center mb-8">Enter your credentials to access the dashboard. <br><span class="text-xs text-gray-400">Usernames: police, hospital, touristauthority (Password: password123)</span></p>
        <form id="admin-login-form" class="w-full">
            <div class="mb-5">
                <label for="admin-username" class="block text-gray-700 font-semibold mb-2">Username</label>
                <input type="text" id="admin-username" placeholder="e.g., police" required class="w-full px-4 py-3 border border-gray-300 rounded-xl focus:ring-2 focus:ring-blue-500 focus:border-transparent transition-all duration-200">
            </div>
            <div class="mb-5">
                <label for="admin-password" class="block text-gray-700 font-semibold mb-2">Password</label>
                <input type="password" id="admin-password" placeholder="e.g., password123" required class="w-full px-4 py-3 border border-gray-300 rounded-xl focus:ring-2 focus:ring-blue-500 focus:border-transparent transition-all duration-200">
            </div>
            <button type="submit" class="w-full bg-blue-600 text-white font-bold py-3 rounded-xl shadow-md hover:bg-blue-700 transition-all duration-200 transform hover:scale-105 mb-4">
                Login
            </button>
            <button type="button" id="admin-back-btn" class="w-full bg-gray-400 text-white font-bold py-3 rounded-xl shadow-md hover:bg-gray-500 transition-all duration-200 transform hover:scale-105">
                Back
            </button>
        </form>
    </div>

    <!-- Tourist Form -->
    <div class="container bg-white rounded-3xl shadow-2xl p-6 md:p-10 flex flex-col items-center max-w-lg hidden" id="form-screen">
        <h1 class="text-3xl md:text-4xl font-extrabold text-gray-800 mb-2 text-center">Travel Securely</h1>
        <p class="text-gray-500 text-center mb-8">Enter your details to get started.</p>
        <form id="user-form" class="w-full">
            <div class="mb-5">
                <label for="name-input" class="block text-gray-700 font-semibold mb-2">Your Full Name</label>
                <input type="text" id="name-input" placeholder="e.g., Jane Doe" required class="w-full px-4 py-3 border border-gray-300 rounded-xl focus:ring-2 focus:ring-blue-500 focus:border-transparent transition-all duration-200">
            </div>
            <div class="mb-5">
                <label for="phone-input" class="block text-gray-700 font-semibold mb-2">Your Phone Number</label>
                <div class="flex items-center space-x-2">
                    <input type="tel" id="phone-input" placeholder="e.g., +1234567890" required class="w-full px-4 py-3 border border-gray-300 rounded-xl focus:ring-2 focus:ring-blue-500 focus:border-transparent transition-all duration-200">
                    <button type="button" id="send-otp-btn" class="bg-blue-600 text-white font-bold py-3 px-4 rounded-xl hover:bg-blue-700 transition-all duration-200 whitespace-nowrap hidden">Send OTP</button>
                </div>
                <span id="otp-info" class="text-sm text-gray-500 mt-2 block">Enter your phone number to receive a demo OTP.</span>
            </div>
            <div id="otp-section" class="mb-5 hidden">
                <label for="otp-input" class="block text-gray-700 font-semibold mb-2">Enter OTP</label>
                <div class="flex items-center space-x-2">
                    <input type="tel" id="otp-input" placeholder="e.g., 123456" required class="w-full px-4 py-3 border border-gray-300 rounded-xl focus:ring-2 focus:ring-blue-500 focus:border-transparent transition-all duration-200">
                    <button type="button" id="verify-otp-btn" class="bg-green-600 text-white font-bold py-3 px-4 rounded-xl hover:bg-green-700 transition-all duration-200 whitespace-nowrap">Verify</button>
                </div>
            </div>
            <div id="emergency-contact-section" class="mb-5 hidden">
                <h2 class="text-2xl font-bold text-gray-800 my-4 text-center">Emergency Contact Details</h2>
                <div class="mb-5">
                    <label for="emergency-name-input" class="block text-gray-700 font-semibold mb-2">Emergency Contact Name</label>
                    <input type="text" id="emergency-name-input" placeholder="e.g., John Doe" required class="w-full px-4 py-3 border border-gray-300 rounded-xl focus:ring-2 focus:ring-blue-500 focus:border-transparent transition-all duration-200">
                </div>
                <div class="mb-5">
                    <label for="emergency-phone-input" class="block text-gray-700 font-semibold mb-2">Emergency Contact Phone</label>
                    <input type="tel" id="emergency-phone-input" placeholder="e.g., +1234567890" required class="w-full px-4 py-3 border border-gray-300 rounded-xl focus:ring-2 focus:ring-blue-500 focus:border-transparent transition-all duration-200">
                </div>
            </div>
            <div id="destination-section" class="mb-5 hidden">
                <h2 class="text-2xl font-bold text-gray-800 my-4 text-center">Where are you headed?</h2>
                <div class="mb-5 flex items-center space-x-2">
                    <input type="checkbox" id="custom-location-checkbox" class="rounded-lg text-blue-600 focus:ring-blue-500 h-5 w-5 border-gray-300">
                    <label for="custom-location-checkbox" class="text-gray-700 font-semibold">Traveling to a custom location?</label>
                </div>
                <div id="famous-location-group">
                    <div class="mb-5" id="destination-input-group">
                        <label for="destination-input" class="block text-gray-700 font-semibold mb-2">Destination City</label>
                        <input type="text" id="destination-input" placeholder="e.g., Hyderabad" required class="w-full px-4 py-3 border border-gray-300 rounded-xl focus:ring-2 focus:ring-blue-500 focus:border-transparent transition-all duration-200">
                    </div>
                    <div id="place-selection-section" class="mb-5 hidden">
                        <h2 class="text-xl font-bold text-gray-800 my-4 text-center">Which place are you traveling to?</h2>
                        <label for="famous-place-select" class="block text-gray-700 font-semibold mb-2">Famous Places in City</label>
                        <select id="famous-place-select" class="w-full px-4 py-3 border border-gray-300 rounded-xl focus:ring-2 focus:ring-blue-500 focus:border-transparent transition-all duration-200">
                            <option disabled selected value> -- select a place -- </option>
                        </select>
                    </div>
                </div>
                <div id="custom-destination-section" class="mb-5 hidden">
                    <label for="custom-location-input" class="block text-gray-700 font-semibold mb-2">Custom Destination</label>
                    <input type="text" id="custom-location-input" placeholder="e.g., 123 Main Street, New York" class="w-full px-4 py-3 border border-gray-300 rounded-xl focus:ring-2 focus:ring-blue-500 focus:border-transparent transition-all duration-200">
                </div>
            </div>
            <div class="mb-5 flex items-center justify-start gap-2">
                <input type="checkbox" id="simulated-mode-checkbox" class="rounded-lg text-blue-600 focus:ring-blue-500 h-5 w-5 border-gray-300">
                <label for="simulated-mode-checkbox" class="text-gray-700 font-semibold">Simulated Location Mode (for demo)</label>
            </div>
            <button type="submit" id="start-monitoring-btn" class="w-full bg-blue-600 text-white font-bold py-3 rounded-xl shadow-md hover:bg-blue-700 transition-all duration-200 transform hover:scale-105 hidden">
                Start Safety Monitoring
            </button>
        </form>
    </div>

    <!-- Tourist Dashboard -->
    <div class="container bg-white rounded-3xl shadow-2xl p-6 md:p-10 flex flex-col items-center max-w-lg hidden" id="app-screen">
        <h1 class="text-3xl md:text-4xl font-extrabold text-gray-800 mb-2 text-center">Safety Dashboard</h1>
        <p class="text-gray-500 text-center mb-8">Monitoring your location for potential risks. </p>
        <div id="status-indicator" class="p-4 rounded-xl text-center font-bold mb-4 bg-green-500 text-white safe-zone">
            You are in a safe area.
        </div>
        <p id="user-location" class="text-gray-700 text-center mb-4 text-sm break-all"></p>
        <div id="timer-display" class="hidden mb-4 p-2 bg-yellow-100 rounded-xl text-yellow-700 font-semibold text-center">
            <p id="timer-text">Escalation timer...</p>
        </div>
        <button id="stop-tracking-button" class="w-full bg-red-600 text-white font-bold py-3 rounded-xl shadow-md hover:bg-red-700 transition-all duration-200 transform hover:scale-105 mb-4">
            Stop Monitoring
        </button>
        <button id="sos-button" class="w-full bg-red-800 text-white font-bold py-3 rounded-xl shadow-md hover:bg-red-900 transition-all duration-200 transform hover:scale-105 mb-4">
            SOS - Emergency
        </button>
        <button id="view-log-button" class="w-full bg-gray-600 text-white font-bold py-3 rounded-xl shadow-md hover:bg-gray-700 transition-all duration-200 transform hover:scale-105">
            View Live Location Log
        </button>
    </div>
    
    <!-- Admin Dashboard -->
    <div class="max-w-7xl mx-auto bg-white rounded-xl shadow-lg p-6 md:p-10 hidden" id="admin-dashboard-screen">
        <header class="mb-8 text-center">
            <h1 class="text-3xl md:text-4xl font-bold text-gray-800">Emergency Services Dashboard - <span id="admin-role-display" class="capitalize"></span></h1>
            <p class="text-gray-500 mt-2">Real-time emergency requests</p>
        </header>
        <main class="grid grid-cols-1 md:grid-cols-2 gap-8">
            <div class="bg-gray-50 p-6 rounded-lg border border-gray-200">
                <h2 class="text-2xl font-semibold text-gray-700 mb-4">Active Incidents</h2>
                <div id="incidentList" class="space-y-4">
                    <p id="loadingMessage" class="text-gray-400 text-center">Loading...</p>
                    <p id="noIncidents" class="hidden text-gray-400 text-center">No active incidents.</p>
                </div>
                <button id="admin-logout-btn" class="mt-8 w-full bg-gray-600 text-white font-bold py-3 rounded-xl shadow-md hover:bg-gray-700 transition-all duration-200 transform hover:scale-105">
                    Logout
                </button>
            </div>
            <div class="bg-gray-50 p-6 rounded-lg border border-gray-200">
                <h2 class="text-2xl font-semibold text-gray-700 mb-4">Location Details</h2>
                <div id="mapContainer" class="hidden">
                    <div id="map"></div>
                </div>
                <div id="mapPlaceholder" class="flex justify-center items-center h-[500px] text-gray-400">
                    <p>Select an incident to view location on the map.</p>
                </div>
            </div>
        </main>
    </div>
    
    <!-- Notification Modal -->
    <div id="notification-modal" class="hidden fixed inset-0 bg-gray-600 bg-opacity-50 flex items-center justify-center p-4 z-50">
        <div id="notification-modal-content" class="bg-white p-6 rounded-xl shadow-lg border-t-8 border-red-500 max-w-md w-full relative">
            <h3 id="notification-title" class="text-2xl font-bold mb-2 text-gray-800"></h3>
            <p id="notification-message" class="text-gray-600"></p>
            <button id="close-modal-button" class="mt-4 w-full bg-blue-600 text-white font-bold py-2 rounded-xl hover:bg-blue-700 transition-all">OK</button>
        </div>
    </div>
    
    <!-- Location History Modal -->
    <div id="location-history-modal" class="hidden fixed inset-0 bg-gray-600 bg-opacity-50 flex items-center justify-center p-4 z-50">
        <div class="bg-white p-6 rounded-xl shadow-lg max-w-lg w-full relative">
            <h3 class="text-2xl font-bold mb-4 text-gray-800">Live Location Log</h3>
            <p class="text-gray-600 mb-4 text-sm">This shows a live feed of location data as it would be sent to a server.</p>
            <ul id="location-log" class="divide-y divide-gray-200">
                <!-- Location data will be inserted here by JavaScript -->
            </ul>
            <button id="close-log-modal-button" class="mt-4 w-full bg-blue-600 text-white font-bold py-2 rounded-xl hover:bg-blue-700 transition-all">Close</button>
        </div>
    </div>
</body>
</html>
