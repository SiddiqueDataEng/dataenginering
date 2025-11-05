# Data Cleanup Ahead of Migration: Salesforce Data Preparation

*Published: January 2025 | By: Data Engineering Professional*

## Overview

Production-ready Python solution for normalizing and transforming 7 Excel files into 7 Salesforce-ready CSVs with advanced deduplication, fuzzy matching, data enrichment, QA metrics, and a one-command CLI. This project demonstrates enterprise data preparation patterns for migration scenarios.

## The Challenge

**Business Requirements:**
- Transform 7 Excel files into Salesforce-ready format
- Normalize data from multiple sources
- Implement deduplication and fuzzy matching
- Enrich data with additional information
- Provide QA metrics and validation
- One-command execution for automation

**Technical Challenges:**
- Multiple source formats and structures
- Data quality issues (duplicates, inconsistencies)
- Complex matching requirements
- Performance for large datasets
- Maintainable and extensible architecture

---

## Architecture Overview

### Data Pipeline Architecture

```
┌─────────────────────────────────────────────────────┐
│              Ingestion Layer                        │
│  • Excel File Reading                               │
│  • Schema Detection                                 │
│  • Data Type Inference                              │
└──────────────────┬──────────────────────────────────┘
                   │
┌──────────────────▼──────────────────────────────────┐
│              Staging Layer                          │
│  • Data Cleaning                                    │
│  • Standardization                                 │
│  • Validation                                       │
└──────────────────┬──────────────────────────────────┘
                   │
┌──────────────────▼──────────────────────────────────┐
│         Deduplication & Fuzzy Matching              │
│  • Duplicate Detection                             │
│  • Fuzzy Matching Algorithms                       │
│  • Record Linking                                  │
└──────────────────┬──────────────────────────────────┘
                   │
┌──────────────────▼──────────────────────────────────┐
│         Company Generation & Normalization           │
│  • Company Record Creation                         │
│  • Relationship Mapping                            │
│  • Data Normalization                              │
└──────────────────┬──────────────────────────────────┘
                   │
┌──────────────────▼──────────────────────────────────┐
│              Enrichment Layer                       │
│  • External Data Lookup                            │
│  • Data Augmentation                               │
│  • Business Rule Application                       │
└──────────────────┬──────────────────────────────────┘
                   │
┌──────────────────▼──────────────────────────────────┐
│              QA & Validation                        │
│  • Quality Metrics                                 │
│  • Validation Rules                                │
│  • Exception Reporting                             │
└──────────────────┬──────────────────────────────────┘
                   │
┌──────────────────▼──────────────────────────────────┐
│              Export Layer                           │
│  • Salesforce CSV Format                           │
│  • File Generation                                 │
│  • QA Summary Report                               │
└─────────────────────────────────────────────────────┘
```

### Project Structure

```
sf-cleanup/
├── config/
│   └── example.yml              # Configuration file
├── input/                        # Input Excel files
│   ├── file1.xlsx
│   ├── file2.xlsx
│   └── ...
├── output/                       # Generated CSVs
│   ├── output1.csv
│   ├── output2.csv
│   └── qa_summary.json
├── src/
│   ├── ingestion/
│   │   ├── excel_reader.py
│   │   └── schema_detector.py
│   ├── staging/
│   │   ├── cleaner.py
│   │   └── validator.py
│   ├── deduplication/
│   │   ├── fuzzy_matcher.py
│   │   └── record_linker.py
│   ├── normalization/
│   │   ├── company_generator.py
│   │   └── relationship_mapper.py
│   ├── enrichment/
│   │   ├── data_enricher.py
│   │   └── business_rules.py
│   ├── qa/
│   │   ├── metrics_calculator.py
│   │   └── validator.py
│   └── export/
│       ├── csv_generator.py
│       └── report_generator.py
├── tests/
│   ├── test_ingestion.py
│   ├── test_deduplication.py
│   └── ...
├── Dockerfile
├── pyproject.toml
└── README.md
```

---

## Core Components

### 1. Ingestion Layer

**Excel File Reading:**
```python
import pandas as pd
from pathlib import Path

def read_excel_file(file_path: Path, config: dict) -> pd.DataFrame:
    """Read Excel file with configuration-based parsing"""
    df = pd.read_excel(
        file_path,
        sheet_name=config.get('sheet_name', 0),
        header=config.get('header_row', 0),
        skiprows=config.get('skip_rows', 0)
    )
    return df
```

