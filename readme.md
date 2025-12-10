# Rideau Canal Skateway - Real-time Monitoring System


#  Project Overview

The Rideau Canal Skateway in Ottawa is the world's largest skating rink, attracting thousands of visitors daily. Ensuring skater safety requires continuous monitoring of ice conditions across multiple locations. This project implements a complete **real-time IoT data pipeline** using Azure cloud services to automate this monitoring process.


## Student Information

- **Name:** Aryan Rudani
- **Student ID:** 041171391
- **Course:** CST8916 - Remote Data and Real-time Applications
- **Semester:** Fall 2025

<br>
<br>

# Youtube Video Link
- https://youtu.be/EIGuOP0KjiI

<br>
<br>

# System Architecture

![alt text](<architecture/Screenshot 2025-12-09 at 3.00.22 PM.png>)

<br>
<br>

### Component Breakdown

| Component | Purpose | Technology |
|-----------|---------|------------|
| **IoT Sensors** | Generate telemetry data | Python simulation (3 devices) |
| **IoT Hub** | Device management & ingestion | Azure IoT Hub (F1 tier) |
| **Stream Analytics** | Real-time processing | Azure Stream Analytics (1 SU) |
| **Cosmos DB** | Fast query database | Azure Cosmos DB (NoSQL) |
| **Blob Storage** | Historical archive | Azure Blob Storage |
| **Dashboard** | User interface | Node.js + Express + Chart.js |
| **App Service** | Web hosting | Azure App Service (Linux) |

---
<br>
<br>

##  Technical Implementation

### IoT Sensor Simulation

**Technology:** Python 3.8+ with Azure IoT Device SDK

