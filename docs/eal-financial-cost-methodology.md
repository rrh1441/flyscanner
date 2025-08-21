# EAL (Expected Annual Loss) and Financial Cost Methodology

## Executive Summary

The Flyscanner platform implements a comprehensive Expected Annual Loss (EAL) calculation system that automatically assigns financial risk values to security findings. This methodology provides quantified risk assessments for cybersecurity, legal compliance, and cloud infrastructure vulnerabilities, enabling data-driven risk management decisions.

## Overview

The EAL system converts qualitative security findings into quantitative financial impact estimates using a multi-factor calculation approach that considers:

- **Severity levels** (CRITICAL, HIGH, MEDIUM, LOW, INFO)
- **Finding types** (e.g., VERIFIED_CVE, DATA_BREACH_EXPOSURE, ADA_LEGAL_CONTINGENT_LIABILITY)
- **Attack categories** (CYBER, LEGAL, CLOUD)
- **Confidence intervals** (Low, Most Likely, High estimates)
- **Time horizons** (Daily, Annual exposure)

## Architecture and Components

### Database Schema

The EAL system is built on five core database tables:

#### 1. attack_meta
Defines attack categories and base financial impacts:
```sql
- attack_type_code (Primary Key): PHISHING_BEC, SITE_HACK, MALWARE, etc.
- prevalence: Likelihood factor (0-1 scale)
- raw_weight: Base financial impact in dollars
- category: CYBER, LEGAL, or CLOUD
```

#### 2. finding_type_mapping
Maps specific finding types to attack categories:
```sql
- finding_type: Specific security finding (e.g., VERIFIED_CVE)
- attack_type_code: Associated attack category (e.g., SITE_HACK)
- custom_multiplier: Type-specific adjustment factor
- severity_override: Optional severity adjustment
```

#### 3. severity_weight
Severity-based multipliers for risk calculation:
```sql
- CRITICAL: 5.0x multiplier
- HIGH: 2.5x multiplier  
- MEDIUM: 1.0x multiplier
- LOW: 0.3x multiplier
- INFO: 0.0x multiplier
```

#### 4. risk_constants
Configurable system parameters:
```sql
- LOW_CONFIDENCE_CONSTANT: Conservative estimate factor
- ML_CONFIDENCE_CONSTANT: Most likely estimate factor
- HIGH_CONFIDENCE_CONSTANT: Worst-case estimate factor
- ADA_SETTLEMENT_LOW/ML/HIGH: Fixed legal liability amounts
```

#### 5. dow_cost_constants
Denial of Wallet service cost parameters:
```sql
- service_type: Cloud service identifier
- cost_per_request: Base cost per API call
- typical_rps: Expected requests per second
- amplification_factor: Attack amplification multiplier
```

## Calculation Methodology

### Core EAL Formula

```
Base Impact = raw_weight × severity_multiplier × custom_multiplier × prevalence

EAL Low = Base Impact × severity_low_confidence × LOW_CONFIDENCE_CONSTANT
EAL ML = Base Impact × severity_ml_confidence × ML_CONFIDENCE_CONSTANT  
EAL High = Base Impact × severity_high_confidence × HIGH_CONFIDENCE_CONSTANT
```

### Base Values by Severity

| Severity | EAL Low | EAL Most Likely | EAL High | EAL Daily |
|----------|---------|----------------|----------|-----------|
| CRITICAL | $50,000 | $250,000 | $1,000,000 | $10,000 |
| HIGH | $10,000 | $50,000 | $250,000 | $2,500 |
| MEDIUM | $2,500 | $10,000 | $50,000 | $500 |
| LOW | $500 | $2,500 | $10,000 | $100 |
| INFO | $0 | $0 | $0 | $0 |

### Finding Type Multipliers

#### Critical Financial Impact (10x daily cost)
- **DENIAL_OF_WALLET**: Cloud cost amplification attacks
- **CLOUD_COST_AMPLIFICATION**: Resource exhaustion attacks

#### Legal/Compliance Risks
- **ADA_LEGAL_CONTINGENT_LIABILITY**: Fixed $25k-$500k liability
- **GDPR_VIOLATION**: 3-10x multiplier for data privacy violations
- **PCI_COMPLIANCE_FAILURE**: 2-8x multiplier for payment card security

#### Data Exposure (High risk multipliers)
- **EXPOSED_DATABASE**: 4-15x multiplier
- **DATA_BREACH_EXPOSURE**: 3-10x multiplier  
- **CLIENT_SIDE_SECRET_EXPOSURE**: 2-5x multiplier
- **SENSITIVE_FILE_EXPOSURE**: 2-8x multiplier

#### Verified Vulnerabilities
- **VERIFIED_CVE**: 2-8x multiplier for confirmed CVEs
- **VULNERABILITY**: 1.5-5x multiplier for potential vulnerabilities

#### Brand/Reputation Damage
- **MALICIOUS_TYPOSQUAT**: 1.5-6x multiplier
- **PHISHING_INFRASTRUCTURE**: 2-8x multiplier
- **ADVERSE_MEDIA**: 1-5x multiplier

#### Infrastructure/Operational
- **EXPOSED_SERVICE**: 1.5-5x multiplier
- **MISSING_RATE_LIMITING**: 1-4x multiplier
- **TLS_CONFIGURATION_ISSUE**: 0.8-3x multiplier
- **EMAIL_SECURITY_GAP**: 1-4x multiplier

## Attack Categories and Aggregation

