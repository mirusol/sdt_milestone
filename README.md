# MILESTONE 5- StreamFlix - Microservices Video Streaming Platform

A Netflix-like video streaming platform demonstrating microservices architecture, design patterns, **message queue integration**, and a **CI/CD pipeline**

**Team:** Bilciurescu Elena-Alina, 
Solomon Miruna-Maria,
Toma Daria-Maria  
**Group:** 1241EA CTI-E  
**Course:** Software Design Techniques  

## Project Overview

StreamFlix is a proof-of-concept microservices application that demonstrates four non-trivial design patterns (Factory Method, Strategy, Observer, Singleton) integrated into a video streaming platform architecture.

This **Milestone 5** extends the previous milestones by:
- Integrating **RabbitMQ** as a message queue for **asynchronous communication** between services.
- Adding a **GitHub Actions CI/CD pipeline** that automatically **builds, tests, and deploys** the microservices into a local Docker environment.

### Core Features

- **User Management**: Registration, authentication, subscription tiers (BASIC, STANDARD, PREMIUM)
- **Content Catalog**: Movies and TV Series with search and filtering
- **Video Streaming**: Watch events, progress tracking, watch history
- **Rating System**: 1-5 star ratings that influence recommendations
- **Recommendation Engine**: Personalized content suggestions using multiple algorithms
- **Real-time Notifications**: Event-driven architecture with observer pattern

### Technology Stack

- **Backend**: Java 17, Spring Boot 3.2.0, Spring Cloud Gateway
- **Database**: PostgreSQL 15 (4 instances - one per service)
- **Message Queue**: RabbitMQ 3.12 (asynchronous communication)
- **Containerization**: Docker, Docker Compose
- **CI/CD**: GitHub Actions (automated build, test, deploy)
- **API Testing**: Postman (30 essential requests)
- **Build Tool**: Maven 3.8+

---

## Architecture

### Microservices Architecture

```
CLIENT
   |
   v
API GATEWAY (Port 8080)
   |
   +-- User Service (Port 8081) - Singleton Pattern
   |
   +-- Content Service (Port 8082) - Factory Pattern
   |
   +-- Video Service (Port 8083) - Observer Pattern
   |
   +-- Recommendation Service (Port 8084) - Strategy Pattern
```

### Database-per-Service

Each microservice has its own PostgreSQL database:
- `streamflix_user` (Port 5432)
- `streamflix_content` (Port 5433)
- `streamflix_video` (Port 5434)
- `streamflix_recommendation` (Port 5435)

### Message Queue Architecture

**RabbitMQ** is used for asynchronous communication between services:
- **Queue**: `user.preference.updates`
- **Publisher**: Video Service (publishes watch/rating events)
- **Consumer**: Recommendation Service (updates user preferences)

**Benefits:**
- **Decoupling**: Services don't need to be available simultaneously
- **Scalability**: Messages are queued and processed asynchronously
- **Fault Tolerance**: If Recommendation Service is down, messages are queued and processed when it's back
- **Performance**: Non-blocking, doesn't slow down Video Service operations

**Flow:**
1. User watches/rates content → Video Service saves to database
2. Video Service publishes message to RabbitMQ queue
3. Recommendation Service consumes message asynchronously
4. Recommendation Service updates user preferences in its database

### Services Overview

| Service | Port | Pattern | Purpose | Key Endpoints |
|---------|------|---------|---------|---------------|
| **API Gateway** | 8080 | Gateway | Request routing, CORS | `/api/*` |
| **User Service** | 8081 | Singleton | User management | `/api/users/*` |
| **Content Service** | 8082 | Factory | Content catalog | `/api/content/*` |
| **Video Service** | 8083 | Observer | Watch events, ratings | `/api/videos/*` |
| **Recommendation Service** | 8084 | Strategy | Personalized recommendations | `/api/recommendations/*` |

---

## Design Patterns

### 1. Factory Method Pattern (Content Service)

**Purpose**: Create different content types (Movie vs TV Series) without conditional logic.

**Implementation**:
- `ContentFactory` interface
- `MovieFactory` creates `Movie` objects with movie-specific fields (duration, director)
- `TVSeriesFactory` creates `TVSeries` objects with series-specific fields (seasons, episodes)

