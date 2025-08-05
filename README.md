# moonmama

# 🍼 Pregnancy Hub App – Technical Product Requirements Document (PRD)

## Executive Summary
A mobile-first Progressive Web App (PWA) for comprehensive pregnancy wellness tracking. Built on the PERN stack, initially serving a single user with architecture designed for future multi-tenant scaling. Core features include nutrition tracking with pregnancy-specific micronutrients, fitness logging, symptom tracking, appointment management, and daily affirmations.

---

## 📋 Table of Contents
1. [Project Overview](#project-overview)
2. [Technical Architecture](#technical-architecture)
3. [Database Design](#database-design)
4. [Feature Specifications](#feature-specifications)
5. [API Design](#api-design)
6. [Security & Compliance](#security-compliance)
7. [Development Roadmap](#development-roadmap)
8. [Testing Strategy](#testing-strategy)
9. [Deployment & Infrastructure](#deployment-infrastructure)

---

## 🎯 Project Overview

### Goals
- **Primary**: Build a fully functional MVP for single-user pregnancy tracking
- **Secondary**: Design architecture to support future multi-user scaling
- **Tertiary**: Prepare foundation for HIPAA compliance

### Success Criteria
- Daily active usage for 3+ consecutive weeks
- All core features functional on mobile devices
- Page load time < 3 seconds on 3G
- Offline capability for core features

### Technical Constraints
- Must work as web app (no React Native for MVP)
- Apple HealthKit integration limited by browser capabilities
- Must support real-time data entry throughout the day

---

## 🏗️ Technical Architecture

### Frontend Stack
```
React 18+ with Vite
├── UI Framework: Tailwind CSS (mobile-first utilities)
├── State Management: Zustand
├── Routing: React Router v6
├── Data Fetching: TanStack Query (React Query)
├── Forms: React Hook Form + Zod validation
├── Charts: Recharts or Victory
├── PWA: Workbox for service workers
└── Camera/Barcode: react-qr-barcode-scanner
```

### Backend Stack
```
Node.js 18+ with Express.js
├── Database: PostgreSQL 14+
├── ORM: Prisma (type-safe, migration-friendly)
├── Authentication: JWT (future-ready)
├── File Storage: AWS S3
├── Caching: Redis (optional for MVP)
├── API Documentation: OpenAPI/Swagger
└── Task Queue: Bull (for calendar sync)
```

### Third-Party Services
- **AWS S3**: Photo storage
- **Google Calendar API**: Appointment sync
- **Open Food Facts API**: Nutrition data
- **USDA API**: Backup nutrition source
- **SendGrid**: Future email notifications

### Architecture Diagram
```
┌─────────────────┐     ┌─────────────────┐
│   React PWA     │────▶│   Express API   │
│  (Mobile-First) │     │   (REST/JSON)   │
└─────────────────┘     └────────┬────────┘
                                 │
                    ┌────────────┼────────────┐
                    │            │            │
              ┌─────▼─────┐ ┌───▼────┐ ┌────▼────┐
              │PostgreSQL │ │  AWS   │ │External │
              │    RDS    │ │   S3   │ │  APIs   │
              └───────────┘ └────────┘ └─────────┘
```

---

## 💾 Database Design

### Core Schema

```sql
-- Users (future multi-tenant ready)
CREATE TABLE users (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  email VARCHAR(255) UNIQUE NOT NULL,
  password_hash VARCHAR(255),
  tenant_id UUID DEFAULT gen_random_uuid(),
  created_at TIMESTAMP DEFAULT NOW(),
  updated_at TIMESTAMP DEFAULT NOW()
);

-- Pregnancies
CREATE TABLE pregnancies (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  user_id UUID REFERENCES users(id),
  due_date DATE NOT NULL,
  start_date DATE NOT NULL,
  baby_name VARCHAR(100),
  is_active BOOLEAN DEFAULT true,
  created_at TIMESTAMP DEFAULT NOW()
);

-- Nutrition Logs
CREATE TABLE nutrition_logs (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  pregnancy_id UUID REFERENCES pregnancies(id),
  logged_at TIMESTAMP DEFAULT NOW(),
  meal_type VARCHAR(20), -- breakfast, lunch, dinner, snack
  food_name VARCHAR(255) NOT NULL,
  barcode VARCHAR(50),
  quantity DECIMAL(10,2),
  unit VARCHAR(20),
  calories INTEGER,
  macros JSONB, -- {protein, carbs, fat, fiber}
  micronutrients JSONB -- {folate, iron, calcium, etc}
);

-- Symptom Logs
CREATE TABLE symptom_logs (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  pregnancy_id UUID REFERENCES pregnancies(id),
  date DATE NOT NULL,
  symptoms JSONB, -- [{name, severity, time_of_day}]
  mood VARCHAR(50),
  energy_level INTEGER CHECK (energy_level BETWEEN 1 AND 10),
  notes TEXT,
  updated_at TIMESTAMP DEFAULT NOW()
);

-- Fitness Logs
CREATE TABLE fitness_logs (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  pregnancy_id UUID REFERENCES pregnancies(id),
  logged_at TIMESTAMP DEFAULT NOW(),
  activity_type VARCHAR(50),
  duration_minutes INTEGER,
  intensity VARCHAR(20),
  heart_rate_avg INTEGER,
  steps INTEGER,
  notes TEXT
);

-- Water/Hydration Logs
CREATE TABLE hydration_logs (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  pregnancy_id UUID REFERENCES pregnancies(id),
  date DATE NOT NULL,
  water_oz INTEGER DEFAULT 0,
  electrolyte_servings INTEGER DEFAULT 0
);

-- Sleep Logs
CREATE TABLE sleep_logs (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  pregnancy_id UUID REFERENCES pregnancies(id),
  date DATE NOT NULL,
  night_sleep_hours DECIMAL(3,1),
  night_sleep_quality INTEGER CHECK (night_sleep_quality BETWEEN 1 AND 5),
  naps JSONB -- [{start_time, duration_minutes}]
);

-- Photos
CREATE TABLE photos (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  pregnancy_id UUID REFERENCES pregnancies(id),
  s3_key VARCHAR(255) NOT NULL,
  caption TEXT,
  week_number INTEGER,
  photo_type VARCHAR(50), -- bump, ultrasound, other
  taken_at TIMESTAMP DEFAULT NOW()
);

-- Appointments
CREATE TABLE appointments (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  pregnancy_id UUID REFERENCES pregnancies(id),
  title VARCHAR(255) NOT NULL,
  provider_name VARCHAR(255),
  location TEXT,
  scheduled_at TIMESTAMP NOT NULL,
  google_event_id VARCHAR(255),
  notes TEXT,
  test_results JSONB -- {weight, blood_pressure, fundal_height, etc}
);

-- Affirmations
CREATE TABLE affirmations (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  text TEXT NOT NULL,
  week_range_start INTEGER,
  week_range_end INTEGER,
  category VARCHAR(50)
);

-- User Affirmation Interactions
CREATE TABLE user_affirmations (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  pregnancy_id UUID REFERENCES pregnancies(id),
  affirmation_id UUID REFERENCES affirmations(id),
  shown_at TIMESTAMP DEFAULT NOW(),
  is_favorited BOOLEAN DEFAULT false
);

-- Nutrient Goals
CREATE TABLE nutrient_goals (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  pregnancy_id UUID REFERENCES pregnancies(id),
  nutrient_name VARCHAR(50) NOT NULL,
  daily_goal_amount DECIMAL(10,2),
  unit VARCHAR(20),
  week_number INTEGER,
  is_custom BOOLEAN DEFAULT false
);

-- Unsafe Ingredients Reference
CREATE TABLE unsafe_ingredients (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  name VARCHAR(255) NOT NULL,
  category VARCHAR(100),
  risk_level VARCHAR(20), -- avoid, limit, caution
  reason TEXT
);
```

### Indexes
```sql
CREATE INDEX idx_nutrition_logs_pregnancy_date ON nutrition_logs(pregnancy_id, logged_at);
CREATE INDEX idx_symptom_logs_pregnancy_date ON symptom_logs(pregnancy_id, date);
CREATE INDEX idx_appointments_pregnancy_date ON appointments(pregnancy_id, scheduled_at);
CREATE INDEX idx_photos_pregnancy_week ON photos(pregnancy_id, week_number);
```

---

## 📱 Feature Specifications

### 1. Daily Dashboard
**Route**: `/`

**Components**:
- Greeting with current week of pregnancy
- Today's progress rings (nutrition, hydration, fitness)
- Quick action buttons (floating action button style)
- Upcoming appointment reminder
- Daily affirmation card
- Recent activity feed

**Technical Requirements**:
- Cache dashboard data in localStorage
- Refresh on pull-down gesture
- Lazy load activity feed

### 2. Nutrition Tracking
**Route**: `/nutrition`

**Features**:
- **Barcode Scanner**: 
  - Uses device camera via `getUserMedia()`
  - Falls back to manual barcode entry
  - Caches scanned items locally

- **Manual Entry**:
  - Autocomplete from previous entries
  - Quick-add common foods
  - Custom food creation

- **Micronutrient Tracking** (13 tracked nutrients):
  - Folate, Choline, Iron, Vitamin D3, DHA
  - Vitamin A, Iodine, Vitamin B12, Magnesium
  - Zinc, Calcium, Selenium, Copper

- **Visual Progress**:
  - Circular progress rings (Apple Fitness style)
  - Daily, weekly, monthly views
  - Nutrient trend charts

**API Integrations**:
```javascript
// Open Food Facts API
GET https://world.openfoodfacts.org/api/v0/product/{barcode}.json

// USDA FoodData Central (backup)
GET https://api.nal.usda.gov/fdc/v1/foods/search
```

### 3. Fitness Tracking
**Route**: `/fitness`

**Features**:
- Activity logging (type, duration, intensity)
- Manual step count entry
- Heart rate tracking
- Prenatal-safe exercise library
- Weekly activity summary

**Apple Health Integration Workaround**:
```javascript
// Option 1: Manual CSV import
const handleHealthDataImport = (csvFile) => {
  // Parse Apple Health export CSV
  // Map to our fitness_logs schema
};

// Option 2: iOS Shortcuts webhook
POST /api/health-import
{
  "steps": 5000,
  "heartRate": 75,
  "source": "apple-health-shortcut"
}
```

### 4. Symptom & Mood Tracking
**Route**: `/journal`

**Features**:
- Common symptom checklist:
  - Nausea, fatigue, headaches, cramping
  - Braxton Hicks, swelling, back pain
  - Custom symptom addition
- Mood selector with emojis
- Energy level slider (1-10)
- Free-form notes with rich text
- Photo attachment for entries

### 5. Sleep Tracking
**Route**: `/sleep`

**Features**:
- Night sleep duration picker
- Sleep quality rating
- Nap logging with timestamps
- Weekly sleep pattern visualization
- Sleep debt calculator

### 6. Hydration Tracking
**Route**: `/hydration`

**Features**:
- Tap to add 8oz increments
- Custom amount input
- Electrolyte serving counter
- Daily goal progress bar
- Reminder notifications

### 7. Medical & Appointments
**Route**: `/medical`

**Features**:
- Appointment CRUD with Google Calendar sync
- Test result logging (weight, BP, measurements)
- Document upload (ultrasounds, lab results)
- Provider contact information
- Appointment history timeline

**Google Calendar Integration**:
```javascript
// OAuth2 flow
const SCOPES = ['https://www.googleapis.com/auth/calendar'];

// Create event
const createAppointment = async (appointmentData) => {
  const event = {
    summary: appointmentData.title,
    location: appointmentData.location,
    start: { dateTime: appointmentData.scheduledAt },
    end: { dateTime: appointmentData.endTime },
    description: appointmentData.notes
  };
  
  await calendar.events.insert({
    calendarId: 'primary',
    resource: event
  });
};
```

### 8. Progress Photos
**Route**: `/photos`

**Features**:
- Camera capture with grid overlay
- Weekly organization
- Side-by-side comparisons
- Private/secure S3 storage
- Caption and milestone notes

**Implementation**:
```javascript
// Progressive camera capture
<input 
  type="file" 
  accept="image/*" 
  capture="camera"
  onChange={handlePhotoCapture}
/>

// S3 upload with presigned URL
const uploadPhoto = async (file) => {
  const { presignedUrl } = await api.getPresignedUrl();
  await fetch(presignedUrl, {
    method: 'PUT',
    body: file,
    headers: { 'Content-Type': file.type }
  });
};
```

### 9. Daily Affirmations
**Route**: `/affirmations`

**Features**:
- Week-appropriate affirmations
- Save/favorite functionality
- Share capability
- Affirmation history
- Categories: body positivity, preparation, mindfulness

### 10. Unsafe Ingredient Checker
**Route**: `/ingredient-check`

**Features**:
- Search ingredient database
- Scan product ingredients
- Risk level indicators
- Alternative suggestions
- Source citations

---

## 🔌 API Design

### RESTful Endpoints

```typescript
// Base URL: https://api.pregnancyhub.app/v1

// Authentication (future)
POST   /auth/login
POST   /auth/logout
POST   /auth/refresh

// Dashboard
GET    /dashboard/summary?date={date}

// Nutrition
GET    /nutrition/logs?date={date}
POST   /nutrition/logs
PUT    /nutrition/logs/{id}
DELETE /nutrition/logs/{id}
GET    /nutrition/search?q={query}
GET    /nutrition/barcode/{barcode}
GET    /nutrition/goals/week/{weekNumber}
PUT    /nutrition/goals

// Fitness
GET    /fitness/logs?startDate={date}&endDate={date}
POST   /fitness/logs
PUT    /fitness/logs/{id}
DELETE /fitness/logs/{id}

// Symptoms
GET    /symptoms/logs?date={date}
POST   /symptoms/logs
PUT    /symptoms/logs/{id}
GET    /symptoms/common

// Sleep
GET    /sleep/logs?date={date}
POST   /sleep/logs
PUT    /sleep/logs/{id}

// Hydration
GET    /hydration/logs?date={date}
POST   /hydration/logs
PUT    /hydration/logs/{id}

// Photos
GET    /photos?week={weekNumber}
POST   /photos/upload-url
POST   /photos
DELETE /photos/{id}

// Appointments
GET    /appointments?startDate={date}&endDate={date}
POST   /appointments
PUT    /appointments/{id}
DELETE /appointments/{id}
POST   /appointments/{id}/sync-calendar

// Affirmations
GET    /affirmations/daily
GET    /affirmations/favorites
POST   /affirmations/{id}/favorite

// Ingredient Safety
GET    /ingredients/check?name={ingredient}
GET    /ingredients/unsafe
```

### Response Format
```json
{
  "success": true,
  "data": {},
  "error": null,
  "metadata": {
    "timestamp": "2024-01-01T00:00:00Z",
    "version": "1.0"
  }
}
```

---

## 🔒 Security & Compliance

### Security Measures
1. **Authentication**:
   - JWT tokens with refresh rotation
   - Secure HttpOnly cookies
   - Rate limiting on all endpoints

2. **Data Protection**:
   - TLS 1.3 for all connections
   - Encryption at rest (AWS RDS)
   - S3 bucket policies for photos

3. **HIPAA Compliance Preparation**:
   - Audit logging for all data access
   - Data retention policies
   - User consent tracking
   - Right to deletion implementation

### Privacy Considerations
```javascript
// Example: Photo upload with privacy
const uploadSecurePhoto = async (file, metadata) => {
  // 1. Get presigned URL with expiration
  const { url, key } = await api.getPresignedUploadUrl({
    contentType: file.type,
    metadata: {
      pregnancyWeek: metadata.week,
      photoType: 'bump'
    }
  });
  
  // 2. Client-side encryption option
  const encryptedFile = await encryptFile(file);
  
  // 3. Upload with server-side encryption
  await fetch(url, {
    method: 'PUT',
    body: encryptedFile,
    headers: {
      'x-amz-server-side-encryption': 'AES256'
    }
  });
};
```

---

## 📅 Development Roadmap

### Phase 0: Foundation (Week 1-2)
- [ ] Project setup (React, Express, PostgreSQL)
- [ ] Database schema creation and migrations
- [ ] Basic routing and navigation
- [ ] PWA configuration
- [ ] S3 bucket setup

### Phase 1: Core Features (Week 3-4)
- [ ] Nutrition logging (manual + barcode)
- [ ] Micronutrient tracking UI
- [ ] Quick logging widgets (water, sleep)
- [ ] Basic symptom tracking
- [ ] Daily dashboard

### Phase 2: Advanced Features (Week 5-6)
- [ ] Google Calendar integration
- [ ] Photo upload and gallery
- [ ] Fitness tracking
- [ ] Appointment management
- [ ] Unsafe ingredient checker

### Phase 3: Polish & Data Viz (Week 7)
- [ ] Progress visualizations
- [ ] Daily affirmations
- [ ] Offline functionality
- [ ] Performance optimization
- [ ] UI/UX refinements

### Phase 4: Testing & Deployment (Week 8)
- [ ] Comprehensive testing
- [ ] AWS deployment
- [ ] Monitoring setup
- [ ] Documentation
- [ ] Beta testing with real data

---

## 🧪 Testing Strategy

### Testing Approach
```javascript
// Unit Tests (Jest + React Testing Library)
- Component rendering
- Form validations
- API endpoint logic
- Utility functions

// Integration Tests (Cypress)
- User flows (meal logging, appointment booking)
- API integration
- Calendar sync
- Photo upload

// Performance Tests
- Lighthouse scores > 90
- Time to Interactive < 3s
- First Contentful Paint < 1.5s
```

### Test Data
- Real pregnancy data for accurate testing
- Anonymized dataset for edge cases
- Automated daily user simulation

---

## 🚀 Deployment & Infrastructure

### AWS Infrastructure
```yaml
Production Environment:
  - EC2: t3.medium for API server
  - RDS: PostgreSQL db.t3.micro
  - S3: pregnancy-hub-photos bucket
  - CloudFront: CDN for static assets
  - Route53: DNS management
  - ACM: SSL certificates

Monitoring:
  - CloudWatch: Logs and metrics
  - Sentry: Error tracking
  - Datadog: APM (future)
```

### Deployment Pipeline
```bash
# GitHub Actions workflow
- Build React app
- Run tests
- Build Docker image
- Push to ECR
- Deploy to ECS
- Run migrations
- Invalidate CloudFront cache
```

### Environment Variables
```env
# .env.production
DATABASE_URL=postgresql://...
AWS_BUCKET_NAME=pregnancy-hub-photos
GOOGLE_CLIENT_ID=...
GOOGLE_CLIENT_SECRET=...
JWT_SECRET=...
OPEN_FOOD_FACTS_API_KEY=...
```

---

## 📊 Monitoring & Analytics

### Key Metrics
- Daily Active Users (DAU)
- Feature adoption rates
- API response times
- Error rates
- Photo upload success rate

### Analytics Events
```javascript
// Example tracking
analytics.track('MealLogged', {
  mealType: 'breakfast',
  hasBarcode: true,
  nutrientCompleteness: 0.87
});
```

---

## 🔄 Future Enhancements

### Post-MVP Roadmap
1. **Multi-user support** with family sharing
2. **AI-powered meal suggestions** based on nutrient gaps
3. **Wearable integration** (Apple Watch, Fitbit)
4. **Telehealth integration** for virtual appointments
5. **Community features** with privacy controls
6. **Postpartum transition** tracking
7. **Partner app** for support persons
8. **Recipe database** with pregnancy-safe meals

---

## 📝 Appendix

### Micronutrient Daily Goals by Trimester
| Nutrient | First Trimester | Second Trimester | Third Trimester | Unit |
|----------|----------------|------------------|-----------------|------|
| Folate | 600 | 600 | 600 | mcg |
| Choline | 450 | 450 | 450 | mg |
| Iron | 27 | 27 | 27 | mg |
| Vitamin D3 | 600 | 600 | 600 | IU |
| DHA | 200 | 200 | 300 | mg |
| Calcium | 1000 | 1000 | 1000 | mg |
| Iodine | 220 | 220 | 220 | mcg |
| Vitamin B12 | 2.6 | 2.6 | 2.6 | mcg |
| Magnesium | 350 | 360 | 360 | mg |
| Zinc | 11 | 11 | 11 | mg |
| Selenium | 60 | 60 | 60 | mcg |
| Copper | 1000 | 1000 | 1000 | mcg |
| Vitamin A | 770 | 770 | 770 | mcg RAE |

### PWA Manifest Example
```json
{
  "name": "Pregnancy Hub",
  "short_name": "PregHub",
  "description": "Your personal pregnancy wellness tracker",
  "start_url": "/",
  "display": "standalone",
  "theme_color": "#FEC5E5",
  "background_color": "#FFF0F5",
  "icons": [
    {
      "src": "/icon-192.png",
      "sizes": "192x192",
      "type": "image/png"
    },
    {
      "src": "/icon-512.png",
      "sizes": "512x512",
      "type": "image/png"
    }
  ]
}
```
