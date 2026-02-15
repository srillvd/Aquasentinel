# AquaSentinel - Design Document

## System Architecture

### High-Level Architecture
```
User (Mobile/Web)
    ↓
Frontend (Flutter/React)
    ↓
API Gateway
    ↓
Backend (FastAPI)
    ↓
┌─────────────┬──────────────┬─────────────┐
│   Image     │  ML Model    │  LLM        │
│  Processing │  Prediction  │  Recomm.    │
│  (OpenCV)   │  (Sklearn)   │  (Bedrock)  │
└─────────────┴──────────────┴─────────────┘
    ↓
Storage Layer (S3 + DynamoDB)
```

### Component Architecture

#### 1. Frontend Layer
- **Technology**: Flutter (mobile) or React (web)
- **Responsibilities**:
  - Image capture/upload
  - Environmental data form
  - Risk visualization
  - Recommendation display
- **Key Features**:
  - Offline data collection
  - Local language support
  - Simple, intuitive UI

#### 2. API Layer
- **Technology**: AWS API Gateway
- **Endpoints**:
  - `POST /api/scan` - Upload image and data
  - `GET /api/waterbody/{id}` - Fetch water body details
  - `GET /api/history/{id}` - Get scan history
  - `POST /api/feedback` - Submit action feedback

#### 3. Backend Processing Layer
- **Technology**: Python FastAPI + AWS Lambda
- **Components**:
  - Image processor service
  - Risk prediction service
  - Recommendation generator
  - Data management service

#### 4. Storage Layer
- **S3 Buckets**:
  - `aquasentinel-images` - Raw uploaded images
  - `aquasentinel-processed` - Processed image data
- **DynamoDB Tables**:
  - WaterBodies
  - ScanRecords
  - ActionLogs

---

## Module Design

### MODULE 1: Image-Based Algae Detection

#### Component: Image Processor

**Input**: 
- Raw image (JPG/PNG)
- Image metadata (location, timestamp)

**Processing Pipeline**:
```python
1. Image validation (size, format)
2. Resize to standard dimensions (640x480)
3. Convert RGB → HSV color space
4. Apply green color mask
   - Hue range: 35-85
   - Saturation: 40-255
   - Value: 40-255
5. Calculate metrics:
   - Green pixel ratio = (green_pixels / total_pixels) * 100
   - Green density clusters (contour detection)
   - Bloom intensity score
```

**Output**:
```json
{
  "green_ratio": 38.5,
  "density_score": 0.72,
  "cluster_count": 3,
  "processed_image_url": "s3://..."
}
```

**Algorithm Choice**:
- **MVP**: Rule-based HSV detection (OpenCV)
- **Advanced**: Lightweight CNN (MobileNet) for binary classification

---

### MODULE 2: Environmental Risk Input

#### Component: Data Collector

**Input Schema**:
```json
{
  "rainfall_mm": 120.5,
  "temperature_c": 32.0,
  "fertilizer_level": "high",
  "water_stagnation": true,
  "location": {
    "lat": 12.9716,
    "lon": 77.5946
  }
}
```

**Data Encoding**:
| Parameter | Type | Encoding |
|-----------|------|----------|
| Rainfall | Continuous | 0-500 mm |
| Temperature | Continuous | 15-45°C |
| Fertilizer | Categorical | Low=0, Medium=1, High=2 |
| Stagnation | Binary | No=0, Yes=1 |

**Validation Rules**:
- Rainfall: 0-500 mm (reject outliers)
- Temperature: 15-45°C
- Required fields: All parameters mandatory

---

### MODULE 3: Risk Prediction Model

#### Component: ML Predictor

**Model Architecture**:
- **Algorithm**: Random Forest Classifier
- **Framework**: Scikit-learn
- **Deployment**: AWS SageMaker endpoint (optional) or Lambda

**Feature Vector**:
```python
features = [
    green_ratio,        # 0-100
    density_score,      # 0-1
    rainfall_mm,        # 0-500
    temperature_c,      # 15-45
    fertilizer_encoded, # 0-2
    stagnation_flag     # 0-1
]
```

**Training Process**:
1. Collect labeled dataset (200-1000 samples)
2. Feature engineering and normalization
3. Train-test split (80-20)
4. Hyperparameter tuning (GridSearchCV)
5. Model evaluation (accuracy, precision, recall)
6. Export model (pickle/joblib)

**Prediction Output**:
```json
{
  "risk_probability": 0.78,
  "risk_level": "high",
  "confidence": 0.85,
  "contributing_factors": [
    "high_green_ratio",
    "heavy_rainfall",
    "high_fertilizer"
  ]
}
```

**Risk Classification Logic**:
```python
if probability < 0.40:
    risk_level = "low"
    color = "green"
elif probability < 0.70:
    risk_level = "medium"
    color = "yellow"
else:
    risk_level = "high"
    color = "red"
```

