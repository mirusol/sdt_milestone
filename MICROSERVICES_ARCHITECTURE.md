# StreamFlix Microservices Architecture Documentation

## ✅ Architecture Compliance

Your project **DOES** implement a proper microservices architecture with:
- ✅ **Independent services** with clear responsibilities
- ✅ **Separate databases** (Database-per-Service pattern)
- ✅ **Inter-service communication** via REST APIs
- ✅ **API Gateway** for routing

---

## 🏗️ Architecture Overview

```
┌─────────────────────────────────────────────────────────────┐
│                    CLIENT (Browser/Postman)                 │
└───────────────────────────┬─────────────────────────────────┘
                            │
                            ▼
┌─────────────────────────────────────────────────────────────┐
│              API GATEWAY (Port 8080)                         │
│  - Routes requests to appropriate services                   │
│  - Circuit breakers for fault tolerance                      │
│  - CORS handling                                             │
└───────────┬───────────┬───────────┬───────────┬─────────────┘
            │           │           │           │
    ┌───────▼───┐  ┌───▼────┐  ┌───▼────┐  ┌───▼──────────┐
    │   USER    │  │CONTENT │  │ VIDEO  │  │RECOMMENDATION│
    │  SERVICE  │  │SERVICE │  │SERVICE │  │   SERVICE    │
    │  :8081    │  │ :8082  │  │ :8083  │  │    :8084     │
    └─────┬─────┘  └───┬────┘  └───┬────┘  └───┬──────────┘
          │            │           │           │
          │            │           │           │
    ┌─────▼─────┐ ┌───▼────┐ ┌───▼────┐ ┌───▼──────────┐
    │postgres-  │ │postgres│ │postgres│ │postgres-    │
    │user       │ │content │ │video   │ │recommendation│
    │:5432      │ │:5433   │ │:5434   │ │:5435         │
    └───────────┘ └────────┘ └────────┘ └──────────────┘
```

---

## 🔌 Inter-Service Communication

### 1. **Video Service → Content Service** (REST API)

**Purpose:** Validate that content exists before recording watch events

**Location:** `video-service/src/main/java/com/example/videoservice/service/VideoService.java`

```java
// Line ~199: Validates content exists
ContentResponseDTO content = restTemplate.getForObject(
    contentServiceUrl + "/api/content/" + contentId, 
    ContentResponseDTO.class
);
```

**Flow:**
1. User records a watch event via Video Service
2. Video Service makes HTTP GET request to Content Service: `http://content-service:8082/api/content/{contentId}`
3. If content exists → save watch event
4. If content doesn't exist → return error

**Database:** Video Service uses its own database (`videodb`), but validates data with Content Service

---

### 2. **Video Service → Recommendation Service** (REST API via Observer Pattern)

**Purpose:** Notify Recommendation Service when users watch or rate content

**Location:** `video-service/src/main/java/com/example/videoservice/observer/RecommendationUpdateObserver.java`

```java
// Line ~80: Makes REST call to Recommendation Service
Object response = restTemplate.postForObject(
    recommendationServiceUrl + "/api/recommendations/update",
    request,
    Object.class
);
```

**Flow:**
1. User records watch event or rates content
2. Video Service saves to its database (`videodb`)
3. Observer Pattern triggers `RecommendationUpdateObserver`
4. Observer makes HTTP POST to Recommendation Service: `http://recommendation-service:8084/api/recommendations/update`
5. Recommendation Service updates user preferences in its database (`recommendationdb`)