**Why?**: 
- Eliminates if/else statements for content type handling
- Easy to extend with new content types (Documentaries, Podcasts, etc.)
- Type-safe content creation with compile-time validation

**Example**:
```java
ContentFactory factory = contentType.equals("MOVIE") 
    ? new MovieFactory() 
    : new TVSeriesFactory();
Content content = factory.createContent(dto);
```

### 2. Strategy Pattern (Recommendation Service)

**Purpose**: Swap recommendation algorithms dynamically based on user behavior.

**Implementation**:
- `RecommendationStrategy` interface
- `TrendingStrategy` - For new users without history
- `HistoryBasedStrategy` - For users with watch history
- `RatingBasedStrategy` - For users who rate content

**Why?**:
- Avoids 200+ line methods with complex if/else logic
- Easy A/B testing by switching strategies
- Each algorithm is independently testable

**Example**:
```java
RecommendationStrategy strategy = selectStrategy(user);
List<Content> recommendations = strategy.recommend(user, limit);
```

### 3. Observer Pattern (Video Service)

**Purpose**: Multiple services react to watch events without tight coupling.

**Implementation**:
- `VideoEventPublisher` publishes watch and rating events
- `AnalyticsObserver` - Records metrics locally
- `RecommendationUpdateObserver` - Makes REST call to update recommendations
- `WatchHistoryObserver` - Updates user's watch history

**Why?**:
- Decouples event producers from consumers
- Easy to add new observers without modifying existing code
- Asynchronous event processing doesn't block main operations

**Example**:
```java
// Watch event triggers all registered observers
publisher.publishWatchEvent(watchEvent);
// -> AnalyticsObserver logs metrics
// -> RecommendationUpdateObserver updates recommendations
// -> WatchHistoryObserver updates history
```

### 4. Singleton Pattern (User Service)

**Purpose**: Single instance of shared resources (ConfigurationManager).

**Implementation**:
- `ConfigurationManager` - Manages service-wide configuration
- Thread-safe lazy initialization with double-checked locking
- Volatile keyword ensures visibility across threads

**Why?**:
- Prevents multiple instances wasting memory
- Ensures consistent configuration across all components
- Thread-safe access to shared resources

**Example**:
```java
ConfigurationManager config = ConfigurationManager.getInstance();
String jwtSecret = config.getJwtSecret();
```

### Pattern Interactions

The patterns work together cohesively:

1. **Factory → Observer**: ContentService creates content using factories, then publishes ContentAddedEvent through Observer pattern
2. **Observer → Strategy**: Watch events collected by observers feed data to recommendation strategies
3. **Singleton → All**: All services use ConfigurationManager singleton for consistent configuration
4. **Strategy → Observer**: Recommendation strategies analyze data collected by observers

---

## Getting Started

### Prerequisites

- Docker Desktop
- Java 17 JDK (for local development)
- Maven 3.8+ (for local development)
- Postman (for API testing)

### Quick Start with Docker

**1. Build all services:**
```bash
./build-all.sh  # Mac/Linux
# or
build-all.bat   # Windows
```

**2. Start all services:**
```bash
docker-compose up -d
```

**3. Verify services are running:**
```bash
docker-compose ps
```

All services should show "Up" and "healthy" status.

**4. Access the UI:**
```
http://localhost:8080/ui/index.html
```

**5. Check health endpoints:**
- API Gateway: http://localhost:8080/actuator/health
- User Service: http://localhost:8080/api/users/health
- Content Service: http://localhost:8080/api/content/health
- Video Service: http://localhost:8080/api/videos/health
- Recommendation Service: http://localhost:8080/api/recommendations/health

### Manual Build (without Docker)

```bash
# Build each service
cd user-service && mvn clean package -DskipTests
cd content-service && mvn clean package -DskipTests
cd video-service && mvn clean package -DskipTests
cd recommendation-service && mvn clean package -DskipTests
cd api-gateway && mvn clean package -DskipTests
```

### Stopping Services

```bash
docker-compose down
```

---

## API Documentation

### User Service (Port 8081)

**Base URL**: `/api/users`