---

### MODULE 4: AI-Based Recommendation Engine

#### Component: LLM Recommendation Generator

**Technology**: AWS Bedrock (Claude/Titan)

**Prompt Template**:
```
Context:
- Water Body: Rural pond in {location}
- Risk Level: {risk_level}
- Green Density: {green_ratio}%
- Rainfall: {rainfall}mm (last 7 days)
- Fertilizer Usage: {fertilizer_level}
- Temperature: {temperature}°C

Task: Generate 3-5 practical preventive actions for rural farmers to prevent eutrophication. Keep recommendations simple, actionable, and culturally appropriate.

Output format:
1. [Action]
2. [Action]
3. [Action]
Timeline: [When to monitor next]
```

**Sample Output**:
```
Immediate Actions:
1. Install temporary barriers to reduce fertilizer runoff from nearby fields
2. Perform partial water flushing (20-30%) if possible
3. Introduce aeration using simple paddle wheels or manual stirring
4. Avoid adding any organic waste to the pond

Monitoring:
- Check water color daily for next 3 days
- Re-scan after 5 days
- Contact local agriculture officer if bloom appears
```

**Fallback Logic**:
If LLM unavailable, use rule-based recommendations:
```python
recommendations = {
    "high": [
        "Reduce fertilizer runoff immediately",
        "Increase water circulation",
        "Monitor daily for 3 days"
    ],
    "medium": [
        "Monitor fertilizer usage nearby",
        "Check water quality weekly"
    ],
    "low": [
        "Continue normal monitoring",
        "Re-scan in 2 weeks"
    ]
}
```

---

## Data Model Design

### DynamoDB Schema

#### Table 1: WaterBodies
```json
{
  "water_id": "WB_001",          // Partition Key
  "location": {
    "lat": 12.9716,
    "lon": 77.5946,
    "name": "Village Pond A"
  },
  "type": "pond",                // pond/lake/reservoir
  "size_sqm": 5000,
  "last_scan_date": "2026-02-14T10:30:00Z",
  "last_risk_level": "medium",
  "created_at": "2026-01-01T00:00:00Z"
}
```

#### Table 2: ScanRecords
```json
{
  "scan_id": "SCAN_12345",       // Partition Key
  "water_id": "WB_001",          // GSI
  "timestamp": "2026-02-14T10:30:00Z", // Sort Key
  "image_url": "s3://...",
  "green_ratio": 38.5,
  "density_score": 0.72,
  "rainfall_mm": 120.5,
  "temperature_c": 32.0,
  "fertilizer_level": 2,
  "stagnation": 1,
  "risk_probability": 0.78,
  "risk_level": "high",
  "confidence": 0.85
}
```

#### Table 3: ActionLogs
```json
{
  "action_id": "ACT_789",        // Partition Key
  "scan_id": "SCAN_12345",
  "water_id": "WB_001",
  "recommendations": [
    "Reduce fertilizer runoff",
    "Increase aeration"
  ],
  "followup_date": "2026-02-19",
  "status": "pending",           // pending/completed/skipped
  "user_feedback": null,
  "created_at": "2026-02-14T10:30:00Z"
}
```

---

## API Design

### Endpoint Specifications

#### 1. Submit Scan
```
POST /api/scan
Content-Type: multipart/form-data

Request:
{
  "image": <file>,
  "water_id": "WB_001",
  "rainfall_mm": 120.5,
  "temperature_c": 32.0,
  "fertilizer_level": "high",
  "water_stagnation": true
}

Response (200):
{
  "scan_id": "SCAN_12345",
  "risk_level": "high",
  "risk_probability": 0.78,
  "recommendations": [...],
  "next_scan_date": "2026-02-19"
}
```

#### 2. Get Water Body History
```
GET /api/waterbody/{water_id}/history?limit=10

Response (200):
{
  "water_id": "WB_001",
  "scans": [
    {
      "scan_id": "SCAN_12345",
      "timestamp": "2026-02-14T10:30:00Z",
      "risk_level": "high",
      "green_ratio": 38.5
    }
  ],
  "trend": "increasing"
}
```

---

## Processing Flow

### End-to-End Flow
```
1. User uploads image + environmental data
   ↓
2. API Gateway → Lambda (Image Processor)
   - Store image in S3
   - Extract green ratio using OpenCV
   ↓
3. Lambda (Risk Predictor)
   - Combine image features + environmental data
   - Call ML model endpoint
   - Generate risk score
   ↓
4. Lambda (Recommendation Generator)
   - Send context to LLM
   - Generate actionable recommendations
   ↓
5. Store results in DynamoDB
   ↓
6. Return response to user
   - Risk meter visualization
   - Recommendations in local language
```

### Timing Budget
- Image upload: 1-2s
- Image processing: 2-3s
- ML prediction: 1-2s
- LLM generation: 2-3s
- Total: < 10s