### CYBER (Aggregated as cyber_total)
- **PHISHING_BEC**: Business email compromise ($300k base impact)
- **SITE_HACK**: Website vulnerabilities ($500k base impact)
- **MALWARE**: Malware infections ($400k base impact)
- **CLIENT_SIDE_SECRET_EXPOSURE**: Exposed secrets ($600k base impact)

### LEGAL (Separate line items)
- **ADA_COMPLIANCE**: Fixed $25k-$500k liability
- **GDPR_VIOLATION**: GDPR fines ($500k base impact)
- **PCI_COMPLIANCE_FAILURE**: PCI violations ($250k base impact)

### CLOUD (Daily cost calculations)
- **DENIAL_OF_WALLET**: Cloud cost attacks (calculated daily)

## Special Case Calculations

### ADA Compliance
Fixed legal liability amounts regardless of severity:
```
- EAL Low: $25,000 (minimum settlement)
- EAL ML: $75,000 (average settlement)  
- EAL High: $500,000 (major lawsuit)
- EAL Daily: $0 (not a recurring cost)
```

### Denial of Wallet
Dynamic calculation based on extracted costs:
```
If finding description contains "Estimated daily cost: $X":
- EAL Daily = Extracted amount
- EAL Low = Daily × 30 (1 month exposure)
- EAL ML = Daily × 90 (3 month exposure)
- EAL High = Daily × 365 (1 year exposure)
```

## Implementation Details

### Automatic Calculation
The system uses database triggers to automatically calculate EAL values:
```sql
-- Trigger function: calculate_finding_eal()
-- Triggers: calculate_eal_on_insert, calculate_eal_on_update
```

### Manual Calculation
Backup calculation via Edge Function:
```bash
# Trigger EAL calculation for specific scan
node scripts/trigger-eal-calculation.js <scan_id>

# Debug EAL calculation
node scripts/query-findings-eal.js <scan_id>
```

### Integration with Sync Worker
The sync worker aggregates EAL values by attack_type_code:
```sql
SELECT attack_type_code, 
       SUM(eal_low) as total_eal_low,
       SUM(eal_ml) as total_eal_ml,
       SUM(eal_high) as total_eal_high
FROM findings 
WHERE scan_id = ? 
GROUP BY attack_type_code
```

Maps to scan_totals_automated columns:
- PHISHING_BEC → phishing_bec_low/ml/high
- SITE_HACK → site_hack_low/ml/high  
- MALWARE → malware_low/ml/high
- ADA_COMPLIANCE → ada_compliance_low/ml/high
- DENIAL_OF_WALLET → dow_daily_low/ml/high

## Configuration and Maintenance

### Adjusting Financial Impacts
```sql
-- Update base impact for attack type
UPDATE attack_meta 
SET raw_weight = 750000 
WHERE attack_type_code = 'SITE_HACK';
```

### Adding New Finding Types
```sql
-- Map new finding type to attack category
INSERT INTO finding_type_mapping (finding_type, attack_type_code, custom_multiplier)
VALUES ('NEW_FINDING_TYPE', 'SITE_HACK', 1.2);
```

### Modifying Risk Constants
```sql
-- Adjust confidence intervals
UPDATE risk_constants 
SET value = 4.0 
WHERE key = 'HIGH_CONFIDENCE';
```

## Quality Assurance

### Validation Queries
```sql
-- Check EAL calculation completeness
SELECT 
    COUNT(*) as total_findings,
    COUNT(eal_ml) as findings_with_eal,
    AVG(eal_ml) as avg_eal_ml
FROM findings 
WHERE scan_id = 'YOUR_SCAN_ID';

-- Identify findings without EAL values
SELECT finding_type, severity, COUNT(*) 
FROM findings 
WHERE scan_id = 'YOUR_SCAN_ID' AND eal_ml IS NULL
GROUP BY finding_type, severity;
```

### Summary Views
```sql
-- Get scan EAL summary
SELECT * FROM scan_eal_summary WHERE scan_id = 'YOUR_SCAN_ID';

-- Get detailed findings with EAL
SELECT finding_type, severity, eal_low, eal_ml, eal_high, eal_daily 
FROM findings 
WHERE scan_id = 'YOUR_SCAN_ID'
ORDER BY eal_ml DESC;
```

## Benefits and Outcomes

1. **Quantified Risk Management**: Converts qualitative security findings into quantifiable financial impact estimates

2. **Data-Driven Prioritization**: Enables risk-based prioritization of security remediation efforts

3. **Regulatory Compliance**: Provides structured approach to legal and compliance risk assessment

4. **Cost-Benefit Analysis**: Supports business case development for security investments

5. **Trend Analysis**: Enables tracking of risk exposure changes over time

6. **Stakeholder Communication**: Provides business-relevant metrics for executive reporting

## Limitations and Considerations

- **Point-in-Time Assessment**: EAL values represent risk at time of calculation, not future projections
- **Industry Variability**: Base impact values may need adjustment for specific industry sectors
- **Correlation Risks**: Multiple findings may have overlapping or correlated impacts not captured in simple summation
- **External Factors**: Market conditions, regulatory changes, and threat landscape evolution may affect accuracy
- **Confidence Intervals**: Estimates represent ranges rather than precise values due to inherent uncertainty in risk quantification

## Future Enhancements

- **Machine Learning Integration**: Incorporate ML models for dynamic risk factor adjustment
- **Industry Benchmarking**: Add industry-specific impact multipliers
- **Threat Intelligence Integration**: Connect with threat feeds for dynamic prevalence updates
- **Monte Carlo Simulation**: Implement statistical modeling for more sophisticated risk calculations
- **Business Context Integration**: Include company-specific factors (revenue, industry, geography) in calculations