| Method | Endpoint | Description | Request Body |
|--------|----------|-------------|--------------|
| POST | `/register` | Register new user | `{username, email, password, tier}` |
| POST | `/login` | User login | `{email, password}` |
| GET | `/{id}` | Get user by ID | - |
| GET | `/` | Get all users | - |
| PUT | `/{id}` | Update user profile | `{username, email, password}` |
| PUT | `/{id}/subscription` | Update subscription tier | `{tier}` |
| DELETE | `/{id}` | Delete user | - |
| GET | `/demo/singleton-test` | Test singleton pattern | - |

**Subscription Tiers**: `BASIC`, `STANDARD`, `PREMIUM`

### Content Service (Port 8082)

**Base URL**: `/api/content`

| Method | Endpoint | Description | Request Body |
|--------|----------|-------------|--------------|
| POST | `/` | Create content (Factory) | `{type, title, description, genre, releaseYear, ...}` |
| GET | `/` | Get all content | - |
| GET | `/{id}` | Get content by ID | - |
| GET | `/movies` | Get all movies | - |
| GET | `/series` | Get all TV series | - |
| GET | `/search?query={q}` | Search content | - |
| GET | `/genre/{genre}` | Get content by genre | - |
| GET | `/top-rated` | Get top rated content | - |

**Content Types**: `MOVIE`, `TV_SERIES`

**Movie Fields**: `duration` (minutes), `director`  
**TV Series Fields**: `seasons`, `episodesPerSeason`

### Video Service (Port 8083)

**Base URL**: `/api/videos`

| Method | Endpoint | Description | Request Body |
|--------|----------|-------------|--------------|
| POST | `/watch` | Record watch event (Observer) | `{userId, contentId, progress, completed}` |
| POST | `/rate` | Rate content (Observer) | `{userId, contentId, score}` |
| GET | `/watch/user/{userId}` | Get watch history | - |
| GET | `/rate/user/{userId}` | Get user ratings | - |
| GET | `/watch/{id}` | Get watch event by ID | - |
| GET | `/rate/{id}` | Get rating by ID | - |
| GET | `/rate/content/{contentId}/average` | Get average rating | - |

**Progress**: Seconds watched  
**Score**: 0.0 to 5.0

### Recommendation Service (Port 8084)

**Base URL**: `/api/recommendations`

| Method | Endpoint | Description | Request Body |
|--------|----------|-------------|--------------|
| GET | `/{userId}?limit={n}` | Get recommendations (Strategy) | - |
| GET | `/trending?limit={n}` | Get trending content | - |
| GET | `/similar/{contentId}` | Get similar content | - |
| GET | `/preferences/{userId}` | Get user preferences | - |
| POST | `/preferences/update` | Update preferences | `{userId, contentId, interactionType}` |
| GET | `/strategy/demo` | Demo strategy pattern | - |
| GET | `/strategy/current/{userId}` | Get current strategy | - |

**Strategy Selection**:
- New user (no history) → `TrendingStrategy`
- User with watch history → `HistoryBasedStrategy`
- User with ratings → `RatingBasedStrategy`

---

## Testing

### Postman Collection

A comprehensive Postman collection with 67 requests is provided.

**Location**: `postman/StreamFlix_Complete_Collection.postman_collection.json`

**Collection Contents**:
1. Health Checks (6 requests)
2. User Service - Singleton Pattern (15 requests)
3. Content Service - Factory Pattern (15 requests)
4. Video Service - Observer Pattern (15 requests)
5. Recommendation Service - Strategy Pattern (10 requests)
6. Error Handling Tests (6 requests)

**How to Use**:
1. Open Postman
2. Import → File → Select `StreamFlix_Complete_Collection.postman_collection.json`
3. Run requests in order (health checks first)
4. Variables are auto-captured from responses

### Testing Flow

**Step 1: Health Checks**
```bash
curl http://localhost:8080/actuator/health
```

**Step 2: Register User**
```bash
curl -X POST http://localhost:8080/api/users/register \
  -H "Content-Type: application/json" \
  -d '{"username":"john_doe","email":"john@example.com","password":"Password123!","tier":"PREMIUM"}'
```

**Step 3: Create Movie (Factory Pattern)**
```bash
curl -X POST http://localhost:8080/api/content \
  -H "Content-Type: application/json" \
  -d '{"type":"MOVIE","title":"Inception","description":"Dream thriller","genre":"Sci-Fi","releaseYear":2010,"duration":148,"director":"Christopher Nolan"}'
```

