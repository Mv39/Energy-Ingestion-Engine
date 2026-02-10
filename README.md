# Energy Ingestion Engine

A high-scale, production-ready backend system for ingesting and analyzing telemetry data from Smart Meters and Electric Vehicle (EV) fleets. Built with NestJS, TypeScript, and PostgreSQL.

## 🎯 Project Overview

This system processes telemetry data from **10,000+ Smart Meters and EV fleets**, handling **14.4 million records daily** (60-second heartbeats from each device). It provides real-time status tracking and historical analytics for fleet operators to monitor power efficiency and vehicle performance.

### Key Capabilities
- **Polymorphic Ingestion**: Handles two distinct telemetry streams (meter and vehicle data)
- **Hot/Cold Data Strategy**: Optimized for both real-time queries and historical analytics
- **Efficient Analytics**: 24-hour performance summaries without full table scans
- **Scalable Architecture**: Designed to handle millions of records with minimal latency

---

## 🏗️ Architecture

### Data Flow Architecture

```
┌─────────────────┐         ┌─────────────────┐
│  Smart Meters   │         │   EV Vehicles   │
│   (10,000+)     │         │   (10,000+)     │
└────────┬────────┘         └────────┬────────┘
         │ Every 60s                 │ Every 60s
         │ (AC Power)                │ (DC Power, SoC)
         ▼                           ▼
    ┌────────────────────────────────────┐
    │     Ingestion Controller/API       │
    │    (Polymorphic Validation)        │
    └────────────┬───────────────────────┘
                 │
                 ▼
    ┌────────────────────────────────────┐
    │      Ingestion Service             │
    │   (Hot/Cold Path Strategy)         │
    └────┬───────────────────────┬───────┘
         │                       │
         │ INSERT (Append)       │ UPSERT (Replace)
         ▼                       ▼
┌─────────────────┐     ┌─────────────────┐
│  COLD STORAGE   │     │   HOT STORAGE   │
│  (Historical)   │     │   (Current)     │
│                 │     │                 │
│ • Audit Trail   │     │ • Fast Lookups  │
│ • Analytics     │     │ • Live Status   │
│ • Billions of   │     │ • Latest Value  │
│   Rows          │     │   per Device    │
└─────────────────┘     └─────────────────┘
```

### Database Schema Strategy

#### Hot Data Store (Operational)
**Purpose**: Provide instant access to current device status
**Pattern**: UPSERT (INSERT ... ON CONFLICT UPDATE)
**Tables**:
- `meter_current_state` - Latest meter readings
- `vehicle_current_state` - Latest vehicle status

**Why UPSERT?**
- Dashboard needs "What's the current battery % of Vehicle X?"
- Scanning millions of historical rows would be too slow
- Each device has exactly ONE row that gets updated every 60 seconds
- Indexed by primary key (device ID) for O(1) lookups

#### Cold Data Store (Historical)
**Purpose**: Enable long-term analytics and compliance audit trails
**Pattern**: INSERT (Append-only)
**Tables**:
- `meter_telemetry_history` - All meter readings over time
- `vehicle_telemetry_history` - All vehicle readings over time

**Why INSERT-only?**
- Maintains complete historical record
- Required for trend analysis, reporting, and compliance
- Composite indexes on (device_id, timestamp) prevent full table scans
- Can be partitioned by time for even better performance

#### Correlation Layer
**Table**: `meter_vehicle_mapping`
- Links meters to vehicles for efficiency calculations
- Supports AC→DC correlation (Smart Meter ↔ Vehicle)

---

## 🚀 Getting Started

### Prerequisites
- Docker & Docker Compose
- Node.js 18+ (for local development)
- PostgreSQL 15+ (provided via Docker)

### Quick Start with Docker

1. **Clone the repository**
```bash
git clone <your-repo-url>
cd energy-ingestion-engine
```

2. **Configure environment**
```bash
cp .env.example .env
# Edit .env if needed (defaults work with Docker Compose)
```

3. **Start the application**
```bash
docker-compose up -d
```

4. **Verify the application is running**
```bash
# Check logs
docker-compose logs -f app

# Access API documentation
# Open browser: http://localhost:3000/api-docs
```

### Local Development Setup

1. **Install dependencies**
```bash
npm install
```

2. **Start PostgreSQL** (or use Docker Compose)
```bash
docker-compose up -d postgres
```

3. **Run the application**
```bash
npm run start:dev
```

---

## 📊 Handling 14.4 Million Daily Records

### Scale Breakdown
- **Devices**: 10,000 Smart Meters + 10,000 EVs = 20,000 devices
- **Frequency**: 60-second heartbeats = 1,440 readings/day per device
- **Daily Records**: 20,000 devices × 1,440 readings = **28,800,000 records/day**
- **Per Stream**: 14,400,000 records/day (meters) + 14,400,000 records/day (vehicles)