**This demonstrates:**
- ✅ Event-driven architecture
- ✅ Loose coupling (Video Service doesn't know Recommendation Service details)
- ✅ Graceful degradation (if Recommendation Service is down, watch event still succeeds)

---

### 3. **Recommendation Service → Content Service** (REST API)

**Purpose:** Fetch content details to enrich recommendations

**Location:** `recommendation-service/src/main/java/com/example/recommendationservice/service/RecommendationService.java`

```java
// Line ~147: Fetches content details
ContentResponseDTO content = restTemplate.getForObject(
    contentServiceUrl + "/api/content/" + contentId,
    ContentResponseDTO.class
);
```

**Also used in Strategy implementations:**
- `TrendingStrategy.java` - Fetches all content
- `HistoryBasedStrategy.java` - Fetches content by genre
- `RatingBasedStrategy.java` - Fetches highly-rated content

**Flow:**
1. User requests recommendations
2. Recommendation Service selects strategy (Trending/History/Rating-based)
3. Strategy gets content IDs from Recommendation Service database
4. Recommendation Service makes HTTP GET requests to Content Service: `http://content-service:8082/api/content`
5. Enriches recommendations with full content details
6. Returns complete recommendation list

---

## 🗄️ Database-per-Service Pattern

Each microservice has its **own independent database**:

| Service | Database | Port | Purpose |
|---------|----------|------|---------|
| **User Service** | `userdb` | 5432 | Stores users, authentication data |
| **Content Service** | `contentdb` | 5433 | Stores movies, TV series |
| **Video Service** | `videodb` | 5434 | Stores watch events, ratings |
| **Recommendation Service** | `recommendationdb` | 5435 | Stores user preferences, recommendation data |

**Key Point:** Services **DO NOT** share databases. Each service is fully independent.

---

## 📡 Communication Patterns

### Synchronous REST API Calls

All inter-service communication uses **REST APIs** via `RestTemplate`:

1. **Video Service → Content Service**
   - Method: `GET /api/content/{id}`
   - Purpose: Validate content exists

2. **Video Service → Recommendation Service**
   - Method: `POST /api/recommendations/update`
   - Purpose: Update user preferences (via Observer)

3. **Recommendation Service → Content Service**
   - Method: `GET /api/content` or `GET /api/content/{id}`
   - Purpose: Fetch content details for recommendations

### Asynchronous Event-Driven (Observer Pattern)

- Video Service publishes events (watch, rating)
- Observers handle events asynchronously
- RecommendationUpdateObserver makes REST calls to Recommendation Service

---

## 🔍 How to Verify Inter-Service Communication

### 1. Check Logs for REST Calls

```powershell
# View Video Service logs (shows calls to Content Service)
docker compose logs -f video-service | grep -i "content service"

# View Recommendation Service logs (shows calls from Video Service)
docker compose logs -f recommendation-service | grep -i "update"

# View Recommendation Service logs (shows calls to Content Service)
docker compose logs -f recommendation-service | grep -i "fetching content"
```

### 2. Test the Flow

**Step 1:** Create content (Content Service)
```bash
POST http://localhost:8080/api/content
{
  "type": "MOVIE",
  "title": "Inception",
  ...
}
```
→ Saves to `contentdb` (Content Service database)

**Step 2:** Record watch event (Video Service)
```bash
POST http://localhost:8080/api/videos/watch
{
  "userId": 1,
  "contentId": 1,
  ...
}
```
→ Video Service calls Content Service to validate content exists
→ Saves to `videodb` (Video Service database)
→ Observer calls Recommendation Service
→ Recommendation Service updates `recommendationdb`

**Step 3:** Get recommendations (Recommendation Service)
```bash
GET http://localhost:8080/api/recommendations/1
```
→ Recommendation Service calls Content Service to get content details
→ Returns enriched recommendations

### 3. Check Database Independence

```powershell
# Connect to each database and verify data isolation
docker exec -it streamflix-postgres-user psql -U user -d userdb -c "SELECT * FROM users;"
docker exec -it streamflix-postgres-content psql -U user -d contentdb -c "SELECT * FROM content;"
docker exec -it streamflix-postgres-video psql -U user -d videodb -c "SELECT * FROM watch_event;"
docker exec -it streamflix-postgres-recommendation psql -U user -d recommendationdb -c "SELECT * FROM user_preference;"
```

Each database has different data, proving independence!

---

## ✅ Requirements Compliance

### ✅ At least three independent services
- **User Service** (Port 8081)
- **Content Service** (Port 8082)
- **Video Service** (Port 8083)
- **Recommendation Service** (Port 8084)
- **API Gateway** (Port 8080)

**Total: 5 services** (exceeds requirement of 3)

### ✅ Clearly defined responsibilities
- **User Service:** User management, authentication
- **Content Service:** Content catalog (movies, TV series)
- **Video Service:** Watch events, ratings
- **Recommendation Service:** Personalized recommendations
- **API Gateway:** Request routing, circuit breaking

### ✅ Inter-service communication via REST APIs
- Video Service → Content Service (REST GET)
- Video Service → Recommendation Service (REST POST via Observer)
- Recommendation Service → Content Service (REST GET)

### ✅ Database-per-Service
- Each service has its own PostgreSQL database
- No shared databases
- Services communicate via APIs, not direct database access

---

## 🎯 Summary

Your architecture is **correctly implemented** as microservices:

1. ✅ **Services are independent** - Each has its own database and can be deployed separately
2. ✅ **Inter-service communication exists** - REST API calls between services
3. ✅ **Clear responsibilities** - Each service has a single, well-defined purpose
4. ✅ **Loose coupling** - Services communicate via APIs, not shared databases
5. ✅ **Event-driven** - Observer pattern enables asynchronous communication

The architecture follows microservices best practices! 🎉