**Step 4: Record Watch Event (Observer Pattern)**
```bash
curl -X POST http://localhost:8080/api/videos/watch \
  -H "Content-Type: application/json" \
  -d '{"userId":1,"contentId":1,"progress":3600,"completed":true}'
```

**Step 5: Rate Content**
```bash
curl -X POST http://localhost:8080/api/videos/rate \
  -H "Content-Type: application/json" \
  -d '{"userId":1,"contentId":1,"score":5.0}'
```

**Step 6: Get Recommendations (Strategy Pattern)**
```bash
curl http://localhost:8080/api/recommendations/1?limit=5
```

**Step 7: Test Singleton Pattern**
```bash
curl http://localhost:8080/api/users/demo/singleton-test
```

### Viewing Logs

```bash
# View all logs
docker-compose logs -f

# View specific service
docker-compose logs -f user-service

# View pattern-specific logs
docker-compose logs user-service | grep "ConfigurationManager"  # Singleton
docker-compose logs content-service | grep -i "factory"         # Factory
docker-compose logs video-service | grep -i "observer"          # Observer
docker-compose logs recommendation-service | grep -i "strategy" # Strategy
```

---

## Message Queue Integration (RabbitMQ)

### Overview

**RabbitMQ** is integrated for asynchronous communication between **Video Service** and **Recommendation Service**. This replaces synchronous REST calls with a message queue, improving scalability, decoupling, and fault tolerance.

### Architecture

```
Video Service (Publisher)          RabbitMQ Queue          Recommendation Service (Consumer)
     |                                    |                              |
     |  User watches/rates content        |                              |
     |  ────────────────────────────────>|                              |
     |                                    |  Message queued              |
     |                                    |  ───────────────────────────>|
     |                                    |                              |  Update preferences
     |                                    |                              |  in database
```

### Queue Configuration

- **Queue Name**: `user.preference.updates`
- **Type**: Durable queue (survives broker restarts)
- **Message Format**: JSON (using Jackson2JsonMessageConverter)
- **Publisher**: Video Service (`MessageQueuePublisher`)
- **Consumer**: Recommendation Service (`UserPreferenceMessageConsumer`)

### Message Flow

1. **User watches content** or **rates content** via Video Service
2. Video Service saves event to its database (`videodb`)
3. Video Service publishes `UserPreferenceMessage` to RabbitMQ queue
4. Recommendation Service consumes message asynchronously
5. Recommendation Service updates user preferences in its database (`recommendationdb`)

### Benefits

#### 1. **Decoupling**
- Video Service doesn't need to know Recommendation Service details
- Services can be deployed/updated independently
- No direct dependency between services

#### 2. **Scalability**
- Messages are queued and processed asynchronously
- Can handle traffic spikes without blocking
- Multiple consumers can process messages in parallel

#### 3. **Fault Tolerance**
- If Recommendation Service is down, messages are queued
- Messages are processed when service comes back online
- No data loss during service outages

#### 4. **Performance**
- Non-blocking: Video Service doesn't wait for Recommendation Service
- Watch/rating operations complete immediately
- Background processing doesn't slow down user experience

### Configuration

**Video Service** (`video-service/src/main/resources/application.yml`):
```yaml
spring:
  rabbitmq:
    host: ${SPRING_RABBITMQ_HOST:localhost}
    port: ${SPRING_RABBITMQ_PORT:5672}
    username: ${SPRING_RABBITMQ_USERNAME:admin}
    password: ${SPRING_RABBITMQ_PASSWORD:admin123}
```

**Recommendation Service** (`recommendation-service/src/main/resources/application.yml`):
```yaml
spring:
  rabbitmq:
    host: ${SPRING_RABBITMQ_HOST:localhost}
    port: ${SPRING_RABBITMQ_PORT:5672}
    username: ${SPRING_RABBITMQ_USERNAME:admin}
    password: ${SPRING_RABBITMQ_PASSWORD:admin123}
    listener:
      simple:
        retry:
          enabled: true
          initial-interval: 1000
          max-attempts: 3
```

### Monitoring RabbitMQ

**Access RabbitMQ Management UI:**
```
http://localhost:15672
Username: admin
Password: admin123
```