**Schema Detection:**
```python
def detect_schema(df: pd.DataFrame) -> dict:
    """Detect and infer schema from DataFrame"""
    schema = {
        'columns': list(df.columns),
        'dtypes': df.dtypes.to_dict(),
        'nullable': df.isnull().any().to_dict(),
        'sample_values': df.head(5).to_dict('records')
    }
    return schema
```

### 2. Staging & Data Cleaning

**Data Cleaning:**
```python
def clean_data(df: pd.DataFrame, rules: dict) -> pd.DataFrame:
    """Apply cleaning rules to DataFrame"""
    # Remove whitespace
    df = df.apply(lambda x: x.str.strip() if x.dtype == "object" else x)
    
    # Standardize formats
    if 'phone' in df.columns:
        df['phone'] = df['phone'].str.replace(r'[^\d]', '', regex=True)
    
    # Handle missing values
    df = df.fillna('')
    
    return df
```

**Validation:**
```python
def validate_data(df: pd.DataFrame, validators: list) -> dict:
    """Validate data against rules"""
    results = {
        'valid': True,
        'errors': [],
        'warnings': []
    }
    
    for validator in validators:
        result = validator(df)
        if not result['valid']:
            results['valid'] = False
            results['errors'].extend(result['errors'])
    
    return results
```

### 3. Deduplication & Fuzzy Matching

**Fuzzy Matching:**
```python
from fuzzywuzzy import fuzz
from fuzzywuzzy import process

def find_duplicates(df: pd.DataFrame, key_columns: list, threshold: int = 85) -> pd.DataFrame:
    """Find duplicate records using fuzzy matching"""
    duplicates = []
    processed = set()
    
    for idx, row in df.iterrows():
        if idx in processed:
            continue
        
        # Create comparison key
        key = ' '.join([str(row[col]) for col in key_columns])
        
        # Find similar records
        matches = process.extract(
            key,
            df[key_columns].apply(lambda x: ' '.join(x.astype(str)), axis=1),
            limit=10,
            scorer=fuzz.token_sort_ratio
        )
        
        # Group matches above threshold
        group = [idx]
        for match, score, match_idx in matches:
            if score >= threshold and match_idx not in processed:
                group.append(match_idx)
                processed.add(match_idx)
        
        if len(group) > 1:
            duplicates.append({
                'group_id': len(duplicates),
                'records': group,
                'confidence': max([score for _, score, _ in matches])
            })
        
        processed.add(idx)
    
    return pd.DataFrame(duplicates)
```

**Record Linking:**
```python
def link_records(df: pd.DataFrame, linking_rules: dict) -> pd.DataFrame:
    """Link related records based on business rules"""
    linked_df = df.copy()
    
    # Add relationship columns
    linked_df['parent_record_id'] = None
    linked_df['relationship_type'] = None
    
    for rule in linking_rules:
        # Apply linking logic
        condition = eval(rule['condition'])
        linked_df.loc[condition, 'parent_record_id'] = rule['parent_id']
        linked_df.loc[condition, 'relationship_type'] = rule['type']
    
    return linked_df
```

### 4. Company Generation & Normalization

**Company Record Creation:**
```python
def generate_company_records(df: pd.DataFrame) -> pd.DataFrame:
    """Generate company records from individual records"""
    companies = []
    
    # Group by company identifier
    grouped = df.groupby('company_name')
    
    for company_name, group in grouped:
        company_record = {
            'Name': company_name,
            'Type': 'Account',
            'Industry': group['industry'].mode()[0] if 'industry' in group.columns else '',
            'Phone': group['phone'].iloc[0] if 'phone' in group.columns else '',
            'Website': group['website'].mode()[0] if 'website' in group.columns else '',
            'Description': f"Auto-generated from {len(group)} records"
        }
        companies.append(company_record)
    
    return pd.DataFrame(companies)
```

**Data Normalization:**
```python
def normalize_data(df: pd.DataFrame, mapping: dict) -> pd.DataFrame:
    """Normalize data to Salesforce format"""
    normalized_df = pd.DataFrame()
    
    for sf_field, source_field in mapping.items():
        if source_field in df.columns:
            normalized_df[sf_field] = df[source_field]
        else:
            normalized_df[sf_field] = ''
    
    return normalized_df
```

### 5. Enrichment Layer