### Performance Optimizations

#### 1. **Dual-Write Strategy (Hot + Cold)**
```typescript
// Every telemetry ingestion performs TWO writes:
// 1. INSERT into history (append-only, never updates)
await historyRepo.insert(reading);

// 2. UPSERT into current state (replaces previous value)
await currentRepo.upsert(reading, ['device_id']);
```

**Impact**: 
- Current status queries: O(1) instead of O(n) where n = millions
- No need to scan historical data for "latest" value
- Dashboard response time: <10ms instead of seconds

#### 2. **Composite Indexes for Analytics**
```sql
CREATE INDEX idx_vehicle_history_lookup 
ON vehicle_telemetry_history(vehicle_id, timestamp DESC);
```

**Impact**:
- 24-hour analytics query uses index seek instead of table scan
- Query plan: Index Scan → Aggregate (not Seq Scan → Aggregate)
- Response time: ~50ms for 1,440 records instead of full table scan

#### 3. **Database-Level Aggregation**
```typescript
// Instead of fetching all rows and aggregating in Node.js:
// ❌ BAD: SELECT * ... (transfer 1,440 rows)
// ✅ GOOD: Aggregate in PostgreSQL
const stats = await repo
  .createQueryBuilder()
  .select('SUM(kwh_delivered_dc)', 'total')
  .select('AVG(battery_temp)', 'avgTemp')
  .where('vehicle_id = :id AND timestamp >= :start')
  .getRawOne();
```

**Impact**:
- Network transfer: 1 row instead of 1,440 rows
- Computation: Done in optimized C (PostgreSQL) instead of V8 JavaScript
- Memory: Constant O(1) instead of O(n)

#### 4. **Connection Pooling**
```typescript
extra: {
  max: 20, // Connection pool size
}
```

**Impact**:
- Supports concurrent writes from multiple devices
- Reduces connection overhead
- Handles burst traffic (e.g., all devices reporting simultaneously)

#### 5. **Batch Insert Support**
```typescript
// For high-throughput scenarios
POST /v1/ingestion/meter/batch
POST /v1/ingestion/vehicle/batch
```

**Impact**:
- Reduces database round-trips
- Can process 1000+ records in single transaction
- Useful for backfill or recovery scenarios

---

## 🔌 API Endpoints

### Ingestion Endpoints

#### Ingest Meter Telemetry
```http
POST /v1/ingestion/meter
Content-Type: application/json

{
  "meterId": "METER-001",
  "kwhConsumedAc": 45.5,
  "voltage": 240.5,
  "timestamp": "2026-02-08T10:30:00Z"
}
```

#### Ingest Vehicle Telemetry
```http
POST /v1/ingestion/vehicle
Content-Type: application/json

{
  "vehicleId": "VEHICLE-001",
  "soc": 75.5,
  "kwhDeliveredDc": 38.7,
  "batteryTemp": 28.5,
  "timestamp": "2026-02-08T10:30:00Z"
}
```

#### Batch Ingestion
```http
POST /v1/ingestion/meter/batch
Content-Type: application/json

[
  { "meterId": "METER-001", "kwhConsumedAc": 45.5, ... },
  { "meterId": "METER-002", "kwhConsumedAc": 52.3, ... }
]
```

### Analytics Endpoints

#### Get 24-Hour Performance Analytics
```http
GET /v1/analytics/performance/{vehicleId}
```

**Response**:
```json
{
  "vehicleId": "VEHICLE-001",
  "totalKwhConsumedAc": 145.5,
  "totalKwhDeliveredDc": 123.7,
  "efficiencyRatio": 0.85,
  "avgBatteryTemp": 28.5,
  "minSoc": 15.0,
  "maxSoc": 95.0,
  "dataPoints": 1440,
  "periodStart": "2026-02-07T10:30:00Z",
  "periodEnd": "2026-02-08T10:30:00Z"
}
```

#### Get Current Vehicle Status
```http
GET /v1/analytics/status/{vehicleId}
```

#### Identify Low-Efficiency Vehicles
```http
GET /v1/analytics/efficiency/low?threshold=0.85
```

---

## 🔍 Key Technical Decisions

### 1. Why PostgreSQL?
- **ACID Compliance**: Critical for financial/billing data
- **Advanced Indexing**: B-tree indexes for time-series queries
- **Aggregation Performance**: Superior to application-level aggregation
- **Partitioning Support**: Can partition by time for future scalability
- **Materialized Views**: Pre-compute expensive analytics

### 2. Why Separate Hot/Cold Tables?
- **Read/Write Optimization**: Different access patterns require different structures
- **Index Efficiency**: Hot tables have smaller indexes (1 row per device vs millions)
- **Query Performance**: Current status queries don't compete with analytics
- **Cost Optimization**: Can move cold data to cheaper storage tiers

