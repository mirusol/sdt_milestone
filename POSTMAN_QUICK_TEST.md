# Quick Postman Test Guide - Database Verification

## ✅ All Services Are Running!
All 9 containers are healthy and ready to use.

---

## 🚀 Quick Test Sequence (Copy-Paste Ready)

### Step 1: Health Check (Verify Services)
**GET** `http://localhost:8080/actuator/health`

---

### Step 2: Register a User (Tests User Database)
**POST** `http://localhost:8080/api/users/register`
**Headers:** `Content-Type: application/json`
**Body:**
```json
{
  "username": "test_user",
  "email": "test@example.com",
  "password": "Test123!",
  "tier": "PREMIUM"
}
```

**Expected Response:** User object with `id`, `username`, `email`, `tier`
**Database:** Saves to `streamflix_user` database (PostgreSQL port 5432)

---

### Step 3: Get All Users (Verify User Database Read)
**GET** `http://localhost:8080/api/users/`

**Expected Response:** Array of all users
**Database:** Reads from `streamflix_user` database

---

### Step 4: Create a Movie (Tests Content Database - Factory Pattern)
**POST** `http://localhost:8080/api/content`
**Headers:** `Content-Type: application/json`
**Body:**
```json
{
  "type": "MOVIE",
  "title": "Inception",
  "description": "A mind-bending thriller",
  "genre": "Sci-Fi",
  "releaseYear": 2010,
  "duration": 148,
  "director": "Christopher Nolan"
}
```

**Expected Response:** Movie object with `id`, `title`, `type`, `duration`, `director`
**Database:** Saves to `streamflix_content` database (PostgreSQL port 5433)

---

### Step 5: Create a TV Series (Tests Content Database - Factory Pattern)
**POST** `http://localhost:8080/api/content`
**Headers:** `Content-Type: application/json`
**Body:**
```json
{
  "type": "TV_SERIES",
  "title": "Breaking Bad",
  "description": "A high school chemistry teacher turned methamphetamine manufacturer",
  "genre": "Drama",
  "releaseYear": 2008,
  "seasons": 5,
  "episodesPerSeason": 16
}
```

**Expected Response:** TV Series object with `id`, `title`, `type`, `seasons`, `episodesPerSeason`
**Database:** Saves to `streamflix_content` database

---

### Step 6: Get All Content (Verify Content Database Read)
**GET** `http://localhost:8080/api/content/`

**Expected Response:** Array of all content (movies and TV series)
**Database:** Reads from `streamflix_content` database

---

### Step 7: Record Watch Event (Tests Video Database - Observer Pattern)
**POST** `http://localhost:8080/api/videos/watch`
**Headers:** `Content-Type: application/json`
**Body:**
```json
{
  "userId": 1,
  "contentId": 1,
  "progress": 3600,
  "completed": true
}
```

**Note:** Use the `id` from Step 2 for `userId` and `id` from Step 4/5 for `contentId`

**Expected Response:** Watch event object with `id`, `userId`, `contentId`, `progress`, `completed`
**Database:** Saves to `streamflix_video` database (PostgreSQL port 5434)
**Observer Pattern:** Triggers multiple observers (check logs!)

---

### Step 8: Rate Content (Tests Video Database - Observer Pattern)
**POST** `http://localhost:8080/api/videos/rate`
**Headers:** `Content-Type: application/json`
**Body:**
```json
{
  "userId": 1,
  "contentId": 1,
  "score": 5.0
}
```

**Expected Response:** Rating object with `id`, `userId`, `contentId`, `score`
**Database:** Saves to `streamflix_video` database
**Observer Pattern:** Triggers observers to update recommendations

---

### Step 9: Get Watch History (Verify Video Database Read)
**GET** `http://localhost:8080/api/videos/watch/user/1`

**Expected Response:** Array of watch events for user 1
**Database:** Reads from `streamflix_video` database

---

### Step 10: Get Recommendations (Tests Recommendation Database - Strategy Pattern)
**GET** `http://localhost:8080/api/recommendations/1?limit=5`

**Note:** Use the `userId` from Step 2

**Expected Response:** Array of recommended content
**Database:** Reads/writes to `streamflix_recommendation` database (PostgreSQL port 5435)
**Strategy Pattern:** Dynamically selects algorithm based on user activity

---

## 🔍 Verify Data in Databases Directly

### Connect to PostgreSQL Databases

**User Database (Port 5432):**
```powershell
docker exec -it streamflix-postgres-user psql -U user -d userdb
```
Then run: `SELECT * FROM users;`

**Content Database (Port 5433):**
```powershell
docker exec -it streamflix-postgres-content psql -U user -d contentdb
```
Then run: `SELECT * FROM content;`

**Video Database (Port 5434):**
```powershell
docker exec -it streamflix-postgres-video psql -U user -d videodb
```
Then run: `SELECT * FROM watch_event;` and `SELECT * FROM rating;`

**Recommendation Database (Port 5435):**
```powershell
docker exec -it streamflix-postgres-recommendation psql -U user -d recommendationdb
```
Then run: `SELECT * FROM user_preference;`

---

## 📋 Complete Test Flow

1. ✅ Health Check → Verify all services
2. ✅ Register User → Creates user in database
3. ✅ Get All Users → Reads from database
4. ✅ Create Movie → Factory pattern + database write
5. ✅ Create TV Series → Factory pattern + database write
6. ✅ Get All Content → Reads from database
7. ✅ Record Watch → Observer pattern + database write
8. ✅ Rate Content → Observer pattern + database write
9. ✅ Get Watch History → Reads from database
10. ✅ Get Recommendations → Strategy pattern + database read/write

---

## 🎯 Import Full Postman Collection

For the complete collection with 30+ requests:
1. Open Postman
2. Click **Import**
3. Select file: `postman/StreamFlix_Essential_Collection.postman_collection.json`
4. All requests are ready to use!

---

## 🐛 Check Logs to See Database Operations

```powershell
# View all logs
docker compose logs -f

# View specific service logs
docker compose logs -f user-service
docker compose logs -f content-service
docker compose logs -f video-service
```

You'll see SQL queries like:
- `Hibernate: insert into users...`
- `Hibernate: select * from content...`
- `Hibernate: insert into watch_event...`