**Data Enrichment:**
```python
def enrich_data(df: pd.DataFrame, enrichment_sources: list) -> pd.DataFrame:
    """Enrich data with external sources"""
    enriched_df = df.copy()
    
    for source in enrichment_sources:
        if source['type'] == 'api':
            # Call external API
            enriched_df = call_enrichment_api(enriched_df, source)
        elif source['type'] == 'lookup':
            # Database lookup
            enriched_df = lookup_enrichment(enriched_df, source)
    
    return enriched_df
```

### 6. QA & Validation

**QA Metrics Calculation:**
```python
def calculate_qa_metrics(df: pd.DataFrame, original_df: pd.DataFrame) -> dict:
    """Calculate quality assurance metrics"""
    metrics = {
        'total_records': len(df),
        'original_records': len(original_df),
        'duplicates_removed': len(original_df) - len(df),
        'completeness_score': calculate_completeness(df),
        'accuracy_score': calculate_accuracy(df),
        'consistency_score': calculate_consistency(df),
        'valid_records': len(df[df['validation_status'] == 'valid']),
        'invalid_records': len(df[df['validation_status'] == 'invalid'])
    }
    
    return metrics
```

**Validation Rules:**
```python
def apply_validation_rules(df: pd.DataFrame, rules: list) -> pd.DataFrame:
    """Apply validation rules and mark records"""
    df['validation_status'] = 'valid'
    df['validation_errors'] = ''
    
    for rule in rules:
        condition = eval(rule['condition'])
        invalid_mask = ~condition
        
        df.loc[invalid_mask, 'validation_status'] = 'invalid'
        df.loc[invalid_mask, 'validation_errors'] += f"{rule['name']}; "
    
    return df
```

### 7. Export Layer

**CSV Generation:**
```python
def generate_salesforce_csv(df: pd.DataFrame, output_path: Path, config: dict):
    """Generate Salesforce-ready CSV"""
    # Filter valid records
    valid_df = df[df['validation_status'] == 'valid']
    
    # Select only required columns
    columns = config.get('salesforce_columns', df.columns.tolist())
    export_df = valid_df[columns]
    
    # Format for Salesforce
    export_df = format_for_salesforce(export_df, config)
    
    # Export to CSV
    export_df.to_csv(output_path, index=False, encoding='utf-8')
```

**QA Report Generation:**
```python
def generate_qa_report(metrics: dict, output_path: Path):
    """Generate QA summary report"""
    report = {
        'timestamp': datetime.now().isoformat(),
        'metrics': metrics,
        'summary': {
            'total_processed': metrics['total_records'],
            'success_rate': metrics['valid_records'] / metrics['total_records'] * 100,
            'quality_score': (
                metrics['completeness_score'] +
                metrics['accuracy_score'] +
                metrics['consistency_score']
            ) / 3
        }
    }
    
    with open(output_path, 'w') as f:
        json.dump(report, f, indent=2)
```

---

## CLI Interface

### Command-Line Interface

**Main CLI Implementation:**
```python
import click
from pathlib import Path
import yaml

@click.group()
def cli():
    """Salesforce Data Cleanup CLI"""
    pass

@cli.command()
@click.option('--config', required=True, type=click.Path(exists=True))
@click.option('--input', required=True, type=click.Path(exists=True))
@click.option('--output', required=True, type=click.Path())
def run(config, input, output):
    """Execute the full pipeline"""
    # Load configuration
    with open(config) as f:
        config_data = yaml.safe_load(f)
    
    # Run pipeline
    pipeline = DataCleanupPipeline(config_data)
    results = pipeline.run(Path(input), Path(output))
    
    click.echo(f"Processing complete: {results['total_records']} records processed")

@cli.command()
@click.option('--output', default='./config/example.yml')
def sample(output):
    """Generate sample config and folder structure"""
    # Generate sample configuration
    sample_config = {
        'input_files': {
            'file1': {'sheet_name': 0, 'header_row': 0},
            'file2': {'sheet_name': 0, 'header_row': 0}
        },
        'deduplication': {
            'enabled': True,
            'threshold': 85,
            'key_columns': ['name', 'email']
        },
        'enrichment': {
            'enabled': True,
            'sources': []
        }
    }
    
    with open(output, 'w') as f:
        yaml.dump(sample_config, f, default_flow_style=False)
    
    click.echo(f"Sample config created at {output}")

@cli.command()
@click.option('--output-dir', required=True, type=click.Path(exists=True))
def qa(output_dir):
    """Print QA metrics for a run"""
    qa_file = Path(output_dir) / 'qa_summary.json'
    
    if not qa_file.exists():
        click.echo("QA summary not found")
        return
    
    with open(qa_file) as f:
        metrics = json.load(f)
    
    click.echo("QA Metrics:")
    click.echo(f"Total Records: {metrics['metrics']['total_records']}")
    click.echo(f"Success Rate: {metrics['summary']['success_rate']:.2f}%")
    click.echo(f"Quality Score: {metrics['summary']['quality_score']:.2f}%")
```

