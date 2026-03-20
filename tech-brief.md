# Tech Brief: Open-Meteo Weather Widget Implementation

## Overview
This document outlines the implementation of a weather widget using the Open-Meteo API for a static HTML/CSS website hosted on Vercel.

## Technology Stack
- **Frontend**: HTML5, CSS3, Vanilla JavaScript
- **API**: Open-Meteo (https://open-meteo.com/)
- **Hosting**: Vercel
- **Geolocation**: Browser Geolocation API

## Open-Meteo API Analysis

### Key Advantages
- **No API Key Required**: Completely free, no registration needed
- **No Rate Limits**: Unlimited requests for non-commercial use
- **CORS Enabled**: Can be called directly from browser
- **Comprehensive Data**: Weather, forecasts, historical data
- **Fast Response**: Typically <100ms response times
- **Multiple Units**: Celsius/Fahrenheit, km/h, mph, etc.

### API Endpoints We'll Use
```
Base URL: https://api.open-meteo.com/v1/forecast
```

**Current Weather Parameters:**
- `latitude` & `longitude`: User coordinates
- `current_weather=true`: Include current conditions
- `temperature_unit`: fahrenheit/celsius
- `windspeed_unit`: mph/kmh
- `timezone`: auto (uses location timezone)

**Optional Enhancements:**
- `daily`: Daily forecasts
- `hourly`: Hourly forecasts
- `precipitation_unit`: inch/mm

### Sample API Response
```json
{
  "latitude": 47.6,
  "longitude": -122.33,
  "current_weather": {
    "temperature": 68.2,
    "windspeed": 8.1,
    "winddirection": 245,
    "weathercode": 3,
    "time": "2026-03-20T18:00"
  }
}
```

## Architecture Design

### Component Structure
```
Weather Widget
├── Location Detection
├── API Data Fetching
├── Data Processing
├── UI Rendering
└── Error Handling
```

### Data Flow
1. **Page Load** → Request user location
2. **Location Granted** → Fetch weather data
3. **API Response** → Process and display data
4. **Error States** → Show fallback content

### Weather Code Mapping
Open-Meteo uses WMO weather codes. We'll map these to user-friendly descriptions and icons:
- `0`: Clear sky ☀️
- `1-3`: Partly cloudy ⛅
- `45,48`: Fog 🌫️
- `51-67`: Rain 🌧️
- `71-86`: Snow ❄️
- `95-99`: Thunderstorm ⛈️

---

# Step-by-Step Implementation Plan

## Phase 1: Basic Setup (30 minutes)

### Step 1: Create Widget HTML Structure
Add to your existing HTML file:

```html
<!-- Add this where you want the widget to appear -->
<div id="weather-widget" class="weather-container">
    <div class="weather-header">
        <h3>Current Weather</h3>
    </div>
    <div id="weather-content" class="weather-loading">
        <div class="loading-spinner"></div>
        <p>Loading weather data...</p>
    </div>
</div>
```

### Step 2: Add Base CSS Styles
Add to your CSS file:

```css
.weather-container {
    background: linear-gradient(135deg, #74b9ff, #0984e3);
    color: white;
    border-radius: 15px;
    padding: 20px;
    max-width: 350px;
    margin: 20px auto;
    box-shadow: 0 8px 32px rgba(0, 0, 0, 0.1);
    font-family: -apple-system, BlinkMacSystemFont, 'Segoe UI', Roboto, sans-serif;
}

.weather-header h3 {
    margin: 0 0 15px 0;
    font-size: 1.2em;
    font-weight: 600;
}

.weather-loading {
    text-align: center;
    padding: 20px 0;
}

.loading-spinner {
    width: 30px;
    height: 30px;
    border: 3px solid rgba(255, 255, 255, 0.3);
    border-top: 3px solid white;
    border-radius: 50%;
    animation: spin 1s linear infinite;
    margin: 0 auto 10px;
}

@keyframes spin {
    0% { transform: rotate(0deg); }
    100% { transform: rotate(360deg); }
}
```

## Phase 2: Core JavaScript Implementation (45 minutes)

### Step 3: Create Weather Service Module
Add before closing `</body>` tag:

```html
<script>
class WeatherService {
    constructor() {
        this.baseURL = 'https://api.open-meteo.com/v1/forecast';
        this.weatherCodes = {
            0: { description: 'Clear sky', icon: '☀️' },
            1: { description: 'Mainly clear', icon: '🌤️' },
            2: { description: 'Partly cloudy', icon: '⛅' },
            3: { description: 'Overcast', icon: '☁️' },
            45: { description: 'Fog', icon: '🌫️' },
            48: { description: 'Depositing rime fog', icon: '🌫️' },
            51: { description: 'Light drizzle', icon: '🌦️' },
            53: { description: 'Moderate drizzle', icon: '🌧️' },
            55: { description: 'Dense drizzle', icon: '🌧️' },
            61: { description: 'Slight rain', icon: '🌧️' },
            63: { description: 'Moderate rain', icon: '🌧️' },
            65: { description: 'Heavy rain', icon: '⛈️' },
            71: { description: 'Slight snow', icon: '🌨️' },
            73: { description: 'Moderate snow', icon: '❄️' },
            75: { description: 'Heavy snow', icon: '❄️' },
            95: { description: 'Thunderstorm', icon: '⛈️' }
        };
    }

    async getCurrentWeather(latitude, longitude) {
        const params = new URLSearchParams({
            latitude: latitude,
            longitude: longitude,
            current_weather: 'true',
            temperature_unit: 'fahrenheit',
            windspeed_unit: 'mph',
            timezone: 'auto'
        });

        const response = await fetch(`${this.baseURL}?${params}`);
        
        if (!response.ok) {
            throw new Error(`Weather API error: ${response.status}`);
        }

        return await response.json();
    }

    getWeatherDescription(code) {
        return this.weatherCodes[code] || { 
            description: 'Unknown', 
            icon: '🌡️' 
        };
    }
}
</script>
```

### Step 4: Create Location Service
```html
<script>
class LocationService {
    async getCurrentPosition() {
        return new Promise((resolve, reject) => {
            if (!navigator.geolocation) {
                reject(new Error('Geolocation is not supported'));
                return;
            }

            const options = {
                enableHighAccuracy: true,
                timeout: 10000,
                maximumAge: 300000 // 5 minutes cache
            };

            navigator.geolocation.getCurrentPosition(
                resolve,
                reject,
                options
            );
        });
    }

    async getCityName(latitude, longitude) {
        try {
            // Using a free reverse geocoding service
            const response = await fetch(
                `https://api.bigdatacloud.net/data/reverse-geocode-client?latitude=${latitude}&longitude=${longitude}&localityLanguage=en`
            );
            const data = await response.json();
            return data.city || data.locality || 'Unknown Location';
        } catch (error) {
            console.warn('Could not get city name:', error);
            return 'Your Location';
        }
    }
}
</script>
```

## Phase 3: Widget Controller (30 minutes)

### Step 5: Create Main Widget Controller
```html
<script>
class WeatherWidget {
    constructor() {
        this.weatherService = new WeatherService();
        this.locationService = new LocationService();
        this.container = document.getElementById('weather-content');
    }