**View queue status:**
- Navigate to "Queues" tab
- Check `user.preference.updates` queue
- Monitor message rates, consumers, and queue depth

**View messages in queue:**
```bash
# Check RabbitMQ logs
docker compose logs -f rabbitmq

# Check queue from command line
docker exec -it streamflix-rabbitmq rabbitmqctl list_queues
```

### Testing Message Queue

**1. Record a watch event:**
```bash
curl -X POST http://localhost:8080/api/videos/watch \
  -H "Content-Type: application/json" \
  -d '{"userId":1,"contentId":1,"progress":3600,"completed":true}'
```

**2. Check Video Service logs:**
```bash
docker compose logs video-service | grep -i "publishing\|rabbitmq"
```

**3. Check Recommendation Service logs:**
```bash
docker compose logs recommendation-service | grep -i "received\|queue"
```

**4. Verify message was processed:**
```bash
# Check user preferences were updated
curl http://localhost:8080/api/recommendations/preferences/1
```

### Code Locations

- **Publisher**: `video-service/src/main/java/com/example/videoservice/messaging/MessageQueuePublisher.java`
- **Consumer**: `recommendation-service/src/main/java/com/example/recommendationservice/messaging/UserPreferenceMessageConsumer.java`
- **Message DTO**: `video-service/src/main/java/com/example/videoservice/messaging/UserPreferenceMessage.java`
- **Configuration**: `video-service/src/main/java/com/example/videoservice/config/RabbitMQConfig.java`

---

## CI/CD Pipeline

### Overview

**GitHub Actions** CI/CD pipeline automatically builds, tests, and deploys all microservices.

### Pipeline Stages

#### 1. **Build and Test** (`build-and-test`)
- Checks out code
- Sets up JDK 17
- Builds all 5 services (User, Content, Video, Recommendation, API Gateway)
- Runs unit tests for each service
- Uploads JAR artifacts

#### 2. **Docker Build** (`docker-build`)
- Downloads JAR artifacts
- Builds Docker images for all services
- Verifies images were created successfully

#### 3. **Deploy to Local** (`deploy-local`)
- Starts all services with Docker Compose
- Waits for services to be healthy
- Checks service health status
- Runs only on `main` or `master` branch

#### 4. **Integration Tests** (`integration-test`)
- Tests API Gateway health endpoint
- Tests all service health endpoints
- Verifies services are responding
- Runs only on `main` or `master` branch

### Pipeline Configuration

**Location**: `.github/workflows/ci-cd.yml`

**Triggers:**
- Push to `main`, `master`, or `develop` branches
- Pull requests to `main`, `master`, or `develop` branches

### Running the Pipeline

#### Automatic Execution
1. Push code to GitHub repository
2. Pipeline runs automatically
3. View status in GitHub Actions tab

#### Manual Execution
1. Go to GitHub repository
2. Click "Actions" tab
3. Select "CI/CD Pipeline" workflow
4. Click "Run workflow"
5. Select branch and click "Run workflow"

### Viewing Pipeline Results

**In GitHub:**
1. Navigate to repository
2. Click "Actions" tab
3. Select workflow run
4. View logs for each job

**Pipeline Status:**
- Green checkmark = Success
- Red X = Failure
- Yellow circle = In progress

### Local Testing

**Run pipeline steps locally:**

```bash
# 1. Build all services
cd user-service && mvn clean package -DskipTests
cd ../content-service && mvn clean package -DskipTests
cd ../video-service && mvn clean package -DskipTests
cd ../recommendation-service && mvn clean package -DskipTests
cd ../api-gateway && mvn clean package -DskipTests

# 2. Build Docker images
docker compose build

# 3. Deploy services
docker compose up -d

# 4. Check health
docker compose ps
curl http://localhost:8080/actuator/health
```

### Pipeline Artifacts

**JAR Files:**
- Uploaded after build step
- Available for download from GitHub Actions
- Retained for 7 days

**Docker Images:**
- Built during Docker Build step
- Available locally after deployment
- Tagged with service names

### Troubleshooting

**Build fails:**
- Check Maven dependencies
- Verify Java 17 is available
- Review build logs in GitHub Actions

**Docker build fails:**
- Ensure Docker is running
- Check Dockerfile syntax
- Verify JAR files exist