---

## Configuration

### YAML Configuration Example

```yaml
input_files:
  file1:
    filename: "contacts.xlsx"
    sheet_name: 0
    header_row: 0
    skip_rows: 0
  file2:
    filename: "companies.xlsx"
    sheet_name: 0
    header_row: 0

deduplication:
  enabled: true
  threshold: 85
  key_columns:
    - name
    - email
    - phone
  algorithm: "fuzzy_token_sort"

fuzzy_matching:
  enabled: true
  methods:
    - token_sort_ratio
    - partial_ratio
  threshold: 80

normalization:
  column_mapping:
    FirstName: first_name
    LastName: last_name
    Email: email
    Phone: phone
    Company: company_name

enrichment:
  enabled: true
  sources:
    - type: "lookup"
      table: "company_directory"
      match_on: "company_name"
      fields: ["industry", "website", "employee_count"]

qa:
  validation_rules:
    - name: "email_format"
      condition: "df['email'].str.contains('@', na=False)"
    - name: "phone_format"
      condition: "df['phone'].str.len() >= 10"
  
  metrics:
    - completeness
    - accuracy
    - consistency

output:
  format: "csv"
  encoding: "utf-8"
  include_qa_report: true
```

---

## Key Features

### Advanced Deduplication

- **Fuzzy Matching**: Multiple algorithms (token_sort, partial_ratio, etc.)
- **Configurable Thresholds**: Adjustable similarity thresholds
- **Multi-column Matching**: Match on multiple columns simultaneously
- **Confidence Scores**: Match confidence scoring

### Data Enrichment

- **External Lookups**: Database and API lookups
- **Business Rules**: Configurable business logic
- **Data Augmentation**: Add calculated fields
- **Relationship Mapping**: Link related records

### Quality Assurance

- **Comprehensive Metrics**: Completeness, accuracy, consistency
- **Validation Rules**: Configurable validation
- **Exception Reporting**: Detailed error reporting
- **QA Summary**: JSON report with all metrics

### Automation

- **One-Command Execution**: Single CLI command
- **Configurable**: YAML-based configuration
- **Extensible**: Plugin-based architecture
- **Testable**: Comprehensive test suite

---

## Best Practices

### Data Preparation Best Practices

1. **Incremental Processing**: Process data in chunks for large files
2. **Validation Early**: Validate at each stage
3. **Error Handling**: Robust error handling and recovery
4. **Logging**: Comprehensive logging for debugging
5. **Testing**: Test with sample data before production

### Migration Best Practices

1. **Data Profiling**: Understand data before migration
2. **Clean Before Migrate**: Clean data before migration
3. **Incremental Migration**: Migrate in phases
4. **Validation**: Validate after each phase
5. **Rollback Plan**: Plan for rollback if needed

---

## Performance Optimization

### Large Dataset Handling

**Chunking:**
```python
def process_large_file(file_path: Path, chunk_size: int = 10000):
    """Process large files in chunks"""
    for chunk in pd.read_excel(file_path, chunksize=chunk_size):
        processed_chunk = process_chunk(chunk)
        yield processed_chunk
```

**Parallel Processing:**
```python
from multiprocessing import Pool

def parallel_process(files: list, num_workers: int = 4):
    """Process multiple files in parallel"""
    with Pool(num_workers) as pool:
        results = pool.map(process_file, files)
    return results
```

---

## Conclusion

Effective data cleanup before migration ensures:
- **High-quality data** in target system
- **Reduced migration issues** through pre-validation
- **Automated processing** reducing manual effort
- **Comprehensive QA** metrics for validation
- **Scalable solution** for enterprise needs

**Key Takeaways:**
1. Data cleanup is critical before migration
2. Fuzzy matching improves duplicate detection
3. QA metrics provide confidence in data quality
4. Automation reduces manual effort and errors
5. Configurable solutions adapt to different scenarios

---

*This solution demonstrates enterprise data preparation patterns. For related content, see [Enterprise Data Warehouse Migration](./enterprise-data-warehouse-migration.html) and [ETL Automation Solution](./etl-automation-solution.html).*

