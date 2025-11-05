# Athenahealth Data Generator

This project generates realistic healthcare data at scale, modeled after Athenahealth's EHR and practice management systems. The script can generate billions of rows of synthetic healthcare data for analytics, testing, and development purposes.

## Features

- Generates multiple related healthcare datasets:
  - Practice information
  - Clinical performance metrics
  - Financial performance metrics
  - Patient engagement metrics
  - AI utilization metrics
  - Patient records
  - Visit records

- Realistic data characteristics:
  - Seasonal variations in healthcare metrics
  - Yearly improvement trends
  - Practice size and specialty-specific variations
  - Realistic patient demographics and clinical values
  - AI adoption curves that follow S-curve patterns
  - Correlations between related metrics

## Requirements

```
pandas
numpy
faker
```

## Usage

1. Install dependencies:
```
pip install pandas numpy faker
```

2. Configure the script parameters in `generate_athenahealth_data.py`:
```python
# Configuration
OUTPUT_DIR = "generated_data"
CHUNK_SIZE = 1_000_000  # Number of records per file
TOTAL_RECORDS = 10_000_000  # Total number of records to generate (adjust as needed)
PRACTICE_COUNT = 1000  # Number of practices to generate
```

3. Run the script:
```
python generate_athenahealth_data.py
```

4. The generated data will be saved to the `generated_data` directory.

## Scaling to Billions of Rows

To generate billions of rows:

1. Increase the `TOTAL_RECORDS` parameter in the script
2. Consider running on a machine with sufficient RAM
3. For extremely large datasets, consider using a distributed processing framework like Dask or Spark

## Data Schema

The generated data follows these schemas:

### Practices
- practice_id: Unique identifier for the practice
- practice_name: Name of the practice
- practice_type: Type of practice (Independent, Health System, etc.)
- specialty: Medical specialty
- size: Size of practice (Small, Medium, Large)
- location: City and state
- athena_products: Athenahealth products used
- implementation_date: When the practice implemented Athenahealth

### Clinical Metrics
- practice_id: Practice identifier
- month, year, date: Time period
- patient_visits: Number of patient visits
- avg_visit_duration_min: Average visit duration in minutes
- charts_closed_same_day_pct: Percentage of charts closed same day
- quality_measures_met_pct: Percentage of quality measures met
- ai_documentation_usage: Level of AI documentation usage

### Financial Metrics
- practice_id: Practice identifier
- month, year, date: Time period
- total_charges: Total charges for the period
- collection_rate_pct: Collection rate percentage
- days_in_ar: Days in accounts receivable
- claim_denial_rate_pct: Claim denial rate percentage
- clean_claim_rate_pct: Clean claim rate percentage

### Patient Engagement Metrics
- practice_id: Practice identifier
- month, year, date: Time period
- portal_adoption_rate_pct: Patient portal adoption rate
- online_scheduling_pct: Online scheduling percentage
- digital_form_completion_pct: Digital form completion percentage
- patient_satisfaction_score: Patient satisfaction score (1-5)
- no_show_rate_pct: No-show rate percentage

### AI Utilization Metrics
- practice_id: Practice identifier
- month, year, date: Time period
- ambient_notes_usage_pct: Ambient notes usage percentage
- ai_assisted_coding_pct: AI-assisted coding percentage
- documentation_time_saved_min_per_visit: Documentation time saved (minutes per visit)
- ai_suggested_diagnoses_accepted_pct: Percentage of AI-suggested diagnoses accepted

### Patients
- patient_id: Unique patient identifier
- practice_id: Practice identifier
- age: Patient age
- gender: Patient gender
- primary_care_provider: Primary care provider name
- practice_name: Practice name
- insurance: Insurance provider
- medications: List of medications
- allergies: List of allergies
- registration_date: Date of registration
- last_visit_date: Date of last visit
- next_appointment_date: Date of next appointment

### Visits
- patient_id: Patient identifier
- visit_id: Unique visit identifier
- practice_id: Practice identifier
- visit_date: Date of visit
- provider: Provider name
- visit_type: Type of visit
- bp_systolic, bp_diastolic: Blood pressure readings
- weight_kg, height_cm, bmi: Physical measurements
- diagnoses: List of diagnoses
- procedures: List of procedures
- notes: Visit notes
- follow_up_needed: Whether follow-up is needed
- follow_up_timeframe: Timeframe for follow-up 