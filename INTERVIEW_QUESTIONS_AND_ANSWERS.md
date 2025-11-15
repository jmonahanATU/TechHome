# TechHome - Interview Questions and Answers

## Table of Contents
1. [General Project Overview](#general-project-overview) (Q1-Q2)
2. [Architecture & Tech Stack](#architecture--tech-stack) (Q3-Q7)
3. [Backend Development](#backend-development) (Q8-Q12)
4. [Frontend Development](#frontend-development) (Q13-Q15)
5. [Integration & Hardware](#integration--hardware) (Q16-Q17)
6. [Database & Data Management](#database--data-management) (Q18-Q20)
7. [Security & Best Practices](#security--best-practices) (Q21-Q23)
8. [Recent Features & Contributions](#recent-features--contributions) (Q24-Q25)
9. [Advanced Technical Questions](#advanced-technical-questions) (Q26-Q29)
10. [Soft Skills & Project Management](#soft-skills--project-management) (Q30-Q31)
11. [Bonus: Behavioral Questions](#bonus-behavioral-questions) (Q32-Q34)

---

## GENERAL PROJECT OVERVIEW

### Q1: Can you describe the TechHome project?

**Answer:** TechHome is a full-stack smart home management system that allows users to control smart devices through a cross-platform mobile app. It features AI-powered automation using machine learning, voice assistant integration, comprehensive analytics dashboards, and integrates with physical hardware via Raspberry Pi running Home Assistant. The system supports device control (lights, thermostats), time-based automation, usage prediction, and detailed analytics with error tracking.

### Q2: What problem does TechHome solve?

**Answer:** TechHome centralizes smart home device management into a single mobile application, eliminating the need to use multiple apps for different devices. It adds intelligence through ML-based usage prediction, automates routine tasks, provides detailed analytics on device usage patterns, and tracks device health with error diagnostics to help users maintain their smart home ecosystem.

---

## ARCHITECTURE & TECH STACK

### Q3: What's the overall architecture of TechHome?

**Answer:** It's a three-tier architecture:
- **Frontend:** React Native mobile app built with Expo (~50.0.0) for cross-platform support (iOS/Android/Web)
- **Backend:** Flask REST API (Python) using Blueprint pattern for modular routing, with JWT authentication
- **Database:** MongoDB (Atlas cloud with local fallback) storing users, devices, rooms, automations, logs, and ML training data
- **Hardware Layer:** Raspberry Pi 3 running Home Assistant for controlling actual smart devices over the local network

### Q4: Why did you choose React Native for the frontend?

**Answer:** React Native with Expo provides cross-platform development (single codebase for iOS, Android, and Web), rapid development with hot reloading, extensive component library, and strong community support. Expo specifically simplifies deployment and adds features like speech recognition without complex native configuration.

### Q5: Why Flask over other Python frameworks?

**Answer:** Flask is lightweight and flexible, ideal for building REST APIs. Its Blueprint system allowed us to modularize routes (auth, devices, analytics, etc.), making the codebase maintainable. It integrates well with our ML libraries (scikit-learn) and has excellent extensions for JWT authentication and CORS handling.

### Q6: Explain your database choice and schema design.

**Answer:** We chose MongoDB for its flexible schema, which suits smart home devices with varying attributes. Key collections:
- `users` - Authentication with bcrypt password hashing
- `devices` - Device registry with room relationships
- `device_logs` - Action logging with user attribution for analytics
- `device_history` - Historical state changes for ML training
- `automations` - Automation rules
- `refresh_tokens` - JWT token management

### Q7: Explain the pub/sub (publish/subscribe) architecture in your system.

**Answer:** TechHome implements **multiple pub/sub patterns** throughout the architecture:

**1. Custom Event Bus (Frontend)**
Located in `TechHome/context/DeviceContext.js:8-25`, we implemented a lightweight observer pattern:
```javascript
const DeviceEvents = {
    listeners: new Map(),
    subscribe: (event, callback) => { /* ... */ },
    emit: (event, data) => { /* ... */ }
};
```
This allows components to subscribe to events like `DEVICE_TOGGLED` and `DEVICES_UPDATED`, enabling reactive UI updates without prop drilling.

**2. React Context Provider Pattern**
We use React's Context API as a hierarchical pub/sub system with `ThemeProvider > AuthProvider > DeviceProvider`. When state changes in a provider, all consuming components are automatically notified and re-render.

**3. Adaptive Polling System**
Since true real-time WebSockets aren't yet implemented, we use intelligent polling in `DeviceContext.js:239-269`:
- **Immediate (500ms)**: For 3 seconds after user actions
- **Regular (5s)**: For 30 seconds during active sessions
- **Background (30s)**: During idle periods

This adaptive approach reduces server load while maintaining responsiveness.

**4. APScheduler (Backend Event System)**
The `scheduler.py` implements time-based pub/sub for automations. Cron-like triggers publish events at scheduled times, which execute device commands asynchronously.

**5. HTTP Interceptor Pattern**
Axios response interceptors act as a pub/sub mechanism for authentication events. All HTTP requests subscribe to the interceptor, which handles 401 errors globally by refreshing tokens.

**Current Limitations & Future Plans:**
- Currently relies on polling instead of true real-time communication
- Planned: WebSocket implementation using Flask-SocketIO for instant device state updates
- Planned: Server-Sent Events (SSE) for one-way server-to-client notifications
- Consideration: Redis pub/sub for multi-instance backend scalability

**Why this matters:** The pub/sub architecture decouples components, making the system more maintainable and enabling features like multi-device synchronization and real-time notifications.

---

## BACKEND DEVELOPMENT

### Q8: How does authentication work in your system?

**Answer:** We use JWT-based dual-token authentication implemented with Flask-JWT-Extended:
- **Login:** User provides credentials, receives access token (1-hour expiry) and refresh token (30-day expiry)
- **Access:** Protected endpoints require `@jwt_required()` decorator
- **Refresh:** Client uses refresh token to get new access token without re-login
- **Logout:** Refresh tokens stored in `refresh_tokens` collection are deleted on logout
- **Security:** Passwords hashed with bcrypt, input validation on registration (email format, password strength)

Located in `backend/routes/auth_routes.py`.

### Q9: Describe the device management system.

**Answer:** The device management system (`backend/routes/device_routes.py` - 849 lines) provides:
- **CRUD operations** for devices with room assignments
- **State toggling** with Home Assistant integration
- **Temperature control** for thermostats
- **Bulk operations** like toggle-all-lights
- **Device diagnostics** including refresh, reset, connectivity testing
- **State synchronization** between MongoDB and Home Assistant
- **Error handling** with 1-1.5s timeouts and comprehensive logging

Each operation is logged to `device_logs` with user attribution for analytics.

### Q10: How does the automation system work?

**Answer:** The automation engine uses APScheduler for background task scheduling:
- **Rule Creation:** Users define time-based or condition-based triggers with actions (toggle/turn_on/turn_off)
- **Storage:** Rules stored in `automations` collection
- **Scheduling:** `scheduler.py` (96 lines) loads all enabled automations at startup
- **Execution:** Jobs trigger at specified times, execute device commands
- **Auto-sync:** CRUD operations on automations automatically reschedule jobs
- **Enable/Disable:** Toggle functionality without deletion

Located in `backend/routes/automation_routes.py` and `backend/scheduler.py`.

### Q11: Explain the machine learning implementation.

**Answer:** We use **Random Forest Classifier** (scikit-learn) for device usage prediction:
- **Features:** Hour of day, day of week, weekend flag, previous state
- **Training Data:** From `device_history` collection (requires 10+ entries)
- **Per-Device Models:** Separate model for each device, persisted as `.joblib` files
- **Preprocessing:** StandardScaler pipeline for feature normalization
- **Suggestions:** Only shown if confidence >70%
- **Feedback Loop:** Users can provide feedback to improve predictions
- **Retraining:** Triggered when new history entries added

Implemented in `backend/ml_models.py` (230 lines) and `backend/routes/ml_routes.py`.

### Q12: What's the most complex part of your backend?

**Answer:** The analytics system (`backend/routes/analytics_routes.py` - **1,757 lines**) is the most complex:
- **Usage Analytics:** Per-user, per-device, hourly/daily/weekly/monthly breakdowns with drill-down capability
- **Error Tracking:** Device health monitoring (healthy/warning/critical), error categorization, troubleshooting suggestions
- **User Engagement:** Streak tracking, gamification badges (Bronze/Silver/Gold/Platinum), ranking system
- **Filtering:** Complex timezone-aware (Europe/Dublin) date range queries with user/device/room filters
- **CSV Export:** Grouped data export with readable device names
- **Performance:** Optimized queries with proper indexing

---

## FRONTEND DEVELOPMENT

### Q13: How is state managed in the React Native app?

**Answer:** We use React Context API for global state:
- **AuthContext:** Manages user authentication state, login/logout, token storage via AsyncStorage
- **DeviceContext:** Manages device list, CRUD operations, state updates
- **ThemeContext:** Handles dark/light mode theming

Custom hooks abstract logic:
- **useDevices:** Device operations and state management
- **useHomeAssistantDevices:** HA integration
- **useLeonAssistant:** Voice control

### Q14: Describe the navigation structure.

**Answer:** Hybrid navigation using React Navigation:
- **Bottom Tab Navigator:** Main navigation (Home, Devices, Automations, Analytics, Settings)
- **Native Stack Navigator:** For nested screens (Login → Home, Device Details, Room Management)
- **Conditional Rendering:** Shows Login screen if not authenticated, otherwise shows main tabs

Defined in `TechHome/App.js`.

### Q15: What's the analytics dashboard like?

**Answer:** The `AnalyticsDashboardScreen.js` (110KB file) provides:
- **Charts:** React Native Chart Kit for bar/line charts
- **Views:** Daily (hourly), Weekly (daily), Monthly (daily) with interactive drill-down
- **Metrics:** Usage per user/device, recent actions, error statistics, device health
- **User Engagement:** Active users with streaks and badge rankings
- **Filters:** Unified filter modal for date range, user, device, room
- **Export:** CSV download functionality
- **Visual Feedback:** Color-coded charts (blue/orange/purple), contextual "no data" messages

---

## INTEGRATION & HARDWARE

### Q16: How does Home Assistant integration work?

**Answer:** Integration via REST API (`backend/routes/home_assistant_routes.py`):
- **Discovery:** Fetch all device states from HA API
- **Control:** Send toggle commands to specific entities
- **Sync:** Keep MongoDB and HA states synchronized
- **Error Handling:** Network timeout handling (1.5s), connection error detection
- **Device Flag:** `isHomeAssistant` boolean in device schema
- **Entity Mapping:** HA `entity_id` stored for API calls

The Raspberry Pi 3 runs Home Assistant locally, communicating with actual smart devices (lights, thermostats) over the network.

### Q17: What hardware have you integrated?

**Answer:** Raspberry Pi 3 running Home Assistant OS acts as the smart home hub. It:
- Controls physical smart devices over local network (Zigbee, Z-Wave, WiFi)
- Exposes REST API for device control
- Provides state management and device registry
- Handles low-level device communication protocols
- Serves as bridge between our app and actual hardware

Documented in `README_david_contribution.md`.

---

## DATABASE & DATA MANAGEMENT

### Q18: How do you handle timezone issues?

**Answer:** Comprehensive timezone strategy:
- **Storage:** All timestamps stored in UTC in MongoDB
- **Display:** Convert to Europe/Dublin timezone (pytz) for user-facing analytics
- **Queries:** Date range filters are timezone-aware, converting user input to UTC bounds
- **Consistency:** All backend operations use UTC, conversion only at presentation layer

Critical for analytics accuracy in `backend/routes/analytics_routes.py`.

### Q19: Explain your logging system.

**Answer:** The `device_logs` collection tracks all device interactions:
```javascript
{
  user: "username",
  device: "device_id or entity_id",
  action: "toggle/add/remove/set_temperature",
  result: "on/off/success/error: message",
  timestamp: DateTime (UTC),
  is_error: Boolean,
  error_type: "timeout/connection_error/permission_denied"
}
```
- **User Attribution:** Added JWT to device endpoints to track who performed actions
- **Automatic Error Detection:** Analyzes `result` strings for error patterns
- **Analytics Foundation:** Powers all usage and error analytics
- **Categorization:** Errors automatically categorized for troubleshooting

Implemented in `backend/models/device_log.py`.

### Q20: How do you ensure data quality for ML models?

**Answer:** Several safeguards:
- **Minimum Data:** Require 10+ historical entries before training
- **Data Validation:** Timestamp, state, and device_id validation
- **Feature Engineering:** Derive hour, day of week, weekend from timestamps
- **Normalization:** StandardScaler ensures feature consistency
- **Feedback Loop:** Users can correct predictions, improving future training
- **Model Versioning:** Joblib persistence allows model rollback if needed

---

## SECURITY & BEST PRACTICES

### Q21: What security measures are implemented?

**Answer:**
- **Authentication:** JWT dual-token system with short-lived access tokens (1 hour)
- **Password Security:** bcrypt hashing with automatic salt generation
- **Token Revocation:** Refresh token blocklist for logout functionality
- **Input Validation:** Email format, password strength (min length), username constraints
- **Protected Routes:** `@jwt_required()` decorator on sensitive endpoints
- **CORS:** Configured for specific origins, not wildcard
- **Environment Variables:** Sensitive config (MongoDB URI, JWT secret) in `.env`

### Q22: How do you handle errors and edge cases?

**Answer:**
- **Network Timeouts:** Short timeouts (1-1.5s) for HA requests to prevent hanging
- **Fallback Mechanisms:** MongoDB Atlas → Local MongoDB if cloud unavailable
- **Validation:** Input validation on all user-submitted data
- **Error Logging:** Comprehensive logging with error categorization
- **User Feedback:** Contextual error messages with troubleshooting suggestions
- **Device Health:** Automatic monitoring with warning/critical thresholds
- **Graceful Degradation:** Features degrade gracefully if dependencies unavailable

### Q23: Describe your testing strategy and implementation.

**Answer:** We have a comprehensive testing suite using **pytest** with **72 test functions** across **8 test files** (1,154 lines of test code):

**Test Files in `backend/tests/`:**
- `test_auth_routes.py` (189 lines) - 14 tests for authentication and user management
- `test_automation_routes.py` (217 lines) - 17 tests for automation CRUD and scheduling
- `test_device_routes.py` (144 lines) - 10 tests for device control and state management
- `test_home_assistant_routes.py` (155 lines) - 9 tests for HA integration
- `test_integration_flow.py` (151 lines) - End-to-end workflow tests
- `test_ml_routes.py` (114 lines) - 6 tests for ML predictions and feedback
- `test_room_routes.py` (109 lines) - 11 tests for room management
- `test_device_logging.py` (75 lines) - Device action logging tests

**Test Types:**
1. **Unit Tests (Majority):** Test individual endpoints in isolation with mocked dependencies
2. **Integration Tests:** 1 marked test (`@pytest.mark.integration`) for real HA API testing
3. **E2E Flow Tests:** Complete user workflows (register → create automation → trigger)
4. **Negative Tests:** 24 test functions specifically testing error conditions and edge cases

**Testing Best Practices:**
- **Fixtures:** Reusable `client` and `auth_headers` fixtures across all test files
- **Auto-Cleanup:** `@pytest.fixture(autouse=True)` with yield pattern for setup/teardown
- **Mock Isolation:** Using `unittest.mock.patch` for external HTTP requests (56 mock usages)
- **Test Data Naming:** MOCK_ prefix convention enables pattern-based cleanup
- **Real Database:** Tests use actual MongoDB (not mocked) to validate CRUD operations
- **Assertions:** 129 total assertions including 62 HTTP status code checks
- **Pytest Markers:** Custom markers for categorizing tests (integration, slow, etc.)

**Example Test Pattern:**
```python
@pytest.fixture
def auth_headers(client):
    # Register and login test user
    client.post('/api/auth/register', json={...})
    response = client.post('/api/auth/login', json={...})
    token = response.get_json()['access_token']
    return {'Authorization': f'Bearer {token}'}

def test_toggle_device(client, auth_headers):
    response = client.post(f'/api/devices/{device_id}/toggle',
                          headers=auth_headers)
    assert response.status_code == 200
```

**Coverage Statistics:**
- **Routes Tested:** 6 out of 7 route files (86%)
- **Gap:** Analytics routes (1,757 lines) currently have no tests - identified improvement area
- **Test-to-Code Ratio:** ~1:3 (excluding analytics)

**Areas for Improvement:**
- Add code coverage tracking (pytest-cov)
- Implement CI/CD pipeline (GitHub Actions)
- Create shared conftest.py to reduce fixture duplication
- Add parametrized tests using `@pytest.mark.parametrize`
- Write tests for analytics endpoints
- Add performance/load testing

---

## RECENT FEATURES & CONTRIBUTIONS

### Q24: What are the most recent features you added?

**Answer:** Based on recent commits:
1. **User Engagement System:** Ranking with Bronze/Silver/Gold/Platinum badges based on device control frequency
2. **Streak Tracking:** Current and longest usage streaks for gamification
3. **Device Health Monitoring:** Healthy/Warning/Critical status based on error rates (<10%, 10-20%, >20%)
4. **Error Diagnostics:** Comprehensive error tracking with quick action buttons (Retry, Refresh, Ping, Reset)
5. **Multi-View Analytics:** Daily/Weekly/Monthly drill-down with interactive charts

Commits show continuous improvement to analytics and user experience.

### Q25: What challenges did you face and overcome?

**Answer:**
1. **User Attribution for Logs:** Initially device endpoints didn't track users. Added JWT to log who performed actions for analytics.
2. **Timezone Complexity:** Analytics showed incorrect data due to UTC/local mismatch. Implemented timezone-aware queries.
3. **Error Detection:** Manually categorizing errors was inefficient. Created automatic detection from result strings.
4. **Device Health:** Users couldn't identify problematic devices. Built health scoring system based on error rates.
5. **Test Database Cleanup:** Tests interfered with each other. Implemented proper isolation and teardown.

---

## ADVANCED TECHNICAL QUESTIONS

### Q26: How would you scale this system for 10,000 users?

**Answer:**
- **Database:** Add indexes on frequently queried fields (user_id, device_id, timestamp)
- **Caching:** Implement Redis for device states, reducing MongoDB queries
- **Load Balancing:** Deploy multiple Flask instances behind nginx
- **ML Optimization:** Pre-train models, cache predictions, use scheduled batch updates
- **Analytics:** Pre-aggregate common queries (daily/weekly stats), use materialized views
- **API Rate Limiting:** Implement rate limiting to prevent abuse
- **CDN:** Serve static assets via CDN
- **Microservices:** Split analytics, ML, and device control into separate services

### Q27: Explain the API design philosophy.

**Answer:** RESTful principles:
- **Resource-Based URLs:** `/api/devices/:id`, `/api/rooms/:id`
- **HTTP Methods:** GET (read), POST (create), PUT/PATCH (update), DELETE (delete)
- **Status Codes:** 200 (success), 201 (created), 400 (bad request), 401 (unauthorized), 404 (not found), 500 (server error)
- **JSON Responses:** Consistent structure with `success`, `data`, `error` fields
- **Stateless:** Each request contains all necessary authentication (JWT)
- **Versioned:** `/api/v1/` prefix for future compatibility (not yet implemented)

### Q28: How do you ensure data consistency between MongoDB and Home Assistant?

**Answer:**
- **Single Source of Truth:** HA is authoritative for device states
- **Periodic Sync:** Refresh endpoints update MongoDB from HA
- **Optimistic Updates:** UI updates immediately, then syncs with HA
- **Error Handling:** If HA update fails, log error but keep MongoDB state
- **Reconciliation:** Manual refresh option for users
- **State Validation:** Compare expected vs actual state after operations

### Q29: Describe the ML model lifecycle.

**Answer:**
1. **Data Collection:** Device state changes logged to `device_history`
2. **Preprocessing:** Extract features (hour, day, weekend, prev_state), normalize with StandardScaler
3. **Training:** Random Forest trained when 10+ entries exist, saved as `.joblib`
4. **Prediction:** On-demand predictions via `/api/ml/predict/device/:id`
5. **Evaluation:** Confidence threshold (70%) filters low-quality predictions
6. **Feedback:** Users submit feedback to `prediction_feedback` collection
7. **Retraining:** Triggered by new data, improved by feedback

Located in `backend/ml_models.py`.

---

## SOFT SKILLS & PROJECT MANAGEMENT

### Q30: How did you organize and prioritize features?

**Answer:** Looking at commit history:
1. **Core Functionality First:** Authentication, device control, basic CRUD
2. **Value-Add Features:** Automation, ML predictions
3. **User Experience:** Analytics dashboard, error tracking
4. **Polish:** Gamification, streaks, advanced filtering, health monitoring

Clear progression from MVP to full-featured product.

### Q31: What would you improve if you had more time?

**Answer:**
- **Real-time Updates:** WebSockets for live device state changes
- **Mobile Push Notifications:** Alert users to automation events or errors
- **Scene Management:** Save and recall multi-device configurations
- **Energy Monitoring:** Track power consumption and costs
- **Advanced ML:** LSTM for time-series prediction, anomaly detection
- **User Permissions:** Multi-user homes with role-based access
- **API Versioning:** Proper `/api/v1/` versioning for backward compatibility
- **Performance Monitoring:** APM integration (New Relic, DataDog)
- **CI/CD:** Automated testing and deployment pipelines

---

## KEY TECHNICAL HIGHLIGHTS TO MEMORIZE

### Project Stats
- **Backend:** 1,757 lines in analytics_routes.py, 849 lines in device_routes.py
- **Frontend:** 110KB AnalyticsDashboardScreen.js
- **Database:** 7 main collections (users, devices, rooms, automations, device_logs, device_history, refresh_tokens)
- **ML:** Random Forest Classifier with 70% confidence threshold
- **Authentication:** Dual-token JWT (1-hour access, 30-day refresh)
- **Testing:** 72 test functions across 8 test files (1,154 lines of test code)
- **Pub/Sub:** Custom event bus + adaptive polling (500ms-30s intervals)

### Tech Stack Summary
- **Frontend:** React Native + Expo (~50.0.0)
- **Backend:** Flask + Flask-JWT-Extended + APScheduler
- **Database:** MongoDB (Atlas + Local)
- **ML:** scikit-learn (Random Forest)
- **Hardware:** Raspberry Pi 3 + Home Assistant
- **Languages:** Python (backend), JavaScript/JSX (frontend)

### Architecture Pattern
- Three-tier: Mobile App → REST API → Database
- Blueprint-based modular routing
- Context API for state management (pub/sub pattern)
- Repository pattern for database operations
- Adaptive polling for pseudo-real-time updates
- APScheduler for time-based event automation

### Standout Features
1. Machine learning usage prediction (Random Forest)
2. Comprehensive analytics (1,757 lines!)
3. Gamification with badges and streaks
4. Device health monitoring with error categorization
5. Timezone-aware analytics (UTC → Europe/Dublin)
6. Physical hardware integration (Raspberry Pi + Home Assistant)
7. Voice assistant integration
8. Custom event bus pub/sub architecture
9. Adaptive polling system (reduces server load)
10. Comprehensive testing suite (72 tests, 86% route coverage)

---

## BONUS: BEHAVIORAL QUESTIONS

### Q32: Tell me about a time you had to debug a complex issue.

**Answer:** When implementing the analytics dashboard, we discovered that date ranges were showing incorrect data. Users selecting "last 7 days" were seeing data from different periods. After investigation, I found the issue was timezone mismatch - the backend stored timestamps in UTC but didn't convert user-selected dates from Europe/Dublin timezone. I implemented a comprehensive timezone strategy: all storage in UTC, conversion to local timezone only at presentation layer, and timezone-aware query construction. This required updating multiple endpoints in the 1,757-line analytics_routes.py file. The fix ensured data accuracy across all analytics features.

### Q33: Describe a feature you're most proud of.

**Answer:** The device health monitoring system. I noticed users struggled to identify problematic devices, so I built an automatic health scoring system based on error rates from device_logs. Devices are categorized as Healthy (<10% errors), Warning (10-20%), or Critical (>20%). I added automatic error categorization (timeout, connection, permission), contextual troubleshooting suggestions, and quick action buttons (Retry, Refresh, Ping, Reset). This transformed error handling from reactive (waiting for user complaints) to proactive (identifying issues automatically). It demonstrates my ability to identify user pain points and create elegant solutions.

### Q34: How do you stay current with technology trends?

**Answer:** I actively practice by building projects like TechHome that incorporate modern technologies. For this project, I researched:
- React Native and Expo for cross-platform development
- JWT best practices for authentication
- Machine learning applications in IoT
- MongoDB schema design for time-series data
- Real-time scheduling with APScheduler

I also follow the evolution of the smart home ecosystem, including Home Assistant's development. The gamification features (badges, streaks) show I'm aware of user engagement trends. I continuously iterate on the project based on emerging best practices, as shown by recent commits adding advanced analytics and error tracking.

---

## INTERVIEW PREPARATION TIPS

1. **Know Your Numbers:**
   - 1,757 lines (analytics)
   - 849 lines (device management)
   - 230 lines (ML models)
   - 1,154 lines (test code)
   - 72 test functions across 8 test files
   - 10+ entries needed for ML training
   - 70% confidence threshold
   - 1-hour access tokens, 30-day refresh tokens
   - 500ms-30s adaptive polling intervals

2. **Be Ready to Discuss Trade-offs:**
   - Why MongoDB over SQL? (Flexible schema for varying device attributes)
   - Why Flask over Django? (Lightweight, API-focused)
   - Why React Native over native? (Cross-platform efficiency)
   - Why Random Forest over neural networks? (Interpretability, smaller dataset)
   - Why adaptive polling over WebSockets? (Simpler to implement, but planning WebSocket upgrade)
   - Why pytest over unittest? (Better fixtures, cleaner syntax, powerful plugins)

3. **Show Problem-Solving:**
   - Discuss specific challenges (timezone, user attribution, error detection)
   - Explain your thought process
   - Demonstrate continuous improvement (commit history)

4. **Highlight Unique Aspects:**
   - Physical hardware integration (Raspberry Pi)
   - ML prediction with feedback loop
   - Comprehensive analytics (not just basic stats)
   - Gamification and user engagement
   - **Pub/Sub architecture:** Custom event bus for reactive UI updates
   - **Adaptive polling:** Intelligent polling that reduces server load (500ms → 5s → 30s)
   - **Testing rigor:** 72 tests with 86% route coverage, fixtures, mocking, auto-cleanup

5. **Know What You'd Improve:**
   - Shows self-awareness
   - Demonstrates forward thinking
   - See Q31 for specific improvements (WebSockets, CI/CD, etc.)

6. **Master Common Interview Topics:**
   - **Pub/Sub (Q7):** Be ready to explain the custom event bus, adaptive polling, and future WebSocket plans
   - **Testing (Q23):** Discuss pytest, fixtures, mocking strategies, and the 86% route coverage (note: analytics routes need tests)
   - **Architecture:** Three-tier with Blueprint pattern, Context API, and APScheduler
   - **Scalability:** Can discuss database indexing, caching, load balancing (Q26)

Good luck with your interview!