    async init() {
        try {
            this.showLoading();
            
            // Get user location
            const position = await this.locationService.getCurrentPosition();
            const { latitude, longitude } = position.coords;
            
            // Get weather data and city name in parallel
            const [weatherData, cityName] = await Promise.all([
                this.weatherService.getCurrentWeather(latitude, longitude),
                this.locationService.getCityName(latitude, longitude)
            ]);
            
            this.displayWeather(weatherData, cityName);
            
        } catch (error) {
            this.showError(error);
        }
    }

    showLoading() {
        this.container.innerHTML = `
            <div class="weather-loading">
                <div class="loading-spinner"></div>
                <p>Loading weather data...</p>
            </div>
        `;
    }

    displayWeather(data, cityName) {
        const weather = data.current_weather;
        const weatherInfo = this.weatherService.getWeatherDescription(weather.weathercode);
        
        this.container.innerHTML = `
            <div class="weather-display">
                <div class="location">
                    📍 ${cityName}
                </div>
                <div class="temperature">
                    ${Math.round(weather.temperature)}°F
                </div>
                <div class="condition">
                    <span class="weather-icon">${weatherInfo.icon}</span>
                    <span class="weather-desc">${weatherInfo.description}</span>
                </div>
                <div class="details">
                    <div class="detail-item">
                        <span class="label">Wind:</span>
                        <span class="value">${weather.windspeed} mph</span>
                    </div>
                    <div class="detail-item">
                        <span class="label">Updated:</span>
                        <span class="value">${new Date(weather.time).toLocaleTimeString()}</span>
                    </div>
                </div>
            </div>
        `;
    }

