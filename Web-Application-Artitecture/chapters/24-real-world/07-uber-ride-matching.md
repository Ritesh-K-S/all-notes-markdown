# How Uber/Ola Handles Real-Time Location & Matching

> **What you'll learn**: How ride-hailing platforms match millions of riders to nearby drivers in under 10 seconds, process billions of GPS updates per day, calculate real-time ETAs, and handle dynamic pricing — all while the entire system is constantly moving.

---

## Real-Life Analogy

Imagine you're a **traffic controller for every taxi in a city** — but the city has 10 million people and 500,000 taxis:

- Every taxi radios its location **every 4 seconds** (that's 125,000 location updates per second!)
- When someone raises their hand for a taxi, you need to find the **nearest available one** in under 5 seconds
- You need to predict **how long it'll take** the taxi to reach them (considering traffic, road closures, one-way streets)
- During rush hour, you increase prices to get more taxis on the road (**surge pricing**)
- You track every ride in real-time so the passenger can see the taxi approaching on a map
- You do all of this across **70 countries and 10,000+ cities simultaneously**

This is Uber's engineering challenge — a massive real-time geospatial computing problem.

---

## Core Concept Explained Step-by-Step

### Step 1: The Location Ingestion Pipeline

Every active Uber/Ola driver sends GPS coordinates every 4 seconds:

```
┌─────────────────────────────────────────────────────────────────┐
│           LOCATION UPDATE PIPELINE                                │
├─────────────────────────────────────────────────────────────────┤
│                                                                   │
│  5 million active drivers × 1 update every 4 seconds             │
│  = 1.25 MILLION location updates per second                      │
│                                                                   │
│  ┌──────────┐  ┌──────────┐  ┌──────────┐                       │
│  │ Driver 1 │  │ Driver 2 │  │ Driver N │  (5M drivers)         │
│  │ GPS: lat, │  │ GPS: lat,│  │ GPS: lat,│                       │
│  │ lng, time │  │ lng, time│  │ lng, time│                       │
│  └─────┬────┘  └─────┬────┘  └─────┬────┘                       │
│        │              │              │                             │
│        └──────────────┼──────────────┘                            │
│                       │                                            │
│                       ▼                                            │
│  ┌─────────────────────────────────────────────────────────┐     │
│  │                 KAFKA CLUSTER                             │     │
│  │  Topic: driver-locations                                 │     │
│  │  Throughput: 1M+ events/sec                              │     │
│  │  Partitioned by city/geohash                             │     │
│  └────────────────────────┬────────────────────────────────┘     │
│                           │                                       │
│              ┌────────────┼────────────┐                         │
│              ▼            ▼            ▼                         │
│  ┌────────────────┐ ┌──────────┐ ┌──────────────┐              │
│  │ Supply Index   │ │   ETA    │ │  Analytics   │              │
│  │ (Where are     │ │  Service │ │  Pipeline    │              │
│  │  drivers NOW?) │ │          │ │              │              │
│  └────────────────┘ └──────────┘ └──────────────┘              │
│                                                                   │
└─────────────────────────────────────────────────────────────────┘
```

### Step 2: Geospatial Indexing — Finding Nearby Drivers

When a rider requests a ride, the system needs to find drivers near them. This requires a **spatial index**:

```
┌─────────────────────────────────────────────────────────────────┐
│              GEOSPATIAL INDEXING (Geohash / H3)                   │
├─────────────────────────────────────────────────────────────────┤
│                                                                   │
│  The Earth's surface is divided into a GRID of cells:           │
│                                                                   │
│  ┌─────┬─────┬─────┬─────┬─────┬─────┐                         │
│  │     │     │     │  🚗 │     │     │                         │
│  │ a1  │ a2  │ a3  │ a4  │ a5  │ a6  │                         │
│  ├─────┼─────┼─────┼─────┼─────┼─────┤                         │
│  │     │     │ 🚗  │     │     │     │                         │
│  │ b1  │ b2  │ b3  │ b4  │ b5  │ b6  │                         │
│  ├─────┼─────┼─────┼─────┼─────┼─────┤                         │
│  │     │ 🚗  │     │ 📍  │ 🚗  │     │                         │
│  │ c1  │ c2  │ c3  │ c4  │ c5  │ c6  │                         │
│  ├─────┼─────┼─────┼─────┼─────┼─────┤                         │
│  │     │     │     │     │ 🚗  │     │                         │
│  │ d1  │ d2  │ d3  │ d4  │ d5  │ d6  │                         │
│  └─────┴─────┴─────┴─────┴─────┴─────┘                         │
│                                                                   │
│  📍 = Rider requesting a ride (cell c4)                          │
│  🚗 = Available drivers                                          │
│                                                                   │
│  Algorithm:                                                       │
│  1. Find rider's cell (c4)                                       │
│  2. Search that cell + neighboring cells (b3,b4,b5,c3,c5,d3,d4,d5)│
│  3. Calculate actual distance to each driver found               │
│  4. Rank by ETA (distance + traffic)                             │
│                                                                   │
│  Uber uses H3 (hexagonal hierarchical grid by Uber):            │
│  • Hexagons have uniform distances to neighbors                  │
│  • Multiple resolution levels (city → block → street)           │
│  • Each hex cell has a unique 64-bit ID                          │
│                                                                   │
└─────────────────────────────────────────────────────────────────┘
```

### Step 3: The Ride Matching / Dispatch Algorithm

```
Rider requests a ride from point A to point B
         │
         ▼
┌────────────────────────────────────────────────────────────────────┐
│ Step 1: FIND NEARBY DRIVERS                                        │
│ • Query spatial index: drivers within 5 km radius                  │
│ • Filter: available, correct vehicle type, good rating            │
│ • Result: 10-50 candidate drivers                                  │
└──────────────────────┬─────────────────────────────────────────────┘
                       │
                       ▼
┌────────────────────────────────────────────────────────────────────┐
│ Step 2: CALCULATE ETA FOR EACH CANDIDATE                           │
│ • Routing engine calculates drive time from driver → rider        │
│ • Considers: traffic, road type, turn costs, historical data      │
│ • Result: Ranked list by predicted arrival time                    │
└──────────────────────┬─────────────────────────────────────────────┘
                       │
                       ▼
┌────────────────────────────────────────────────────────────────────┐
│ Step 3: DISPATCH OPTIMIZATION                                      │
│                                                                     │
│ NOT just "send to closest driver"! Consider:                        │
│ • Will this create a supply gap in that area?                      │
│ • Is there another rider nearby who needs this driver more?        │
│ • Driver's earnings fairness (spread rides evenly)                  │
│ • Global optimization: minimize TOTAL wait across ALL riders       │
│                                                                     │
│ Uses: Hungarian Algorithm / Online Bipartite Matching              │
└──────────────────────┬─────────────────────────────────────────────┘
                       │
                       ▼
┌────────────────────────────────────────────────────────────────────┐
│ Step 4: OFFER RIDE TO DRIVER                                       │
│ • Send push notification to selected driver                        │
│ • Driver has 15 seconds to accept                                  │
│ • If declined → offer to next best driver                          │
│ • If no one accepts → expand search radius and retry              │
└────────────────────────────────────────────────────────────────────┘
```

### Step 4: ETA Calculation — Predicting Travel Time

```
┌─────────────────────────────────────────────────────────────────┐
│              ETA PREDICTION SYSTEM                                │
├─────────────────────────────────────────────────────────────────┤
│                                                                   │
│  Simple approach: Distance / Average Speed = ETA                 │
│  Problem: Ignores traffic, signals, road type                    │
│                                                                   │
│  Uber's approach: ML on road segments                            │
│                                                                   │
│  Road Network Graph:                                              │
│  ┌──────┐     ┌──────┐     ┌──────┐     ┌──────┐              │
│  │Node A│─5m──│Node B│─3m──│Node C│─8m──│Node D│              │
│  └──┬───┘     └──┬───┘     └──────┘     └──────┘              │
│     │             │                                              │
│     │─7m──┌──────┐│                                             │
│     │     │Node E││                                             │
│     │     └──────┘│                                             │
│     │─12m─────────┘                                             │
│                                                                   │
│  Each edge has:                                                   │
│  • Base travel time (from historical data)                       │
│  • REAL-TIME multiplier (from current GPS data of drivers        │
│    on that segment)                                              │
│  • Time-of-day pattern (rush hour vs. midnight)                  │
│  • Weather impact factor                                         │
│                                                                   │
│  Algorithm: Modified Dijkstra/A* on this weighted graph          │
│  with real-time edge weights                                     │
│                                                                   │
│  Input data: Billions of past trips → ML model predicts          │
│  travel time for each road segment at each time of day           │
│                                                                   │
└─────────────────────────────────────────────────────────────────┘
```

### Step 5: Surge Pricing (Dynamic Pricing)

```
┌─────────────────────────────────────────────────────────────────┐
│              SURGE / DYNAMIC PRICING                              │
├─────────────────────────────────────────────────────────────────┤
│                                                                   │
│  Goal: Balance supply (drivers) and demand (riders)              │
│                                                                   │
│  ┌─────────────────────────────────────────────────────┐        │
│  │               SUPPLY vs DEMAND                       │        │
│  │                                                     │        │
│  │  Price        Supply (drivers willing to drive)     │        │
│  │    │         ╱                                      │        │
│  │  3x│────────╱─────────── Equilibrium ──────────    │        │
│  │    │       ╱        ╲                              │        │
│  │  2x│──────╱──────────╲──────────                   │        │
│  │    │     ╱            ╲                            │        │
│  │  1x│────╱──────────────╲────── Demand (riders)     │        │
│  │    │   ╱                ╲                          │        │
│  │    └───────────────────────── Quantity             │        │
│  └─────────────────────────────────────────────────────┘        │
│                                                                   │
│  How it works per-zone:                                          │
│  1. Divide city into hexagonal zones (H3 cells)                 │
│  2. Every 1-2 minutes, calculate per zone:                      │
│     • Active riders requesting / Nearby available drivers       │
│  3. If ratio > threshold → surge multiplier applied             │
│     • 1.2x, 1.5x, 2.0x, 3.0x...                               │
│  4. Higher price → more drivers go to that area (incentive)     │
│  5. Higher price → some riders wait (reduces demand)            │
│  6. Market reaches equilibrium                                   │
│                                                                   │
│  Calculated independently per zone, updated every ~2 minutes    │
│                                                                   │
└─────────────────────────────────────────────────────────────────┘
```

---

## How It Works Internally

### Uber's Architecture Overview

```
┌─────────────────────────────────────────────────────────────────────┐
│              UBER PLATFORM ARCHITECTURE                               │
├─────────────────────────────────────────────────────────────────────┤
│                                                                      │
│  ┌──────────────┐     ┌──────────────┐                              │
│  │ Rider App    │     │ Driver App   │                              │
│  └──────┬───────┘     └──────┬───────┘                              │
│         │                    │                                       │
│         └────────┬───────────┘                                       │
│                  ▼                                                    │
│  ┌──────────────────────────────────────────────────────────┐       │
│  │               API GATEWAY                                 │       │
│  └──────────────────────────┬───────────────────────────────┘       │
│                             │                                        │
│    ┌────────────────────────┼────────────────────────┐              │
│    ▼            ▼           ▼           ▼            ▼              │
│ ┌────────┐ ┌────────┐ ┌────────┐ ┌──────────┐ ┌──────────┐       │
│ │Dispatch│ │  ETA   │ │ Pricing│ │  Trip    │ │ Payment  │       │
│ │Service │ │Service │ │Service │ │ Service  │ │ Service  │       │
│ └────┬───┘ └────┬───┘ └────┬───┘ └────┬─────┘ └────┬─────┘       │
│      │          │          │          │            │               │
│      ▼          ▼          ▼          ▼            ▼               │
│ ┌──────────────────────────────────────────────────────────┐       │
│ │              DATA LAYER                                    │       │
│ │                                                          │       │
│ │  ┌─────────┐ ┌──────────┐ ┌───────────┐ ┌───────────┐ │       │
│ │  │ Supply  │ │  Map /   │ │  Trip     │ │ Payment   │ │       │
│ │  │ Index   │ │  Routing │ │  Store    │ │ Ledger    │ │       │
│ │  │ (Redis/ │ │  (Graph  │ │  (MySQL/  │ │ (MySQL/   │ │       │
│ │  │  Ring)  │ │   DB)    │ │  Schemaless│ │  Cassandra│ │       │
│ │  └─────────┘ └──────────┘ └───────────┘ └───────────┘ │       │
│ │                                                          │       │
│ └──────────────────────────────────────────────────────────┘       │
│                                                                      │
└─────────────────────────────────────────────────────────────────────┘
```

### The Supply Index — Real-Time Driver Positions

```
┌─────────────────────────────────────────────────────────────────┐
│              SUPPLY INDEX (Ringpop/Custom)                        │
├─────────────────────────────────────────────────────────────────┤
│                                                                   │
│  Purpose: Answer "Which available drivers are near (lat, lng)?" │
│  in < 50ms                                                       │
│                                                                   │
│  Implementation:                                                  │
│  • In-memory geospatial index (not on disk — too slow)          │
│  • Sharded by city/region                                        │
│  • Updated 1M+ times per second (driver GPS updates)            │
│  • Queried millions of times per second (rider requests)        │
│                                                                   │
│  Data structure per city:                                        │
│  ┌──────────────────────────────────────────┐                    │
│  │  H3 Cell ID  │  Drivers in this cell     │                    │
│  │──────────────┼───────────────────────────│                    │
│  │  8a1234567   │  [D1, D2, D3]             │                    │
│  │  8a1234568   │  [D4]                     │                    │
│  │  8a1234569   │  [D5, D6, D7, D8, D9]    │                    │
│  │  ...         │  ...                       │                    │
│  └──────────────────────────────────────────┘                    │
│                                                                   │
│  When driver moves: Remove from old cell, add to new cell        │
│  When rider requests: Query center cell + neighbors              │
│                                                                   │
│  Uber uses H3 (their open-source hexagonal grid):               │
│  • Resolution 7: ~5.16 km² per cell (city-level search)         │
│  • Resolution 9: ~0.1 km² per cell (block-level precision)      │
│                                                                   │
└─────────────────────────────────────────────────────────────────┘
```

### Trip Lifecycle

```
┌─────────────────────────────────────────────────────────────────┐
│              COMPLETE TRIP LIFECYCLE                              │
├─────────────────────────────────────────────────────────────────┤
│                                                                   │
│  1. RIDER REQUESTS ──────────────────────────────────────────   │
│     │ App sends: pickup_lat, pickup_lng, destination            │
│     │ Backend: calculate fare estimate, find drivers             │
│     ▼                                                            │
│  2. MATCHING ────────────────────────────────────────────────   │
│     │ Dispatch picks optimal driver (ETA + fairness)            │
│     │ Offer sent to driver (15s to accept)                      │
│     ▼                                                            │
│  3. EN ROUTE TO PICKUP ──────────────────────────────────────   │
│     │ Driver heading to rider (both track on map)               │
│     │ ETA updating in real-time                                 │
│     │ Driver's location streamed to rider every 4 sec           │
│     ▼                                                            │
│  4. ARRIVED ─────────────────────────────────────────────────   │
│     │ Geofence trigger: driver within 100m of pickup            │
│     │ Timer starts (5 min wait limit)                           │
│     ▼                                                            │
│  5. TRIP IN PROGRESS ────────────────────────────────────────   │
│     │ Route tracking (actual vs. expected)                      │
│     │ Safety monitoring (unexpected stops, route deviations)    │
│     │ Fare meter running (distance + time)                      │
│     ▼                                                            │
│  6. TRIP COMPLETE ───────────────────────────────────────────   │
│     │ Final fare calculated                                     │
│     │ Payment processed (automatic)                             │
│     │ Rating prompt for both rider and driver                   │
│     │ Driver available for next ride                            │
│     ▼                                                            │
│  7. POST-TRIP ───────────────────────────────────────────────   │
│     • Receipt generated                                          │
│     • Trip data archived                                         │
│     • ML models updated (ETA accuracy feedback)                 │
│     • Supply/demand metrics updated                              │
│                                                                   │
└─────────────────────────────────────────────────────────────────┘
```

---

## Code Examples

### Python — Geospatial Nearest Driver Search

```python
# Simplified Uber-style driver matching using H3 geospatial cells
import math
from typing import List, Tuple, Optional
from dataclasses import dataclass

@dataclass
class Driver:
    driver_id: str
    lat: float
    lng: float
    is_available: bool
    vehicle_type: str  # "economy", "premium", "xl"

@dataclass 
class RideRequest:
    rider_id: str
    pickup_lat: float
    pickup_lng: float
    dest_lat: float
    dest_lng: float
    vehicle_type: str

class DriverMatchingService:
    """
    Simplified Uber dispatch: find nearest available drivers 
    using geohash-based spatial indexing.
    """
    
    SEARCH_RADIUS_KM = 5.0
    GEOHASH_PRECISION = 5  # ~5km cells
    
    def __init__(self):
        # geohash → list of drivers in that cell
        self.spatial_index: dict = {}
    
    def update_driver_location(self, driver: Driver):
        """Called every 4 seconds per active driver."""
        cell = self._get_geohash(driver.lat, driver.lng)
        
        # Remove from old cell (if moved)
        for cell_drivers in self.spatial_index.values():
            cell_drivers[:] = [d for d in cell_drivers if d.driver_id != driver.driver_id]
        
        # Add to new cell
        if cell not in self.spatial_index:
            self.spatial_index[cell] = []
        self.spatial_index[cell].append(driver)
    
    def find_nearby_drivers(self, request: RideRequest, max_results: int = 10) -> List[Tuple[Driver, float]]:
        """Find and rank nearest available drivers."""
        
        # Get candidate cells (center + neighbors)
        center_cell = self._get_geohash(request.pickup_lat, request.pickup_lng)
        candidate_cells = self._get_neighboring_cells(center_cell)
        
        # Gather all drivers from candidate cells
        candidates = []
        for cell in candidate_cells:
            for driver in self.spatial_index.get(cell, []):
                if not driver.is_available:
                    continue
                if driver.vehicle_type != request.vehicle_type:
                    continue
                
                # Calculate actual distance
                distance_km = self._haversine(
                    request.pickup_lat, request.pickup_lng,
                    driver.lat, driver.lng
                )
                
                if distance_km <= self.SEARCH_RADIUS_KM:
                    candidates.append((driver, distance_km))
        
        # Sort by distance (in production: sort by ETA instead)
        candidates.sort(key=lambda x: x[1])
        return candidates[:max_results]
    
    def _haversine(self, lat1, lng1, lat2, lng2) -> float:
        """Calculate distance between two GPS points in km."""
        R = 6371  # Earth radius in km
        dlat = math.radians(lat2 - lat1)
        dlng = math.radians(lng2 - lng1)
        a = (math.sin(dlat/2)**2 + 
             math.cos(math.radians(lat1)) * math.cos(math.radians(lat2)) * 
             math.sin(dlng/2)**2)
        return R * 2 * math.asin(math.sqrt(a))
    
    def _get_geohash(self, lat: float, lng: float) -> str:
        """Simplified geohash (in production: use H3 or proper geohash)."""
        # Approximate grid cell at ~5km resolution
        return f"{int(lat*10)}_{int(lng*10)}"
    
    def _get_neighboring_cells(self, cell: str) -> List[str]:
        """Get center cell + 8 neighbors."""
        lat_int, lng_int = map(int, cell.split("_"))
        cells = []
        for dlat in [-1, 0, 1]:
            for dlng in [-1, 0, 1]:
                cells.append(f"{lat_int+dlat}_{lng_int+dlng}")
        return cells
```

### Java — Surge Pricing Calculator

```java
import java.util.*;
import java.util.concurrent.ConcurrentHashMap;

/**
 * Simplified Uber surge pricing engine.
 * Calculates dynamic multiplier based on supply/demand ratio per zone.
 */
public class SurgePricingEngine {
    // H3 cell → current surge multiplier
    private final ConcurrentHashMap<String, Double> surgeMultipliers = 
        new ConcurrentHashMap<>();
    
    // Thresholds for surge activation
    private static final double SURGE_TRIGGER_RATIO = 1.5;  // demand/supply > 1.5
    private static final double MAX_SURGE = 5.0;
    private static final double MIN_SURGE = 1.0;

    /**
     * Recalculate surge for a zone (called every 1-2 minutes per zone).
     */
    public double calculateSurge(String zoneId, int activeRiders, int availableDrivers) {
        if (availableDrivers == 0) {
            surgeMultipliers.put(zoneId, MAX_SURGE);
            return MAX_SURGE;
        }
        
        double demandSupplyRatio = (double) activeRiders / availableDrivers;
        
        double surge;
        if (demandSupplyRatio <= 1.0) {
            surge = MIN_SURGE;  // Supply meets demand — no surge
        } else if (demandSupplyRatio <= SURGE_TRIGGER_RATIO) {
            surge = MIN_SURGE;  // Slightly more demand, but tolerable
        } else {
            // Graduated surge: ratio 2 → 1.5x, ratio 3 → 2.0x, etc.
            surge = Math.min(MAX_SURGE, 1.0 + (demandSupplyRatio - 1.0) * 0.5);
        }
        
        // Smooth transitions: don't jump suddenly
        Double prevSurge = surgeMultipliers.get(zoneId);
        if (prevSurge != null) {
            // Move max 0.5x per calculation cycle (gradual increase/decrease)
            surge = Math.max(prevSurge - 0.5, Math.min(prevSurge + 0.5, surge));
        }
        
        surgeMultipliers.put(zoneId, surge);
        return surge;
    }

    public double getFareMultiplier(String zoneId) {
        return surgeMultipliers.getOrDefault(zoneId, MIN_SURGE);
    }

    public double calculateFare(double baseFare, double distanceKm, 
                                 double timeMinutes, String zoneId) {
        double surge = getFareMultiplier(zoneId);
        double rawFare = baseFare + (distanceKm * 10) + (timeMinutes * 2);
        return rawFare * surge;
    }

    public static void main(String[] args) {
        SurgePricingEngine engine = new SurgePricingEngine();
        
        // Zone with high demand: 100 riders, 20 drivers
        double surge = engine.calculateSurge("zone_downtown", 100, 20);
        System.out.printf("Surge: %.1fx%n", surge);  // ~2.5x
        
        double fare = engine.calculateFare(50, 10, 20, "zone_downtown");
        System.out.printf("Fare: ₹%.0f (with surge)%n", fare);
    }
}
```

---

## Infrastructure Examples

### Uber's Tech Stack

| Component | Technology | Purpose |
|-----------|-----------|---------|
| **Geo Index** | Custom (Ringpop + H3) | Real-time driver positions |
| **Messaging** | Kafka (trillions of messages/day) | Location events, trip events |
| **Database** | MySQL (Schemaless/Docstore) | Trip data, user profiles |
| **Cache** | Redis (Clustered) | Hot data, session state |
| **Routing** | Custom graph DB + OSRM | ETA, directions |
| **Map Data** | OpenStreetMap + proprietary | Road network |
| **ML Platform** | Michelangelo (custom) | ETA prediction, fraud, pricing |
| **Compute** | Kubernetes (on-prem + cloud) | Service hosting |
| **Monitoring** | M3 (custom time-series DB) | Metrics at scale |
| **RPC** | gRPC + Thrift | Inter-service communication |

### System Scale Numbers

```
┌─────────────────────────────────────────────────────────────────┐
│              UBER BY THE NUMBERS                                  │
├─────────────────────────────────────────────────────────────────┤
│                                                                   │
│  • 130 million monthly active riders                             │
│  • 5+ million active drivers                                     │
│  • 28+ million trips per day                                     │
│  • 10,000+ cities in 70+ countries                              │
│  • 1M+ location updates per second (driver GPS)                 │
│  • ETA predictions: billions per day                             │
│  • Kafka: 1 trillion messages/day                               │
│  • Services: 4,000+ microservices                               │
│                                                                   │
│  Performance Requirements:                                       │
│  • Ride matching: < 10 seconds                                  │
│  • Location update processing: < 1 second                       │
│  • ETA calculation: < 200ms                                     │
│  • 99.99% availability (< 1 min downtime/week)                  │
│                                                                   │
└─────────────────────────────────────────────────────────────────┘
```

---

## Real-World Example

### How Uber Handles New Year's Eve (Peak of Peaks)

New Year's Eve midnight = maximum demand in most cities:

```
┌─────────────────────────────────────────────────────────────────┐
│         NEW YEAR'S EVE PREPARATION                               │
├─────────────────────────────────────────────────────────────────┤
│                                                                   │
│  Weeks before:                                                   │
│  • Historical data analyzed (last year's peak)                  │
│  • Capacity pre-provisioned (3x normal)                         │
│  • Surge pricing parameters tuned                               │
│  • Driver incentives configured (earn more on NYE)              │
│                                                                   │
│  Hours before:                                                   │
│  • Load tests run against production-like environment            │
│  • All deployment freezes activated (no code changes)           │
│  • On-call engineers on standby                                 │
│  • CDN cache warmed up                                          │
│                                                                   │
│  At midnight:                                                    │
│  • 10x-20x normal ride requests within minutes                  │
│  • Surge pricing activates automatically (up to 5x)            │
│  • Drivers flock to high-surge areas (economic incentive)       │
│  • System handles 50K+ ride requests/sec in top cities         │
│  • Queue-based dispatch absorbs burst                           │
│                                                                   │
│  Key insight: Surge pricing ISN'T just about revenue.           │
│  It's a SIGNAL that pulls more supply (drivers) to              │
│  high-demand areas. Without it, wait times would be 1 hour+.   │
│                                                                   │
└─────────────────────────────────────────────────────────────────┘
```

### Uber vs. Ola — Architectural Differences

| Aspect | Uber | Ola |
|--------|------|-----|
| Primary market | Global (70+ countries) | India-focused (international expansion) |
| Map data | Custom + OSM + Google | Google Maps + proprietary |
| Spatial index | H3 (custom hexagonal) | Geohash-based |
| Pricing | Surge (demand-based) | Peak pricing (similar) |
| Payments | Cards, UPI, Wallets | Heavy UPI/Wallet focus (India) |
| Tech stack | Go, Java, Python, Node | Java, Python, Go |

---

## Common Mistakes / Pitfalls

| Mistake | Why It's Wrong | Better Approach |
|---------|---------------|-----------------|
| Using traditional DB queries for "nearest driver" | Full table scan with ORDER BY distance is O(n) — too slow | Use spatial indexing (H3, geohash, R-tree) for O(1) cell lookup |
| Matching based purely on distance | Closest driver might be across a river or highway | Use ETA (actual drive time) not crow-flies distance |
| Storing driver locations in a database | Too slow for 1M updates/sec; disk I/O is the bottleneck | In-memory spatial index, updated in real-time |
| Simple "assign closest" dispatch | Doesn't optimize globally; creates supply deserts | Batch matching with Hungarian algorithm or similar |
| Static pricing | During demand spikes, no supply incentive → long wait times | Dynamic pricing that signals drivers to move to high-demand areas |
| Single region deployment | Drivers are everywhere; latency for location updates matters | Multi-region with city-level sharding |

---

## When to Use / When NOT to Use

### When to Use This Architecture
- Real-time matching of mobile supply and demand
- High-frequency location updates (GPS tracking at scale)
- Geospatial search is a core requirement
- Dynamic pricing based on real-time supply/demand
- Delivery, ride-hailing, fleet management applications

### When NOT to Use This Architecture
- Static location lookups (restaurants, shops) → Simple spatial DB query
- Low-frequency updates (weekly, not real-time) → Standard database works
- Small fleet (< 1000 vehicles) → PostGIS or simple spatial queries
- No matching needed (just tracking) → Simpler event streaming
- Pre-scheduled routes (like buses) → Route planning, not real-time matching

---

## Key Takeaways

1. **Geospatial indexing is everything**: Uber's H3 hexagonal grid enables finding nearby drivers in O(1) lookup time, not O(n) full table scans
2. **The supply index is in-memory**: At 1M+ GPS updates/second, you can't use disk-based databases. The spatial index MUST be in RAM.
3. **Dispatch is a global optimization problem**: "Send closest driver" is naive. Real dispatch considers system-wide efficiency, driver fairness, and future demand.
4. **Surge pricing is supply signaling**: It's not just revenue optimization — it physically moves drivers to where demand is, reducing wait times for everyone
5. **ETA accuracy = customer trust**: Uber uses ML trained on billions of past trips, considering traffic, time-of-day, weather, and road segment speeds
6. **4-second GPS updates** from millions of drivers create one of the largest real-time data streams on Earth — Kafka processes trillions of messages per day for this
7. **City-level sharding** makes the problem tractable: each city is largely independent (drivers in Mumbai don't affect dispatch in Delhi)

---

## What's Next?

Next, we'll explore [How Stripe/Razorpay Handles Payment Processing](./08-payment-processing.md) — understanding how payment platforms process billions of dollars in transactions with exactly-once semantics, fraud detection, and multi-party settlement.