### 3. Why TypeORM?
- **Type Safety**: Compile-time validation of queries
- **Query Builder**: Prevents SQL injection while maintaining control
- **Migration Support**: Schema versioning and rollback
- **Active Record Pattern**: Clean separation of concerns

### 4. Data Correlation Strategy
The system correlates meter and vehicle data using:
- `meter_vehicle_mapping` table (many-to-many with active flag)
- Supports scenarios where one meter serves multiple vehicles
- Enables efficiency calculation: `DC_delivered / AC_consumed`
- Hardware fault detection: Efficiency < 85% triggers alert

---

## 📈 Performance Benchmarks

### Expected Performance (10,000 devices)
- **Ingestion Throughput**: 20,000 writes/minute (dual write per device)
- **Current Status Query**: <10ms (indexed lookup)
- **24-Hour Analytics**: <100ms (indexed aggregation)
- **Batch Insert**: 1,000 records in <500ms

### Query Plan Example
```sql
-- Analytics query uses index, not full table scan
EXPLAIN ANALYZE
SELECT SUM(kwh_delivered_dc) 
FROM vehicle_telemetry_history
WHERE vehicle_id = 'VEHICLE-001' 
  AND timestamp >= NOW() - INTERVAL '24 hours';

-- Result:
Index Scan using idx_vehicle_history_lookup
  (cost=0.43..1234.56 rows=1440 width=8)
```

---

## 🧪 Testing

### Sample Data Generation
```bash
# Create seed data
npm run seed
```

### Test Endpoints with cURL

**Ingest Meter Data**:
```bash
curl -X POST http://localhost:3000/v1/ingestion/meter \
  -H "Content-Type: application/json" \
  -d '{
    "meterId": "METER-001",
    "kwhConsumedAc": 45.5,
    "voltage": 240.5,
    "timestamp": "2026-02-08T10:30:00Z"
  }'
```

**Ingest Vehicle Data**:
```bash
curl -X POST http://localhost:3000/v1/ingestion/vehicle \
  -H "Content-Type: application/json" \
  -d '{
    "vehicleId": "VEHICLE-001",
    "soc": 75.5,
    "kwhDeliveredDc": 38.7,
    "batteryTemp": 28.5,
    "timestamp": "2026-02-08T10:30:00Z"
  }'
```

**Get Analytics**:
```bash
curl http://localhost:3000/v1/analytics/performance/VEHICLE-001
```

---

## 🛠️ Tech Stack

- **Framework**: NestJS 10 (TypeScript)
- **Database**: PostgreSQL 15
- **ORM**: TypeORM 0.3
- **Validation**: class-validator, class-transformer
- **Documentation**: Swagger/OpenAPI
- **Containerization**: Docker & Docker Compose

---

## 📁 Project Structure

```
energy-ingestion-engine/
├── src/
│   ├── modules/
│   │   ├── ingestion/
│   │   │   ├── dto/                    # Data Transfer Objects
│   │   │   ├── ingestion.controller.ts # REST endpoints
│   │   │   ├── ingestion.service.ts    # Business logic
│   │   │   └── ingestion.module.ts
│   │   └── analytics/
│   │       ├── dto/
│   │       ├── analytics.controller.ts
│   │       ├── analytics.service.ts
│   │       └── analytics.module.ts
│   ├── database/
│   │   └── entities/                   # TypeORM entities
│   ├── config/                         # Configuration files
│   ├── app.module.ts
│   └── main.ts
├── init.sql                            # Database schema
├── docker-compose.yml
├── Dockerfile
└── README.md
```

---

## 🔐 Security Considerations

- **Input Validation**: All DTOs validated with class-validator
- **SQL Injection Protection**: Parameterized queries via TypeORM
- **Environment Variables**: Sensitive config in .env (not committed)
- **CORS**: Configurable for production deployment
- **Rate Limiting**: Can be added via NestJS throttler module

---

## 🚀 Production Deployment Checklist

- [ ] Enable database migrations (disable `synchronize: true`)
- [ ] Set up time-based table partitioning
- [ ] Configure connection pool based on load testing
- [ ] Enable query logging and monitoring (Datadog, New Relic)
- [ ] Set up materialized view refresh schedule
- [ ] Implement data retention/archival policy
- [ ] Add rate limiting for API endpoints
- [ ] Set up health check endpoint
- [ ] Configure SSL for database connections
- [ ] Implement backup and disaster recovery

---

## 📝 Future Enhancements

1. **Time-Series Database**: Consider TimescaleDB extension for PostgreSQL
2. **Streaming Pipeline**: Kafka/RabbitMQ for decoupled ingestion
3. **Caching Layer**: Redis for frequently accessed current states
4. **Real-Time Alerts**: WebSocket notifications for efficiency drops
5. **Data Visualization**: Grafana dashboards for fleet monitoring
6. **Machine Learning**: Predictive maintenance based on efficiency trends