    showError(error) {
        console.error('Weather widget error:', error);
        
        let errorMessage = 'Unable to load weather data';
        if (error.code === 1) {
            errorMessage = 'Location access denied. Please enable location services.';
        } else if (error.code === 2) {
            errorMessage = 'Location unavailable. Please try again.';
        } else if (error.code === 3) {
            errorMessage = 'Location request timed out. Please try again.';
        }

        this.container.innerHTML = `
            <div class="weather-error">
                <div class="error-icon">⚠️</div>
                <p>${errorMessage}</p>
                <button onclick="weatherWidget.init()" class="retry-btn">
                    Try Again
                </button>
            </div>
        `;
    }
}

// Initialize widget when page loads
let weatherWidget;
document.addEventListener('DOMContentLoaded', () => {
    weatherWidget = new WeatherWidget();
    weatherWidget.init();
});
</script>
```

## Phase 4: Enhanced Styling (20 minutes)

### Step 6: Add Complete CSS Styles
Add these styles to your CSS:

```css
.weather-display {
    text-align: center;
}

.location {
    font-size: 0.9em;
    opacity: 0.9;
    margin-bottom: 10px;
}

.temperature {
    font-size: 3em;
    font-weight: bold;
    margin: 10px 0;
    text-shadow: 0 2px 4px rgba(0, 0, 0, 0.1);
}

.condition {
    display: flex;
    align-items: center;
    justify-content: center;
    gap: 10px;
    margin: 15px 0;
}

.weather-icon {
    font-size: 1.5em;
}

.weather-desc {
    font-size: 1.1em;
    font-weight: 500;
}

.details {
    display: flex;
    justify-content: space-between;
    margin-top: 20px;
    padding-top: 15px;
    border-top: 1px solid rgba(255, 255, 255, 0.2);
}

.detail-item {
    text-align: center;
}

.detail-item .label {
    display: block;
    font-size: 0.8em;
    opacity: 0.8;
    margin-bottom: 2px;
}

.detail-item .value {
    font-weight: 600;
}

.weather-error {
    text-align: center;
    padding: 20px 0;
}

.error-icon {
    font-size: 2em;
    margin-bottom: 10px;
}

.retry-btn {
    background: rgba(255, 255, 255, 0.2);
    color: white;
    border: 1px solid rgba(255, 255, 255, 0.3);
    padding: 8px 16px;
    border-radius: 20px;
    cursor: pointer;
    margin-top: 10px;
    transition: background 0.3s ease;
}

.retry-btn:hover {
    background: rgba(255, 255, 255, 0.3);
}

/* Responsive design */
@media (max-width: 480px) {
    .weather-container {
        margin: 10px;
        max-width: none;
    }
    
    .temperature {
        font-size: 2.5em;
    }
    
    .details {
        flex-direction: column;
        gap: 10px;
    }
}
```

## Phase 5: Testing & Deployment (15 minutes)

### Step 7: Test Locally
1. Open your HTML file in a browser
2. Allow location access when prompted
3. Verify weather data displays correctly
4. Test error states by denying location access

### Step 8: Deploy to Vercel
1. Commit your changes to git
2. Push to your repository
3. Vercel will automatically deploy
4. Test the live version

## Phase 6: Optional Enhancements

### Future Improvements
- Add hourly/daily forecasts
- Implement manual location search
- Add weather alerts
- Include more weather details (humidity, pressure, UV index)
- Add weather maps integration
- Implement dark/light theme toggle

### Performance Optimizations
- Add service worker for offline support
- Implement data caching
- Add loading skeletons
- Optimize for mobile performance

This implementation provides a robust, user-friendly weather widget that will work reliably on your Vercel-hosted static site!