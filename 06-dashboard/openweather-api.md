# OpenWeatherMap API Integration

## OpenWeatherMap Registration

Before using the API:

1. Register account at:
https://openweathermap.org/

2. Generate API key:
Account → API keys

3. Wait for API key activation.
New API keys may need several minutes or longer before they start working.

---

# Goal

Integrate external weather API into Home Assistant.

---

# What I learned

- what API is
- HTTP requests
- JSON response
- API key usage
- API endpoints
- current vs forecast API
- Home Assistant weather entities

---

# OpenWeather API Example

## City request

```text
https://api.openweathermap.org/data/2.5/weather?q=Poprad&appid=API_KEY&units=metric
```

## GPS coordinates request

```text
https://api.openweathermap.org/data/2.5/weather?lat=49.042825&lon=20.293993&appid=API_KEY&units=metric
```

## JSON Example

```JSON
{"coord":{"lon":20.294,"lat":49.0428},"weather":[{"id":802,"main":"Clouds","description":"scattered clouds","icon":"03d"}],"base":"stations","main":
{"temp":19.05,"feels_like":17.98,"temp_min":16.96,"temp_max":20.68,"pressure":1019,"humidity":37,"sea_level":1019,"grnd_level":919},"visibility":10000,"wind":
{"speed":5.81,"deg":16,"gust":8.94},"clouds":{"all":40},"dt":1779285698,"sys":
{"type":1,"id":7053,"country":"SK","sunrise":1779245437,"sunset":1779301220},"timezone":7200,"id":723846,"name":"Poprad","cod":200}
```
---

# Important Concepts

## API
Communication between systems/services.
## Endpoint
Specific API function/path.
## JSON
Structured data format similar to Python dictionaries.
## API Key
Authentication token for API access.

---

# Home Assistant Integration

Path:
Settings → Devices & Services → Add Integration → OpenWeatherMap

Mode used:
- current

Reason:
- v3.0 initially returned invalid API key
- current mode worked immediately

---

# Notes
- new API keys may need activation time
- current mode provides only current weather
- v3.0/forecast provides multi-day forecast