**Data Collection:**
- **Frequency:** Every 10 seconds
- **Locations:** 3 sensors (Dow's Lake, Fifth Avenue, NAC)
- **Metrics:** Ice thickness, surface temp, external temp, snow accumulation

**Data Format:**
```json
{
  "deviceId": "sensor-dows-lake",
  "location": "Dows Lake",
  "timestamp": "2024-12-09T15:30:45.123Z",
  "iceThickness": 31.25,
  "surfaceTemperature": -3.45,
  "externalTemperature": -5.67,
  "snowAccumulation": 2.18
}
```

**Key Implementation Details:**
- Realistic data with controlled random variations
- ISO 8601 timestamp format (required for Stream Analytics)
- Gradual value changes to simulate real physics
- Error handling for network failures

---
<br>
<br>

### Azure IoT Hub Configuration

**Service Tier:** F1 (Free)
- 8,000 messages/day limit
- 3 registered devices
- Device authentication via connection strings

**Message Flow:**
- 3 sensors × 6 messages/minute = **18 messages/minute**
- ~1,080 messages/hour
- Well within F1 tier limits

---

### Stream Analytics Processing

**Job Configuration:**
- **Streaming Units:** 1
- **Window Type:** 5-minute tumbling windows
- **Latency:** < 10 seconds end-to-end

```sql

WITH AggregatedData AS (
    SELECT
        location,
        System.Timestamp() AS windowEnd,
        AVG(iceThickness) AS avgIceThickness,
        MIN(iceThickness) AS minIceThickness,
        MAX(iceThickness) AS maxIceThickness,
        AVG(surfaceTemp) AS avgSurfaceTemp,
        MIN(surfaceTemp) AS minSurfaceTemp,
        MAX(surfaceTemp) AS maxSurfaceTemp,
        MAX(snowAccumulation) AS maxSnowAccumulation,
        AVG(externalTemp) AS avgExternalTemp,
        COUNT(*) AS readingCount
    FROM IoTHubInput
    TIMESTAMP BY timestamp
    GROUP BY location, TumblingWindow(minute, 5)
)

-- Step 2: Send to Cosmos DB with safety status
SELECT
    location,
    windowEnd,
    avgIceThickness,
    minIceThickness,
    maxIceThickness,
    avgSurfaceTemp,
    minSurfaceTemp,
    maxSurfaceTemp,
    maxSnowAccumulation,
    avgExternalTemp,
    readingCount,
    CASE
        WHEN avgIceThickness >= 30 AND avgSurfaceTemp <= -2 THEN 'Safe'
        WHEN avgIceThickness >= 25 AND avgSurfaceTemp <= 0 THEN 'Caution'
        ELSE 'Unsafe'
    END AS safetyStatus
INTO CosmosDBOutput
FROM AggregatedData

-- Step 3: Send to Blob Storage for archival (without safety status)
SELECT
    location,
    windowEnd,
    avgIceThickness,
    minIceThickness,
    maxIceThickness,
    avgSurfaceTemp,
    minSurfaceTemp,
    maxSurfaceTemp,
    maxSnowAccumulation,
    avgExternalTemp,
    readingCount
INTO BlobStorageOutput
FROM AggregatedData

```


<br>
<br>

**Safety Classification Rules:**
- **Safe:** Ice ≥ 30cm AND Surface Temp ≤ -2°C
- **Caution:** Ice ≥ 25cm AND Surface Temp ≤ 0°C
- **Unsafe:** All other conditions




### Data Storage

#### Cosmos DB (Fast Queries)

**Configuration:**
- **Database:** RideauCanalDB
- **Container:** SensorAggregations
- **Partition Key:** `/location`
- **Throughput:** 400 RU/s (manual)



**Document Schema:**
```json
{
  "id": "Dows Lake-2024-12-09-15-30",
  "location": "Dows Lake",
  "windowEnd": "2024-12-09T15:30:00.000Z",
  "avgIceThickness": 31.25,
  "minIceThickness": 30.12,
  "maxIceThickness": 32.45,
  "avgSurfaceTemperature": -3.45,
  "minSurfaceTemperature": -4.12,
  "maxSurfaceTemperature": -2.87,
  "maxSnowAccumulation": 2.34,
  "avgExternalTemperature": -5.67,
  "readingCount": 30,
  "safetyStatus": "Safe"
}
```

#### Blob Storage (Historical Archive)

**Configuration:**
- **Container:** historical-data
- **Path Pattern:** `aggregations/{date}/{time}`
- **Format:** JSON (line separated)

**Purpose:**
- Long-term data retention
- Cost-effective storage ($0.018/GB/month)
- Used for trend analysis and machine learning

---

### Web Dashboard

**Backend:** Node.js  + Express
- RESTful API with 5 endpoints
- Cosmos DB SDK for queries
- Environment-based configuration

**Frontend:** HTML5 + CSS3 + Vanilla JavaScript
- Chart.js for data visualization
- Auto-refresh every 30 seconds
- Responsive design (mobile-friendly)



**Dashboard Features:**
- System status banner (Safe/Caution/Unsafe)
- 3 location cards with real-time metrics
- Historical charts (ice thickness, temperatures)
- Location selector dropdown
- Last update timestamps

##  Setup Instructions

### Prerequisites

- Azure subscription (Free tier eligible)
- Python 3.8+ (for sensors)
- Node.js 20.x+ (for dashboard)
- Git

### Quick Start

1. **Clone all three repositories:**
```bash

https://github.com/ruda0008/rideau-canal-sensor-simulation
https://github.com/ruda0008/rideau-canal-dashboard
https://github.com/ruda0008/rideau-canal-monitoring
```

Contains Python code that simulates three IoT sensors sending telemetry data every 10 seconds.

2. **Deploy Azure infrastructure** (see detailed guides in component repos)

3. **Configure and run sensor simulator:**

-  Edit .env with your IoT Hub connection strings
python sensor_simulator.py
- Run the Python File


4. **Configure and run dashboard:**
```bash
cd rideau-canal-dashboard
npm install
# Edit .env with your Cosmos DB credentials
npm start
```

5. **Access dashboard:** http://localhost:3000

For detailed setup instructions, see the README files in each component repository.

---

##  Results & Analysis

### System Performance Metrics

| Metric | Value | Notes |
|--------|-------|-------|
| **End-to-end Latency** | < 15 seconds | Sensor → Dashboard |
| **Data Processing Rate** | 18 msg/min | 3 sensors × 6/min |
| **Aggregation Window** | 5 minutes | 30 readings → 1 record |
| **Query Response Time** | < 50ms | Cosmos DB queries |
| **Dashboard Refresh** | 30 seconds | Configurable |
| **Data Retention** | Unlimited | Blob Storage archive |

### Data Analysis (24-hour Sample)

**Ice Thickness Trends:**
- **Dow's Lake:** 30.5-32.8 cm (average: 31.6 cm)
- **Fifth Avenue:** 28.9-31.2 cm (average: 30.1 cm)
- **NAC:** 31.2-33.5 cm (average: 32.3 cm)


**Key Observations:**
1. NAC consistently shows thickest ice 
2. Fifth Avenue shows most temperature variation 
3. Safety status changes 3-4 times per day (temperature fluctuations)

### Cost Analysis

| Service | Tier | Monthly Cost |
|---------|------|-------------|
| IoT Hub | F1 Free | $0.00 |
| Stream Analytics | 1 SU (stopped after demo) | $0.00* |
| Cosmos DB | 400 RU/s | ~$23.00 |
| Blob Storage | Standard LRS | ~$0.50 |
| App Service | F1 Free | $0.00 |
| **TOTAL** | | **~$23.50/month** |



### Scalability Assessment

**Current Capacity:**
- 3 sensors → Can scale to 100+ with minimal changes
- 400 RU/s Cosmos DB → Supports ~200 queries/second
- F1 App Service → Can handle 50-100 concurrent users

**Bottlenecks:**
- Stream Analytics (1 SU) → Add more SUs for higher throughput
- Cosmos DB (400 RU/s) → Increase to 1000+ RU/s for production
- App Service (F1) → Upgrade to B1/S1 for more traffic

---

<br>
<br>

##  Challenges & Solutions

### Challenge 1: Stream Analytics Time Windows

**Problem:** Initially used hopping windows, which created overlapping aggregations. This caused duplicate data in Cosmos DB and confused the safety logic.

**Solution:** Switched to tumbling windows (`TumblingWindow(minute, 5)`) for non-overlapping 5-minute buckets. Used `System.Timestamp()` to capture window end time rather than message arrival time.

**Learning:** Understanding windowing semantics is critical for stream processing. Tumbling windows are best for discrete, non-overlapping aggregations.

---

### Challenge 2: Cosmos DB Query Performance

**Problem:** Initial queries scanned all locations when fetching data for a single location. Response times were 200-300ms, too slow for real-time dashboard.

**Solution:** 
- Chose `/location` as partition key
- Added `WHERE c.location = @location` clause to queries
- Used `ORDER BY c.windowEnd DESC` with `TOP 1` for latest data

**Result:** Query performance improved from 300ms to < 10ms (30x faster).

**Learning:** Partition key selection is the most important Cosmos DB design decision. Always query within a single partition when possible.

---


### Challenge 3: Stream Analytics Cosmos DB Connection

**Problem:** Stream Analytics job failed to start with "Owner resource does not exist" error.

**Solution:** 

- Verified exact database and container names (case-sensitive)
- Ensured container was created with correct partition key before starting job

**Learning:** Authentication and naming conventions matter. Stream Analytics requires explicit permissions and exact resource names.

<br>
<br>

#  Repository Links

This project is organized into **three separate repositories** following best practices for microservices architecture:

### 1. Sensor Simulation Repository
**URL:** https://github.com/ruda0008/rideau-canal-sensor-simulation

Contains Python code that simulates three IoT sensors sending telemetry data every 10 seconds.


### 2. Web Dashboard Repository
**URL:** https://github.com/ruda0008/rideau-canal-dashboard

Contains the Node.js backend API and HTML/CSS/JavaScript frontend for the monitoring dashboard.


### 3. Main Documentation Repository (This Repo)
**URL:** https://github.com/ruda0008/rideau-canal-monitoring

Contains comprehensive project documentation, architecture diagrams, screenshots, and Stream Analytics query.



---

## AI Tools Disclosure
### Tool Used
Claude AI by anthropic

- Purpose: 
  - Generating inital code later on combining with my code, Debugging assistance, concept explanations, documentation templates, boilerplate generation, Express routes, Chart.js setup

- My Original Work: 100% Designed by Me:

- System architecture decisions
- Database schema and partition key selection
- Stream Analytics query logic and safety classification rules
- Dashboard UI/UX layout
 
Manually Written:

- Stream Analytics query 
- Logic for safety status
- Cosmos DB query patterns
- Frontend state management

Modified/Enhanced by  AI:

- Express route handlers (simplified error handling)
- Chart.js configuration (customized colors and options)
- CSS responsive design 
- Backend for Dashboard
- Temperature data Simulation in python 