---

## UI/UX Design

### Mobile App Screens

#### Screen 1: Home Dashboard
- List of monitored water bodies
- Quick scan button
- Recent alerts

#### Screen 2: Scan Input
- Camera/gallery image picker
- Environmental data form (simple sliders/dropdowns)
- Submit button

#### Screen 3: Risk Result
- Large risk meter (circular gauge)
- Color-coded risk level
- Green ratio visualization
- "View Recommendations" button

#### Screen 4: Recommendations
- Numbered action list
- Timeline for follow-up
- "Mark as Done" checkboxes
- Share button (WhatsApp integration)

#### Screen 5: History
- Timeline view of past scans
- Trend graph (risk over time)
- Filter by water body

### Design Principles
- Mobile-first responsive design
- High contrast for outdoor visibility
- Large touch targets (min 44px)
- Minimal text input
- Visual feedback for all actions

---

## Technology Stack

### Frontend
- **Mobile**: Flutter 3.x
- **Web**: React 18 + TypeScript
- **State Management**: Provider/Redux
- **UI Library**: Material Design

### Backend
- **Framework**: FastAPI (Python 3.10+)
- **Image Processing**: OpenCV 4.x
- **ML**: Scikit-learn 1.3+
- **LLM**: AWS Bedrock (Claude 3)

### Infrastructure
- **Compute**: AWS Lambda
- **Storage**: S3, DynamoDB
- **API**: API Gateway
- **ML Hosting**: SageMaker (optional)
- **Monitoring**: CloudWatch

### Development Tools
- **Version Control**: Git
- **CI/CD**: GitHub Actions
- **Testing**: Pytest, Jest
- **Documentation**: Swagger/OpenAPI

---

## Deployment Architecture

### AWS Services Layout
```
┌─────────────────────────────────────┐
│         CloudFront (CDN)            │
└──────────────┬──────────────────────┘
               ↓
┌─────────────────────────────────────┐
│         API Gateway                 │
└──────────────┬──────────────────────┘
               ↓
┌──────────────────────────────────────┐
│  Lambda Functions                    │
│  - ImageProcessor                    │
│  - RiskPredictor                     │
│  - RecommendationGenerator           │
│  - DataManager                       │
└──────┬───────────────────────┬───────┘
       ↓                       ↓
┌─────────────┐         ┌─────────────┐
│     S3      │         │  DynamoDB   │
│   Images    │         │   Tables    │
└─────────────┘         └─────────────┘
```

### Environment Setup
- **Development**: Local + AWS Dev account
- **Staging**: AWS Staging environment
- **Production**: AWS Production (multi-region)

---

## Security Design

### Authentication & Authorization
- JWT-based authentication
- Role-based access control (RBAC)
- API key for mobile app

### Data Security
- Images encrypted at rest (S3 SSE)
- Data encrypted in transit (TLS 1.3)
- DynamoDB encryption enabled
- No PII storage without consent

### Privacy
- Location data anonymized for analytics
- User consent for data collection
- GDPR-compliant data retention (90 days)

---

## Testing Strategy

### Unit Testing
- Image processing functions
- ML model predictions
- API endpoints
- Data validation

### Integration Testing
- End-to-end scan flow
- Database operations
- External service calls (LLM)

### Performance Testing
- Load testing (100 concurrent users)
- Image processing latency
- API response times

### User Acceptance Testing
- Field testing with 5-10 rural users
- Usability feedback
- Accuracy validation

---

## Monitoring & Observability

### Metrics to Track
- API latency (p50, p95, p99)
- Error rates by endpoint
- ML model accuracy over time
- User engagement (scans per day)
- Storage usage

### Logging
- Structured JSON logs
- Request/response logging
- Error stack traces
- Audit logs for data access

### Alerts
- High error rate (> 5%)
- Slow response time (> 10s)
- Model accuracy drop (< 70%)
- Storage quota exceeded

---

## MVP Scope for Hackathon

### Must Have (Core Demo)
✅ Image green detection working
✅ Risk classification model trained
✅ Risk meter UI functional
✅ Basic recommendations generated
✅ Demo with 5-10 sample ponds

### Nice to Have (If Time Permits)
- Historical trend visualization
- Multi-language support
- WhatsApp integration
- Offline mode

### Out of Scope (Post-Hackathon)
- IoT sensor integration
- Satellite imagery
- Real-time monitoring
- Mobile app store deployment

---

## Innovation Highlights

### Technical Innovation
- Pre-bloom detection using color analysis
- Lightweight ML for edge deployment
- Context-aware LLM recommendations

### Social Innovation
- Rural-first design
- Local language support
- Practical, low-cost solution
- Community-driven monitoring

### Scalability Innovation
- Serverless architecture
- Pay-per-use model
- Cloud-native design
- Easy replication across regions