**Deployment fails:**
- Check Docker Compose configuration
- Verify all services start correctly
- Review service logs: `docker compose logs`

**Tests fail:**
- Review test output in logs
- Check test configuration
- Verify dependencies

### Customization

**Modify pipeline:**
1. Edit `.github/workflows/ci-cd.yml`
2. Commit and push changes
3. Pipeline runs with new configuration

**Add new stages:**
- Add new job in `jobs` section
- Define dependencies with `needs`
- Configure triggers and conditions

**Environment variables:**
- Add to `env` section at top level
- Use in steps: `${{ env.VARIABLE_NAME }}`

---

## Project Structure

```text
streamflix/
├── api-gateway/                         
│   ├── src/main/java/com/example/apigateway/
│   │   ├── ApiGatewayApplication.java    
│   │   ├── config/                       
│   │   │   └── GatewayConfig.java
│   │   ├── controller/                   
│   │   │   ├── FallbackController.java
│   │   │   └── HealthController.java
│   │   └── filter/                       
│   │       └── RequestLoggingFilter.java
│   ├── src/main/resources/
│   │   ├── application.yml               
│   │   └── static/ui/                   
│   │       ├── app.js
│   │       ├── index.html
│   │       └── styles.css
│   ├── Dockerfile
│   └── pom.xml
│
├── user-service/                         
│   ├── src/main/java/com/example/userservice/
│   │   ├── UserServiceApplication.java   
│   │   ├── config/
│   │   │   ├── ConfigurationManager.java 
│   │   │   └── RabbitMQConfig.java       
│   │   ├── controller/
│   │   │   └── UserController.java       
│   │   ├── dto/                          
│   │   │   ├── LoginDTO.java
│   │   │   ├── LoginResponseDTO.java
│   │   │   ├── RegisterDTO.java
│   │   │   ├── SingletonTestResponse.java
│   │   │   ├── SubscriptionUpdateDTO.java
│   │   │   └── UserDTO.java
│   │   ├── exception/                    
│   │   │   ├── DuplicateEmailException.java
│   │   │   ├── DuplicateUsernameException.java
│   │   │   ├── GlobalExceptionHandler.java
│   │   │   ├── InvalidCredentialsException.java
│   │   │   └── UserNotFoundException.java
│   │   ├── messaging/                    
│   │   │   ├── ContentEventMessage.java
│   │   │   ├── ContentEventMessageConsumer.java
│   │   │   ├── MessageQueuePublisher.java
│   │   │   └── UserEventMessage.java
│   │   ├── model/
│   │   │   └── User.java                 
│   │   ├── repository/
│   │   │   └── UserRepository.java
│   │   └── service/
│   │       └── UserService.java          
│   ├── src/main/resources/
│   │   └── application.yml               
│   ├── Dockerfile
│   └── pom.xml
│
├── content-service/                      
│   ├── src/main/java/com/example/contentservice/
│   │   ├── ContentServiceApplication.java
│   │   ├── config/
│   │   │   └── RabbitMQConfig.java      
│   │   ├── controller/
│   │   │   └── ContentController.java
│   │   ├── dto/
│   │   │   ├── ContentCreateDTO.java
│   │   │   ├── ContentResponseDTO.java
│   │   │   └── ContentUpdateDTO.java
│   │   ├── exception/
│   │   │   ├── ContentNotFoundException.java
│   │   │   ├── ContentValidationException.java
│   │   │   ├── GlobalExceptionHandler.java
│   │   │   └── InvalidContentTypeException.java
│   │   ├── factory/                      
│   │   │   ├── ContentFactory.java       
│   │   │   ├── MovieFactory.java         
│   │   │   └── TVSeriesFactory.java     
│   │   ├── messaging/                    
│   │   │   ├── ContentEventMessage.java
│   │   │   ├── MessageQueuePublisher.java
│   │   │   ├── UserEventMessage.java
│   │   │   └── UserEventMessageConsumer.java
│   │   ├── model/                       
│   │   │   ├── Content.java
│   │   │   ├── Movie.java
│   │   │   └── TVSeries.java
│   │   ├── repository/
│   │   │   └── ContentRepository.java
│   │   └── service/
│   │       └── ContentService.java
│   ├── src/main/resources/
│   │   └── application.yml
│   ├── Dockerfile
│   └── pom.xml
│
├── video-service/                        
│   ├── src/main/java/com/example/videoservice/
│   │   ├── VideoServiceApplication.java
│   │   ├── config/
│   │   │   ├── RabbitMQConfig.java       
│   │   │   └── RestTemplateConfig.java   
│   │   ├── controller/
│   │   │   └── VideoController.java
│   │   ├── dto/
│   │   │   ├── ContentResponseDTO.java
│   │   │   ├── RatingCreateDTO.java
│   │   │   ├── RatingResponseDTO.java
│   │   │   ├── WatchEventCreateDTO.java
│   │   │   └── WatchEventResponseDTO.java
│   │   ├── exception/
│   │   │   ├── ContentNotFoundException.java
│   │   │   └── GlobalExceptionHandler.java
│   │   ├── messaging/                    
│   │   │   ├── ContentEventMessage.java
│   │   │   ├── ContentEventMessageConsumer.java
│   │   │   ├── MessageQueuePublisher.java
│   │   │   └── UserPreferenceMessage.java
│   │   ├── model/
│   │   │   ├── Rating.java
│   │   │   └── WatchEvent.java
│   │   ├── observer/                     
│   │   │   ├── AnalyticsObserver.java
│   │   │   ├── ContentRatedEvent.java
│   │   │   ├── EventObserver.java
│   │   │   ├── RecommendationUpdateObserver.java
│   │   │   ├── VideoEvent.java
│   │   │   ├── VideoEventPublisher.java
│   │   │   └── VideoWatchedEvent.java
│   │   ├── repository/
│   │   │   ├── RatingRepository.java
│   │   │   └── WatchEventRepository.java
│   │   └── service/
│   │       └── VideoService.java
│   ├── src/main/resources/
│   │   └── application.yml
│   ├── Dockerfile
│   └── pom.xml
│
├── recommendation-service/               
│   ├── src/main/java/com/example/recommendationservice/
│   │   ├── RecommendationServiceApplication.java
│   │   ├── config/
│   │   │   ├── RabbitMQConfig.java      
│   │   │   └── RestTemplateConfig.java
│   │   ├── controller/
│   │   │   └── RecommendationController.java
│   │   ├── dto/
│   │   │   ├── ContentResponseDTO.java
│   │   │   ├── RecommendationResponseDTO.java
│   │   │   └── UserPreferenceUpdateDTO.java
│   │   ├── exception/
│   │   │   ├── GlobalExceptionHandler.java
│   │   │   ├── RecommendationException.java
│   │   │   └── UserPreferenceNotFoundException.java
│   │   ├── messaging/                    
│   │   │   ├── ContentEventMessage.java
│   │   │   ├── ContentEventMessageConsumer.java
│   │   │   ├── UserEventMessage.java
│   │   │   ├── UserEventMessageConsumer.java
│   │   │   ├── UserPreferenceMessage.java
│   │   │   └── UserPreferenceMessageConsumer.java
│   │   ├── model/
│   │   │   └── UserPreference.java
│   │   ├── repository/
│   │   │   └── RecommendationRepository.java
│   │   ├── service/
│   │   │   ├── RecommendationEngine.java
│   │   │   └── RecommendationService.java
│   │   └── strategy/                     
│   │       ├── HistoryBasedStrategy.java
│   │       ├── RatingBasedStrategy.java
│   │       ├── RecommendationStrategy.java
│   │       └── TrendingStrategy.java
│   ├── src/main/resources/
│   │   └── application.yml
│   ├── Dockerfile
│   └── pom.xml
│
├── postman/                             
│   ├── StreamFlix_Complete_Collection.postman_collection.json
│   
│
├── docs/
│   └── diagrams/                         
│       ├── ClassDiagram.png
│       ├── SeqDiagram1.png
│       └── SeqDiagram2.png
│
├── docker-compose.yml                     
├── build-all.bat                        
├── README.md                             
├── pom.xml                              
└── src/                                  
    ├── main/java/com/example/            
    └── main/resources/                
```
## Team

**Bilciurescu Elena-Alina** - 1241EA  
**Solomon Miruna-Maria** - 1241EA  
**Toma Daria-Maria** - 1241EA


