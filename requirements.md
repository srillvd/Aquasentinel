# AquaSentinel - Requirements Document

## Project Overview
AquaSentinel is a mobile/web system designed to predict eutrophication risk in rural water bodies using computer vision and machine learning. It provides early-warning alerts before algal blooms occur, enabling preventive action.

## Core Objective
Build a practical early-warning tool that:
- Analyzes pond/lake images for algae detection
- Accepts environmental parameters
- Predicts eutrophication risk
- Provides actionable preventive recommendations
- Targets rural Bharat communities

## Problem Statement
Eutrophication occurs when excess nitrogen/phosphorus from fertilizer runoff, combined with rainfall and high temperatures, leads to algal blooms. This reduces oxygen levels, kills fish, and makes water unsafe. Current systems are reactive; AquaSentinel is predictive.

## Functional Requirements

### FR1: Image-Based Algae Detection
- Accept pond/lake images via mobile/web upload
- Process images to detect green algae concentration
- Calculate green pixel ratio and density clusters
- Support image formats: JPG, PNG
- Handle varying lighting conditions

### FR2: Environmental Data Input
- Collect user inputs:
  - Rainfall (last 7 days in mm)
  - Temperature (°C)
  - Fertilizer usage nearby (Low/Medium/High)
  - Water stagnation status (Yes/No)
- Validate input ranges
- Store historical data for trend analysis

### FR3: Risk Prediction
- Process combined image and environmental data
- Generate risk probability (0-100%)
- Classify risk levels:
  - Low Risk: 0-40% (🟢)
  - Medium Risk: 41-70% (🟡)
  - High Risk: 71-100% (🔴)
- Provide confidence score

### FR4: AI-Based Recommendations
- Generate context-specific preventive actions
- Provide recommendations in local language
- Include timeline for follow-up monitoring
- Suggest practical interventions based on risk level

### FR5: Historical Tracking
- Store scan records with timestamps
- Track water body status over time
- Log recommended actions and follow-up status
- Enable trend visualization

### FR6: User Interface
- Simple mobile-first design
- Visual risk meter display
- Clear action items
- Support for Hindi and English
- Offline capability for data collection

## Non-Functional Requirements

### NFR1: Performance
- Image processing: < 5 seconds
- Risk prediction: < 2 seconds
- Total response time: < 10 seconds
- Support concurrent users: 100+

### NFR2: Scalability
- Handle 1000+ water bodies
- Store 10,000+ scan records
- Cloud-based architecture for growth

### NFR3: Accuracy
- Green detection accuracy: > 80%
- Risk prediction accuracy: > 75%
- False positive rate: < 20%

### NFR4: Usability
- Minimal training required
- Intuitive interface for rural users
- Clear visual indicators
- Simple data entry

### NFR5: Reliability
- System uptime: > 95%
- Data backup and recovery
- Graceful error handling
- Offline data collection with sync

## Technical Requirements

### TR1: Image Processing
- OpenCV for image analysis
- HSV color space conversion
- Green pixel detection algorithm
- Optional: CNN for advanced detection

### TR2: Machine Learning
- Random Forest or Logistic Regression
- Training dataset: 200-1000 labeled images
- Model retraining capability
- Feature engineering for environmental data

### TR3: Backend
- Python with FastAPI
- RESTful API design
- AWS Lambda for serverless processing
- S3 for image storage

### TR4: Database
- DynamoDB for NoSQL storage
- Schema for water bodies, scans, and actions
- Query optimization for historical data

### TR5: AI Integration
- LLM integration (AWS Bedrock)
- Structured prompt templates
- Context-aware recommendations

### TR6: Deployment
- Cloud-based (AWS)
- CI/CD pipeline
- Environment separation (dev/prod)
- Monitoring and logging

## Data Requirements

### Input Data
- Images: 200-1000 pond images (labeled)
- Environmental parameters: Historical weather and fertilizer data
- Location data: GPS coordinates of water bodies

### Output Data
- Risk scores with timestamps
- Recommendation text
- Historical trends
- Action logs

## User Roles

### Primary User (Rural Farmer/Community Worker)
- Upload images
- Input environmental data
- View risk assessment
- Receive recommendations

### Admin User
- Monitor system performance
- Review historical data
- Manage water body database
- Generate reports

## Constraints and Limitations

### Known Limitations
- Image lighting variations may affect accuracy
- Requires periodic model calibration
- Needs local validation for different water types
- Not lab-grade precision

### Scope Exclusions (MVP)
- No satellite data integration
- No IoT sensor deployment
- No real-time water chemistry testing
- No automated drone imaging

## Success Criteria

### MVP Success Metrics
- Functional image green detection
- Working risk classification model
- Operational risk meter UI
- Generated recommendations
- Demo with 5-10 sample ponds

### Long-term Success Metrics
- 500+ active users
- 1000+ water bodies monitored
- Documented prevention of algal blooms
- Community adoption rate > 60%

## Compliance and Security
- User data privacy protection
- Secure image storage
- GDPR-compliant data handling
- Role-based access control

## Innovation Angle
Unlike reactive systems that measure after oxygen drops or blooms appear, AquaSentinel detects:
- Early color shifts
- Environmental triggers
- Pre-bloom conditions

This predictive approach enables preventive action before damage occurs.
