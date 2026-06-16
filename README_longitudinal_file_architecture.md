# README: Recommended Longitudinal File Architecture for the AD/ADRD CDE Project

**Project context:** NIH/NIA RFA-AG-25-001, *Deriving Common Data Elements from Real-World Data for Alzheimer's Disease (AD) and AD-Related Dementias (ADRD)*.

**Primary objective:** Define a reusable, enclave-ready longitudinal data architecture for linking NIA-funded longitudinal study data with CMS CCW claims, EHR-derived real-world data, and harmonized Common Data Elements (CDEs) for AD/ADRD research.

**Recommended core analytic grain:** Person-month, supplemented by study-visit, event-level, CDE, mapping, provenance, and validation files.

**Security assumption:** Files may be created inside a secure NIH/CMS enclave or other approved repository. Direct identifiers should not be exported. Publicly shared artifacts should include code, specifications, value sets, metadata, and synthetic examples unless data sharing is explicitly permitted.

---

## 1. RFA alignment

This architecture is designed to support the RFA's requirements to:

1. Develop CDEs for NIA-funded AD/ADRD studies using EHR and CMS claims data.
2. Harmonize NIA longitudinal study data with real-world data (RWD).
3. Use CMS CCW files, or NIA LINKAGE data where CCW files are not available, to establish AD/ADRD research longitudinal files.
4. Incorporate all relevant CMS file types, including Part A, Part B, Part D, OASIS, and other available CMS sources.
5. Develop CDEs across seven required domains:
   - patient and caregiver demographic information;
   - disease characterization, including AD/ADRD and multiple chronic conditions (MCCs);
   - health assessment, including ADL, IADL, cognitive function, and related measures;
   - biomarkers, genetics, and genomics relevant to AD/ADRD;
   - treatment for all conditions;
   - patient and caregiver health outcomes;
   - social determinants of health, such as financial status, education, and living arrangements, excluding environmental data.
6. Use AI, ML, and NLP to create longitudinal datasets and harmonized variables, with manual validation when needed.
7. Share code, CDE definitions, harmonization documentation, and longitudinal file creation resources.
8. Maintain annual updates to CDEs, harmonized data, code, and documentation.

---

## 2. Architecture overview

The recommended data product is a **modular longitudinal data resource**, not one flat file. The architecture separates person identity, enrollment and observability, claims-derived measures, EHR-derived measures, NIA study visit measures, CDEs, mapping, algorithm provenance, validation, and use-case-specific cohort files.

### Recommended files

| File ID | File name | Grain | Purpose |
|---|---|---|---|
| 00 | `dataset_manifest.csv` | One row per released file | Documents all files, versions, source systems, dates, row counts, access levels, and checksums. |
| 01 | `person_master.csv` | One row per person | Defines the analytic population and stable person-level attributes. |
| 02 | `person_month_enrollment_observability.csv` | One row per person-month | Establishes whether a person is observable in CMS, EHR, and NIA study data during each month. |
| 03 | `claims_person_month.csv` | One row per person-month | Summarizes CMS claims-derived diagnoses, utilization, treatment, costs, and AD/ADRD-related measures. |
| 04 | `ehr_person_month.csv` | One row per person-month | Summarizes EHR-derived structured and NLP-derived measures by month. |
| 05 | `nia_study_visit_wave.csv` | One row per person per NIA study visit/wave | Preserves study-specific longitudinal assessments such as cognition, function, biomarkers, caregiver measures, and survey variables. |
| 06 | `harmonized_cde_person_time.csv` | One row per person-time-CDE | Stores harmonized CDE values in long format across the seven domains. |
| 07 | `cde_codebook.csv` | One row per CDE | Provides the CDE dictionary, definitions, permissible values, time-grain, source precedence, and validation requirements. |
| 08 | `source_to_cde_mapping.csv` | One row per source-to-CDE mapping rule | Documents how raw study, CMS, and EHR variables/codes map to target CDEs. |
| 09 | `algorithm_provenance.csv` | One row per algorithm or model version | Documents rule-based, AI, ML, and NLP algorithms used to derive variables and CDEs. |
| 10 | `quality_validation_summary.csv` | One row per validation check | Stores schema, completeness, linkage, longitudinal consistency, phenotype, privacy, and validation check results. |
| 11 | `research_use_case_cohort.csv` | One row per person per use case | Defines use-case-specific analytic cohorts, index dates, exposure groups, outcomes, censoring, and weights. |

Optional event-level detail files may be maintained inside the secure enclave, such as `cms_claim_event.csv`, `cms_claim_line_event.csv`, `part_d_fill_event.csv`, `ehr_encounter_event.csv`, `ehr_note_nlp_extract_event.csv`, and `care_transition_event.csv`. These files support traceability but may be too sensitive or too large for broad release.

---

## 3. Global data conventions

### 3.1 Identifier conventions

| Convention | Description |
|---|---|
| `person_id` | Project-wide pseudonymized person identifier. This is the primary person key across analytic files. It should not be a raw CMS beneficiary ID, medical record number, or NIA study participant ID. |
| `caregiver_person_id` | Project-wide pseudonymized caregiver identifier, used only when caregiver data are available and linkage is approved. |
| `source_person_id_hash` | Optional source-specific hashed identifier. Used for linkage auditing inside the enclave. Do not export unless permitted. |
| `cms_enclave_person_id` | Optional restricted CMS linkage key. Keep in the enclave only. Do not include in public or broadly shared data. |
| `visit_id`, `claim_id`, `encounter_id` | Source-level event identifiers. Include only in restricted files or in deidentified hashed form. |

### 3.2 Date conventions

| Convention | Description |
|---|---|
| Dates | ISO 8601 format: `YYYY-MM-DD`. |
| Month variables | `month_start` should be the first day of the month; `month_end` should be the last day of the month. |
| Relative time | `months_from_index` is recommended for cross-study harmonization and privacy-preserving longitudinal analyses. |
| Restricted dates | Exact dates may need to remain inside the enclave. Public or limited-release versions may use month, quarter, year, or relative time. |
| Date shifting | If source dates are shifted, the file should document whether within-person intervals are preserved and whether calendar-time analyses are valid. |

### 3.3 Recommended missingness codes

| Code | Meaning |
|---|---|
| `OBSERVED_NO` | Person was observable and the event or condition was not observed. |
| `OBSERVED_YES` | Person was observable and the event or condition was observed. |
| `NOT_OBSERVABLE` | Person was not enrolled or not observable in the relevant source for the period. |
| `NOT_COLLECTED` | Variable was not collected by the source study/site. |
| `NOT_APPLICABLE` | Variable does not apply to the person or period. |
| `UNKNOWN` | Source indicates unknown. |
| `SUPPRESSED` | Value suppressed for privacy, small cell size, or policy reason. |
| `PENDING_VALIDATION` | Value was generated but is awaiting validation. |

### 3.4 Source precedence convention

When the same concept is available from multiple sources, CDE derivation should specify a source precedence rule. A default example is:

1. NIA adjudicated study measure;
2. structured EHR measure;
3. validated NLP-derived EHR measure;
4. CMS claims algorithm;
5. self-report or caregiver report;
6. missing/unknown.

The precedence order should be CDE-specific and documented in `cde_codebook.csv` and `source_to_cde_mapping.csv`.

### 3.5 Recommended coding systems

Use standard code systems wherever possible:

| Domain | Candidate coding systems |
|---|---|
| Diagnoses | ICD-9-CM, ICD-10-CM, SNOMED CT, OMOP concept IDs if applicable |
| Procedures | CPT, HCPCS, ICD-9-PCS, ICD-10-PCS, OMOP concept IDs if applicable |
| Medications | NDC, RxNorm, ATC if applicable |
| Labs and assessments | LOINC, local instrument codes, study-specific item IDs |
| EHR data model | FHIR, USCDI, OMOP, PCORnet, or site-native model, depending on infrastructure |
| SDOH | USCDI SDOH elements, PRAPARE, Hunger Vital Sign, AHC-HRSN, site-specific screening tools |

---

# 4. Detailed file specifications

---

## 4.1 `dataset_manifest.csv`

### Purpose

`dataset_manifest.csv` is the release-level inventory for the data product. It tells users what files exist, what each file contains, what version was released, what source systems were used, what time period is covered, and whether access restrictions apply. This file supports FAIR data principles, reproducibility, auditability, and annual updates.

### Grain

One row per data file per release version.

### Primary key

`file_id`, `file_version`, `data_product_release`

### Column specification

| Column | Type | Required | Description |
|---|---:|---:|---|
| `file_id` | string | Yes | Stable short identifier for the file, such as `00`, `01`, or `06`. |
| `file_name` | string | Yes | Exact file name in the release package, for example `person_master.csv`. |
| `file_label` | string | Yes | Human-readable file title. |
| `file_description` | string | Yes | Brief description of the file's contents and intended use. |
| `file_version` | string | Yes | Version of the individual file, for example `v1.0.0`. |
| `data_product_release` | string | Yes | Overall release label, for example `2026Q1_release`. |
| `grain` | string | Yes | Unit of observation, such as `person`, `person_month`, `person_visit`, `person_time_cde`, or `mapping_rule`. |
| `primary_key` | string | Yes | Comma-delimited list of primary key columns. |
| `foreign_keys` | string | No | Comma-delimited list of foreign keys and referenced files. |
| `population_covered` | string | Yes | Description of people represented in the file, such as all linked NIA study participants or use-case-specific cohort members. |
| `source_systems` | string | Yes | Source systems used, for example `NIA study; CMS CCW; EHR; NLP`. |
| `cms_files_used` | string | No | CMS file families used, such as `MBSF; inpatient; outpatient; carrier; SNF; HHA; hospice; PDE; OASIS`. |
| `start_date` | date | No | Earliest date represented in the file, if releasable. |
| `end_date` | date | No | Latest date represented in the file, if releasable. |
| `relative_start` | integer | No | Earliest relative time, such as minimum `months_from_index`. Useful when exact dates are suppressed. |
| `relative_end` | integer | No | Latest relative time, such as maximum `months_from_index`. |
| `row_count` | integer | Yes | Number of rows in the released file. |
| `person_count` | integer | No | Number of unique `person_id` values represented. |
| `refresh_date` | date | Yes | Date the file was generated or refreshed. |
| `contains_phi_flag` | boolean | Yes | Indicates whether the file contains PHI or restricted information. |
| `access_level` | string | Yes | Recommended values: `public_metadata`, `controlled_access`, `enclave_only`, `synthetic_only`. |
| `privacy_suppression_applied_flag` | boolean | Yes | Indicates whether values were suppressed or generalized for privacy. |
| `checksum_sha256` | string | Recommended | SHA-256 checksum for file integrity verification. |
| `created_by_pipeline` | string | Yes | Name of the pipeline that generated the file. |
| `pipeline_version` | string | Yes | Version of the pipeline. |
| `code_repository_path` | string | Recommended | Path or URL to code that generated the file, if shareable. |
| `documentation_path` | string | Recommended | Path or URL to related documentation. |
| `known_limitations` | string | No | Important limitations users should know before analysis. |
| `notes` | string | No | Additional comments. |

---

## 4.2 `person_master.csv`

### Purpose

`person_master.csv` is the person-level backbone of the architecture. It defines who is included in the linked AD/ADRD longitudinal resource and stores stable or baseline person characteristics needed to interpret longitudinal records. It should include NIA study participation, CMS linkage status, EHR linkage status, baseline demographic variables, index date metadata, baseline and follow-up windows, caregiver linkage availability, and high-level cohort inclusion/exclusion information.

This file should not include raw direct identifiers. It should use pseudonymized IDs and privacy-preserving demographic fields appropriate for the enclave or release setting.

### Grain

One row per person.

### Primary key

`person_id`

### Column specification

| Column | Type | Required | Description |
|---|---:|---:|---|
| `person_id` | string | Yes | Project-wide pseudonymized person identifier. Primary key used across all analytic files. |
| `nia_study_id` | string | Yes | Identifier for the NIA-funded study contributing the participant. If a person appears in multiple studies, use a separate crosswalk table or pipe-delimited list. |
| `nia_participant_id_hash` | string | Restricted | Hashed study participant ID used for linkage auditing. Keep restricted unless approved. |
| `cms_linkage_status` | string | Yes | CMS linkage result. Recommended values: `linked`, `not_linked`, `pending`, `not_eligible`, `linkage_failed`. |
| `cms_enclave_person_id` | string | Restricted | Enclave-only CMS beneficiary linkage key or hashed equivalent. Do not release publicly. |
| `ehr_linkage_status` | string | Yes | EHR linkage result. Recommended values: `linked`, `not_linked`, `pending`, `not_available`, `linkage_failed`. |
| `caregiver_linkage_status` | string | No | Indicates whether caregiver data are linked. Recommended values: `linked`, `not_linked`, `not_collected`, `pending`. |
| `sex_at_birth` | string | Recommended | Sex assigned at birth if available and permitted. Recommended values should follow project CDE codebook. |
| `gender_identity` | string | Optional | Gender identity if collected and permitted. Use controlled categories with `UNKNOWN` and `NOT_COLLECTED`. |
| `race` | string | Recommended | Harmonized race category. May be multi-select or coded according to project CDE rules. |
| `ethnicity` | string | Recommended | Harmonized ethnicity category. |
| `birth_year` | integer | Restricted/Recommended | Year of birth. May be replaced with age bands or age at index for privacy. |
| `age_at_index` | numeric | Recommended | Age in years at `index_date`. Use age bands if exact age is restricted. |
| `age_band_at_index` | string | Recommended | Age band such as `65-69`, `70-74`, `75-79`, `80-84`, `85+`. Useful for privacy-preserving release. |
| `index_date` | date | Recommended/Restricted | Date anchoring longitudinal follow-up, such as first AD/ADRD diagnosis, study enrollment, first qualifying encounter, or use-case-specific index. May be suppressed or converted to month. |
| `index_month` | date | Recommended | First day of index month. Preferred when exact dates are restricted. |
| `index_event_type` | string | Yes | Definition of the index event, such as `first_ad_adrd_claim`, `study_enrollment`, `first_ehr_diagnosis`, `first_cognitive_assessment`, or `use_case_specific`. |
| `baseline_start_date` | date | Recommended | Start of baseline lookback period. Often 6 or 12 months before index. May be represented as relative time. |
| `baseline_end_date` | date | Recommended | End of baseline lookback period, usually day before index. |
| `followup_start_date` | date | Recommended | Start of follow-up. Often index date or first day after index. |
| `followup_end_date` | date | Recommended | Planned or observed follow-up end date. May reflect censoring, death, data end, or study design. |
| `death_date_available_flag` | boolean | Recommended | Indicates whether death information is available from CMS, NIA study, EHR, or another source. |
| `death_month` | date | Restricted/Recommended | First day of month of death when exact death date is restricted. |
| `last_observed_date` | date | Recommended | Last date person is observable in any source used by the project. May be generalized to month. |
| `baseline_ad_adrd_status` | string | Recommended | AD/ADRD status at baseline. Recommended values: `no_evidence`, `prevalent_ad_adrd`, `mci`, `unknown`, `not_observable`. |
| `ad_adrd_case_source` | string | Recommended | Source supporting AD/ADRD case status, such as `claims`, `ehr`, `nia_adjudicated`, `self_report`, `combined_algorithm`. |
| `baseline_mcc_count` | integer | Recommended | Count of baseline multiple chronic condition categories using the project CDE algorithm. |
| `baseline_living_arrangement` | string | Recommended | Harmonized baseline living arrangement if available, such as `alone`, `with_spouse_partner`, `with_family`, `assisted_living`, `nursing_home`, `unknown`. |
| `baseline_education_level` | string | Recommended | Harmonized education category. Use CDE categories and preserve source detail in mapping files. |
| `baseline_economic_security_category` | string | Optional | Harmonized financial/economic security category where available. |
| `baseline_geography_level` | string | Optional | Geographic level available for analysis, such as `state`, `region`, `county`, or `suppressed`. |
| `baseline_state_or_region` | string | Optional/Restricted | State, census region, or other permitted geography. Use privacy rules before release. |
| `primary_caregiver_available_flag` | boolean | Recommended | Indicates whether a primary caregiver is documented in NIA study or EHR data. |
| `primary_caregiver_person_id` | string | Optional/Restricted | Pseudonymized caregiver ID if caregiver-level data are linked and approved. |
| `data_contributor_sites` | string | Restricted/Optional | Contributing study sites or EHR sites. May be generalized or suppressed. |
| `person_inclusion_flag` | boolean | Yes | Indicates whether the person is included in the core longitudinal resource. |
| `exclusion_reason` | string | No | Reason for exclusion if `person_inclusion_flag = false`, such as missing linkage, invalid dates, no observable baseline, or privacy restriction. |
| `deidentification_level` | string | Yes | Recommended values: `enclave_identifiable`, `limited_dataset`, `deidentified`, `synthetic`. |
| `record_status` | string | Yes | Recommended values: `active`, `updated`, `retired`, `excluded`. |
| `created_date` | date | Yes | Date this person-level record was created. |
| `updated_date` | date | Yes | Date this person-level record was last updated. |
| `person_record_version` | string | Yes | Version of the person-level record logic. |

### Notes

- `person_master.csv` should not be used alone to infer longitudinal observability. A person can appear in `person_master.csv` but have only selected months observable in CMS, EHR, or NIA study data.
- Age, geography, caregiver linkage, and genetics-related variables may require stricter privacy handling.
- Use `research_use_case_cohort.csv` for use-case-specific inclusion criteria rather than overloading `person_master.csv`.

---

## 4.3 `person_month_enrollment_observability.csv`

### Purpose

`person_month_enrollment_observability.csv` is the most important file for valid CMS-based longitudinal analysis. It defines the month-by-month periods in which each person is observable in CMS, EHR, NIA study, and other linked data. It allows analysts to distinguish a true absence of claims or events from lack of enrollment, missing source data, Medicare Advantage coverage, death, disenrollment, or other gaps.

This file is the recommended skeleton for person-month analyses.

### Grain

One row per person per calendar month or relative month in the study window.

### Primary key

`person_id`, `month_start`

### Column specification

| Column | Type | Required | Description |
|---|---:|---:|---|
| `person_id` | string | Yes | Project-wide pseudonymized person identifier. Foreign key to `person_master.csv`. |
| `month_start` | date | Yes | First day of the month represented by the row. |
| `month_end` | date | Yes | Last day of the month represented by the row. |
| `calendar_year` | integer | Recommended | Calendar year of `month_start`. Suppress if calendar dates are not releasable. |
| `calendar_month` | integer | Recommended | Calendar month number, 1-12. Suppress if calendar dates are not releasable. |
| `index_month` | date | Recommended | First day of the person's index month. |
| `months_from_index` | integer | Yes | Number of months between `month_start` and `index_month`. Negative values indicate baseline months; zero is the index month; positive values are follow-up months. |
| `age_at_month` | numeric | Recommended | Age in years at `month_start`, or approximate age if exact date of birth is not available. |
| `age_band_at_month` | string | Recommended | Age band at `month_start`, useful for privacy-preserving analysis. |
| `cms_beneficiary_month_flag` | boolean | Recommended | Indicates person appears in a CMS denominator or beneficiary enrollment file for the month. |
| `enrolled_medicare_part_a_flag` | boolean | Recommended | Indicates Medicare Part A enrollment during the month. Needed for inpatient, SNF, hospice, and home health observability. |
| `enrolled_medicare_part_b_flag` | boolean | Recommended | Indicates Medicare Part B enrollment during the month. Needed for outpatient and carrier observability. |
| `enrolled_medicare_part_d_flag` | boolean | Recommended | Indicates Part D enrollment during the month. Needed for prescription drug event observability. |
| `enrolled_medicaid_flag` | boolean | Optional | Indicates Medicaid enrollment during the month if Medicaid data are available. |
| `dual_eligible_flag` | boolean | Recommended | Indicates Medicare-Medicaid dual eligibility during the month. |
| `medicare_advantage_flag` | boolean | Recommended | Indicates Medicare Advantage or HMO enrollment during the month. Traditional fee-for-service claims may be incomplete during such months unless encounter data are available and validated. |
| `ffs_claims_observable_flag` | boolean | Yes | Indicates whether traditional Medicare FFS claims are expected to be complete enough for claims-based event ascertainment in this month. Usually requires Part A/B enrollment and no Medicare Advantage, depending on the analysis. |
| `part_a_observable_flag` | boolean | Recommended | Indicates Part A claim observability for the month. |
| `part_b_observable_flag` | boolean | Recommended | Indicates Part B/carrier/outpatient claim observability for the month. |
| `part_d_observable_flag` | boolean | Recommended | Indicates Part D drug fill observability for the month. |
| `medicaid_observable_flag` | boolean | Optional | Indicates Medicaid observability for the month, if Medicaid data are included. |
| `oasis_observable_flag` | boolean | Optional | Indicates whether OASIS home health assessment data are potentially observable for the person-month. This may depend on home health use and data availability. |
| `ehr_observable_flag` | boolean | Recommended | Indicates whether EHR data from participating sites are available for the person during the month. This does not guarantee an encounter occurred. |
| `nia_study_observed_flag` | boolean | Recommended | Indicates whether the month overlaps an NIA study observation window, visit window, or follow-up interval. |
| `source_observability_summary` | string | Recommended | Compact summary of observable sources, such as `CMS_FFS; PartD; EHR`. |
| `enrollment_source_file` | string | Recommended | Source file used to derive enrollment, such as CMS MBSF or NIA LINKAGE equivalent. |
| `beneficiary_residence_state` | string | Optional/Restricted | State of residence during the month if permitted. May be suppressed or replaced by region. |
| `beneficiary_residence_region` | string | Optional | Census region or other generalized geography. |
| `death_in_month_flag` | boolean | Recommended | Indicates death occurred during this month according to available sources. |
| `disenrolled_in_month_flag` | boolean | Recommended | Indicates disenrollment or loss of required coverage during this month. |
| `censoring_date` | date | Recommended/Restricted | Date of censoring if censoring occurs in this month. May be represented by month. |
| `censoring_reason` | string | Recommended | Reason follow-up should stop, such as `death`, `disenrollment`, `end_of_data`, `loss_to_followup`, `medicare_advantage`, `study_end`, `none`. |
| `observation_gap_flag` | boolean | Recommended | Indicates a gap in observability relative to the required source for the analysis. |
| `continuous_baseline_6m_flag` | boolean | Recommended | Indicates continuous required observability during the 6 months before index. |
| `continuous_baseline_12m_flag` | boolean | Recommended | Indicates continuous required observability during the 12 months before index. |
| `continuous_followup_to_month_flag` | boolean | Recommended | Indicates continuous required observability from index through this month. |
| `analytic_person_month_flag` | boolean | Yes | Indicates the row is eligible for the default person-month analytic file. This should be based on project-defined observability rules. |
| `person_month_weight` | numeric | Optional | Optional weight for survey integration, inverse probability weighting, or use-case-specific design. |
| `created_date` | date | Yes | Date the row was created. |
| `algorithm_version` | string | Yes | Version of the enrollment/observability algorithm. |

### Notes

- Do not treat missing claims as no utilization unless the relevant observability flag is true.
- Medication CDEs should generally require `part_d_observable_flag = true` unless medication data come from EHR or another validated source.
- Some analyses may allow Medicare Advantage months if validated encounter data are available; this rule should be explicit in the use-case cohort file and codebook.

---

## 4.4 `claims_person_month.csv`

### Purpose

`claims_person_month.csv` is the CMS-derived longitudinal utilization, diagnosis, treatment, and cost file. It aggregates raw CMS claims and claim lines into person-month measures. It should support common AD/ADRD analyses, including disease identification, multiple chronic condition measurement, treatment patterns, health services use, care transitions, costs, and outcomes.

This file is intentionally summarized to a person-month grain. Event-level traceability should be preserved in restricted event files or provenance metadata.

### Grain

One row per person per month with CMS claims observability or within the analytic skeleton.

### Primary key

`person_id`, `month_start`

### Column specification

| Column | Type | Required | Description |
|---|---:|---:|---|
| `person_id` | string | Yes | Project-wide pseudonymized person identifier. |
| `month_start` | date | Yes | First day of the month. Foreign key to `person_month_enrollment_observability.csv`. |
| `month_end` | date | Yes | Last day of the month. |
| `months_from_index` | integer | Recommended | Relative month from index. Duplicated for convenience; authoritative value is in the observability file. |
| `ffs_claims_observable_flag` | boolean | Yes | Copied from observability file to identify months where FFS claims are expected to be complete. |
| `part_d_observable_flag` | boolean | Recommended | Copied from observability file to identify months where drug fills are observable. |
| `cms_source_files_present` | string | Recommended | CMS file families contributing to this row, such as `inpatient; outpatient; carrier; SNF; HHA; hospice; PDE; DME; OASIS`. |
| `total_claim_count` | integer | Recommended | Count of unique claims observed across included CMS claim files during the month. |
| `total_claim_line_count` | integer | Recommended | Count of claim lines observed across included claim files. Useful for auditing over-counting. |
| `any_claim_flag` | boolean | Recommended | Indicates at least one CMS claim was observed during the month. |
| `inpatient_admission_count` | integer | Recommended | Count of acute inpatient admissions beginning in the month. Use stay-level logic to avoid double counting transfers or claim segments. |
| `inpatient_days` | integer | Recommended | Total inpatient days attributed to the month. Specify attribution rule for stays spanning months. |
| `any_inpatient_flag` | boolean | Recommended | Indicates at least one inpatient admission or inpatient stay day in the month. |
| `ed_visit_count` | integer | Recommended | Count of emergency department visits derived from outpatient, carrier, or inpatient revenue/HCPCS/place-of-service logic. |
| `any_ed_flag` | boolean | Recommended | Indicates one or more ED visits during the month. |
| `outpatient_visit_count` | integer | Recommended | Count of outpatient facility visits during the month. |
| `carrier_service_count` | integer | Optional | Count of professional/carrier service claims or lines, depending on project rule. |
| `primary_care_visit_count` | integer | Optional | Count of claims classified as primary care visits. |
| `neurology_visit_count` | integer | Optional | Count of neurology visits based on provider specialty or claim logic. |
| `psychiatry_visit_count` | integer | Optional | Count of psychiatry or behavioral health visits. |
| `snf_stay_count` | integer | Recommended | Count of skilled nursing facility stays beginning or active in the month. |
| `snf_days` | integer | Recommended | Number of SNF days attributed to the month. |
| `home_health_episode_count` | integer | Recommended | Count of home health episodes or claims during the month. |
| `home_health_visit_count` | integer | Optional | Count of home health visits, if derivable. |
| `hospice_use_flag` | boolean | Recommended | Indicates hospice enrollment, claim, or hospice day during the month. |
| `hospice_days` | integer | Recommended | Number of hospice days attributed to the month. |
| `dme_claim_count` | integer | Optional | Count of durable medical equipment claims. |
| `oasis_assessment_count` | integer | Optional | Count of OASIS assessments linked to the person-month, when available. |
| `part_d_fill_count` | integer | Recommended | Count of Part D prescription drug event fills during the month. Requires Part D observability for interpretation. |
| `total_days_supply` | integer | Recommended | Total days supply across Part D fills in the month. May exceed days in month. |
| `unique_ndc_count` | integer | Optional | Count of distinct NDCs filled during the month. |
| `unique_rxnorm_ingredient_count` | integer | Optional | Count of distinct RxNorm ingredients after NDC-to-RxNorm mapping. |
| `anti_dementia_drug_fill_flag` | boolean | Recommended | Indicates fill for anti-dementia medication class, such as cholinesterase inhibitors or memantine, based on project value set. |
| `anti_dementia_days_supply` | integer | Recommended | Days supply for anti-dementia medication fills during the month. |
| `antipsychotic_fill_flag` | boolean | Optional | Indicates antipsychotic medication fill during the month. Useful for behavioral symptoms and safety research. |
| `antidepressant_fill_flag` | boolean | Optional | Indicates antidepressant medication fill during the month. |
| `opioid_fill_flag` | boolean | Optional | Indicates opioid medication fill during the month. |
| `polypharmacy_count` | integer | Optional | Count of active medication ingredients or classes during the month, based on fill dates and days supply. |
| `ad_adrd_any_dx_flag` | boolean | Recommended | Indicates any AD/ADRD diagnosis code observed in included claims during the month. |
| `ad_adrd_primary_dx_flag` | boolean | Optional | Indicates AD/ADRD diagnosis appears in a primary/principal diagnosis position during the month. |
| `ad_adrd_dx_claim_count` | integer | Recommended | Number of claims with AD/ADRD diagnosis code during the month. |
| `mci_dx_flag` | boolean | Optional | Indicates mild cognitive impairment diagnosis code observed during the month. |
| `delirium_dx_flag` | boolean | Optional | Indicates delirium diagnosis code observed during the month. |
| `depression_dx_flag` | boolean | Optional | Indicates depression diagnosis code observed during the month. |
| `diabetes_dx_flag` | boolean | Optional | Indicates diabetes diagnosis code observed during the month. |
| `heart_failure_dx_flag` | boolean | Optional | Indicates heart failure diagnosis code observed during the month. |
| `stroke_tia_dx_flag` | boolean | Optional | Indicates stroke or transient ischemic attack diagnosis code observed during the month. |
| `ckd_dx_flag` | boolean | Optional | Indicates chronic kidney disease diagnosis code observed during the month. |
| `copd_dx_flag` | boolean | Optional | Indicates COPD diagnosis code observed during the month. |
| `mcc_condition_count` | integer | Recommended | Count of project-defined chronic condition categories observed or active through this month. The precise logic should be in `cde_codebook.csv`. |
| `new_mcc_condition_count` | integer | Optional | Count of new chronic condition categories first detected during the month. |
| `cognitive_testing_claim_count` | integer | Optional | Count of claims for cognitive testing or neuropsychological assessment based on CPT/HCPCS value sets. |
| `neuroimaging_claim_count` | integer | Optional | Count of neuroimaging claims relevant to AD/ADRD workup. |
| `lab_or_biomarker_claim_count` | integer | Optional | Count of claims for relevant lab/biomarker services when identifiable in claims. Claims may not capture results. |
| `care_transition_count` | integer | Recommended | Count of transitions across care settings, such as hospital to SNF, hospital to home health, or SNF to community, based on event sequencing. |
| `institutional_care_flag` | boolean | Optional | Indicates evidence of institutional care during the month, such as SNF, nursing facility, long-stay setting, or related source evidence. |
| `total_paid_amount` | numeric | Recommended | Total paid amount across included claims, using project-specified payment variables and inflation adjustment rules if applicable. |
| `medicare_paid_amount` | numeric | Recommended | Medicare-paid amount across included claims. |
| `beneficiary_out_of_pocket_amount` | numeric | Optional | Deductible, coinsurance, copayment, or other beneficiary liability if available and valid. |
| `part_d_total_drug_cost` | numeric | Optional | Total gross drug cost or project-specified Part D cost measure. |
| `cost_year_adjusted_to` | integer | Optional | Reference year for inflation-adjusted cost variables. |
| `coding_systems_present` | string | Recommended | Diagnosis/procedure code systems present in the source records contributing to the month, such as `ICD9CM; ICD10CM; CPT; HCPCS; NDC`. |
| `icd_era` | string | Recommended | Recommended values: `ICD9`, `ICD10`, `mixed`, `not_applicable`. Useful for studies spanning October 2015. |
| `deduplication_method` | string | Recommended | Summary of method used to prevent claim-line or split-claim double counting. |
| `claims_algorithm_version` | string | Yes | Version of the claims aggregation and phenotype algorithms. |
| `claims_data_quality_flag` | string | Recommended | Recommended values: `pass`, `warning`, `fail`, `not_observable`. Details should appear in validation file. |

### Notes

- Wide condition flags in this file are convenience variables. The authoritative harmonized variables should be represented as CDEs in `harmonized_cde_person_time.csv`.
- Avoid double counting multi-line claims and split institutional stays.
- Costs should be documented carefully because source payment variables differ across CMS files and years.
- Claims indicate billed/covered care, not necessarily full clinical status or disease severity.

---

## 4.5 `ehr_person_month.csv`

### Purpose

`ehr_person_month.csv` captures structured and unstructured EHR-derived longitudinal information that may not be available in CMS claims. This includes cognitive assessments, ADL/IADL, problem list diagnoses, laboratory results, medications ordered or administered, caregiver documentation, social risk screening, clinical notes processed by NLP, and EHR encounter patterns.

This file helps fill gaps in claims data and supports CDEs in domains where claims are incomplete, especially health assessment, SDOH, caregiver information, and disease severity.

### Grain

One row per person per month where EHR data are available or aligned to the person-month skeleton.

### Primary key

`person_id`, `month_start`, optionally `ehr_site_id` if multiple EHR sites are maintained separately.

### Column specification

| Column | Type | Required | Description |
|---|---:|---:|---|
| `person_id` | string | Yes | Project-wide pseudonymized person identifier. |
| `month_start` | date | Yes | First day of the month. |
| `month_end` | date | Yes | Last day of the month. |
| `months_from_index` | integer | Recommended | Relative month from index. |
| `ehr_site_id` | string | Optional/Restricted | Pseudonymized EHR site identifier. Include only if site-level analysis is permitted. |
| `ehr_observable_flag` | boolean | Yes | Indicates whether EHR data are available for the person-month from participating EHR sources. |
| `encounter_count` | integer | Recommended | Count of EHR encounters during the month. |
| `outpatient_encounter_count` | integer | Optional | Count of outpatient EHR encounters. |
| `inpatient_encounter_count` | integer | Optional | Count of inpatient EHR encounters documented in EHR. |
| `ed_encounter_count` | integer | Optional | Count of ED encounters documented in EHR. |
| `primary_care_encounter_count` | integer | Optional | Count of primary care encounters documented in EHR. |
| `neurology_encounter_count` | integer | Optional | Count of neurology encounters documented in EHR. |
| `behavioral_health_encounter_count` | integer | Optional | Count of psychiatry, psychology, or behavioral health encounters. |
| `problem_list_ad_adrd_flag` | boolean | Recommended | Indicates AD/ADRD appears on the EHR problem list during the month. |
| `encounter_dx_ad_adrd_flag` | boolean | Recommended | Indicates AD/ADRD diagnosis code appears on an EHR encounter diagnosis during the month. |
| `mci_ehr_flag` | boolean | Optional | Indicates mild cognitive impairment evidence in structured EHR data or validated NLP output. |
| `cognitive_assessment_recorded_flag` | boolean | Recommended | Indicates any cognitive assessment was recorded during the month. |
| `cognitive_assessment_instrument` | string | Recommended | Instrument name or code, such as MMSE, MoCA, Mini-Cog, CDR, or study/site-specific test. Multiple instruments may require a separate event file. |
| `cognitive_assessment_score_latest` | numeric | Recommended | Most recent cognitive score in the month after harmonization, if a score exists. |
| `cognitive_assessment_score_min` | numeric | Optional | Minimum cognitive score observed in the month. |
| `cognitive_assessment_score_max` | numeric | Optional | Maximum cognitive score observed in the month. |
| `cognitive_impairment_nlp_flag` | boolean | Optional | Indicates cognitive impairment concept extracted from clinical notes by validated NLP. |
| `adl_assessment_recorded_flag` | boolean | Recommended | Indicates ADL assessment documented during the month. |
| `adl_limitation_count` | integer | Recommended | Count of ADL domains with limitation, based on structured EHR or validated NLP if available. |
| `iadl_assessment_recorded_flag` | boolean | Recommended | Indicates IADL assessment documented during the month. |
| `iadl_limitation_count` | integer | Recommended | Count of IADL domains with limitation. |
| `functional_decline_nlp_flag` | boolean | Optional | Indicates functional decline extracted from notes by validated NLP. |
| `falls_ehr_flag` | boolean | Optional | Indicates fall diagnosis, screening result, or NLP-extracted fall during the month. |
| `falls_count` | integer | Optional | Count of documented falls or fall-related encounters/events in the month. |
| `delirium_ehr_flag` | boolean | Optional | Indicates delirium evidence in structured EHR or validated NLP. |
| `behavioral_symptom_flag` | boolean | Optional | Indicates behavioral or neuropsychiatric symptom evidence, such as agitation, wandering, hallucinations, or aggression. |
| `medication_order_count` | integer | Optional | Count of medication orders during the month. |
| `anti_dementia_med_order_flag` | boolean | Optional | Indicates EHR medication order for an anti-dementia medication class. |
| `antipsychotic_med_order_flag` | boolean | Optional | Indicates EHR medication order for antipsychotic medication. |
| `lab_result_count` | integer | Optional | Count of lab result records relevant to the project during the month. |
| `biomarker_result_available_flag` | boolean | Optional | Indicates AD/ADRD-relevant biomarker result is available in EHR during the month. Detailed values may be stored separately or restricted. |
| `amyloid_result_category` | string | Optional/Restricted | Harmonized amyloid biomarker category if available, such as `positive`, `negative`, `indeterminate`, `not_available`. |
| `tau_result_category` | string | Optional/Restricted | Harmonized tau biomarker category if available. |
| `genetic_result_available_flag` | boolean | Optional/Restricted | Indicates relevant genetic/genomic information is available. Do not include raw genomic data in this general file. |
| `apoe_e4_status` | string | Optional/Restricted | APOE e4 carrier status if available and approved. May require controlled access. |
| `sdoh_screening_recorded_flag` | boolean | Recommended | Indicates an SDOH or social risk screening was recorded in the month. |
| `financial_strain_flag` | boolean | Optional | Indicates financial strain or economic insecurity from structured screening or validated NLP. |
| `food_insecurity_flag` | boolean | Optional | Indicates food insecurity. |
| `housing_instability_flag` | boolean | Optional | Indicates housing instability. |
| `transportation_need_flag` | boolean | Optional | Indicates transportation-related need. |
| `social_isolation_flag` | boolean | Optional | Indicates social isolation or loneliness. |
| `living_arrangement_ehr` | string | Optional | Living arrangement from EHR source, if available. |
| `caregiver_documented_flag` | boolean | Recommended | Indicates caregiver is documented in EHR or notes during the month. |
| `caregiver_relationship` | string | Optional | Relationship of caregiver to person, if available and permitted. |
| `caregiver_burden_ehr_flag` | boolean | Optional | Indicates caregiver strain or burden documented in EHR or notes. |
| `advance_care_planning_flag` | boolean | Optional | Indicates advance care planning discussion, order, or documentation. |
| `palliative_care_flag` | boolean | Optional | Indicates palliative care involvement documented in EHR. |
| `nlp_note_count` | integer | Recommended | Count of clinical notes processed by NLP for the person-month. |
| `nlp_extracted_variable_count` | integer | Recommended | Count of variables extracted from notes for the person-month. |
| `nlp_confidence_mean` | numeric | Optional | Average NLP confidence score for extracted variables in the month. |
| `nlp_confidence_min` | numeric | Optional | Minimum NLP confidence score across extracted variables. |
| `manual_validation_status` | string | Recommended | Recommended values: `not_required`, `pending`, `sample_validated`, `fully_validated`, `failed_validation`. |
| `ehr_data_model` | string | Recommended | EHR data model or standard used, such as `FHIR`, `OMOP`, `PCORnet`, `Epic_native`, `Cerner_native`, or `site_native`. |
| `ehr_algorithm_version` | string | Yes | Version of EHR extraction and harmonization algorithms. |
| `ehr_data_quality_flag` | string | Recommended | Recommended values: `pass`, `warning`, `fail`, `not_observable`. |

### Notes

- EHR data may be encounter-biased. No EHR encounter does not necessarily mean no condition.
- NLP-derived variables should carry confidence scores and validation status.
- Biomarker/genomic data should usually be controlled-access and may require separate specialized files.

---

## 4.6 `nia_study_visit_wave.csv`

### Purpose

`nia_study_visit_wave.csv` preserves the longitudinal structure of NIA-funded studies. NIA studies often collect detailed cognitive, functional, behavioral, caregiver, biomarker, genetic, social, and survey measures at scheduled visits or waves. These data should not be forced into a monthly structure without preserving the original study visit timing and measurement context.

This file serves as the source of high-quality study measures and supports CDE harmonization across disparate NIA-funded studies.

### Grain

One row per person per study visit or survey wave.

### Primary key

`person_id`, `nia_study_id`, `visit_id`

### Column specification

| Column | Type | Required | Description |
|---|---:|---:|---|
| `person_id` | string | Yes | Project-wide pseudonymized person identifier. |
| `nia_study_id` | string | Yes | Identifier for the NIA-funded study. |
| `visit_id` | string | Yes | Pseudonymized visit or wave identifier. |
| `visit_date` | date | Restricted/Recommended | Exact visit date if releasable. Otherwise use `visit_month`, `visit_year`, or relative timing. |
| `visit_month` | date | Recommended | First day of visit month. Preferred if exact date is restricted. |
| `wave_number` | numeric | Recommended | Study wave number or visit sequence. |
| `months_from_index` | integer | Recommended | Relative timing of visit from the person's index month. |
| `months_from_baseline_study_visit` | integer | Optional | Relative timing from first observed NIA study visit. |
| `visit_type` | string | Recommended | Type of visit, such as `baseline`, `annual_followup`, `telephone`, `in_person`, `caregiver_interview`, `biomarker_visit`, `unscheduled`. |
| `visit_completed_flag` | boolean | Recommended | Indicates whether the study visit/wave was completed. |
| `nonresponse_reason` | string | Optional | Reason visit was not completed, such as death, refused, lost to follow-up, proxy unavailable, or not applicable. |
| `attrition_flag` | boolean | Recommended | Indicates attrition from the study at or after this wave. |
| `proxy_respondent_flag` | boolean | Recommended | Indicates whether a proxy or caregiver respondent provided some or all information. |
| `cognitive_global_score` | numeric | Recommended | Harmonized global cognitive score when available. The source instrument and harmonization method must be documented. |
| `cognitive_global_score_scale` | string | Recommended | Scale or standardized metric for the global cognitive score. |
| `cognitive_instrument_primary` | string | Recommended | Primary cognitive instrument used at the visit, such as MMSE, MoCA, CDR, CERAD, or study-specific battery. |
| `memory_score` | numeric | Optional | Harmonized memory domain score. |
| `executive_function_score` | numeric | Optional | Harmonized executive function score. |
| `language_score` | numeric | Optional | Harmonized language domain score. |
| `processing_speed_score` | numeric | Optional | Harmonized processing speed score. |
| `dementia_clinical_diagnosis` | string | Recommended | Study-reported or adjudicated clinical diagnosis, such as normal cognition, MCI, AD dementia, vascular dementia, Lewy body dementia, frontotemporal dementia, mixed dementia, or unknown. |
| `adrd_subtype` | string | Optional | AD/ADRD subtype if determined. |
| `cdr_global` | numeric | Optional | Clinical Dementia Rating global score, if available. |
| `cdr_sum_of_boxes` | numeric | Optional | Clinical Dementia Rating sum of boxes, if available. |
| `mmse_score` | numeric | Optional | Mini-Mental State Examination score if available. |
| `moca_score` | numeric | Optional | Montreal Cognitive Assessment score if available. |
| `adl_total_score` | numeric | Recommended | Total ADL score or harmonized ADL limitation score. |
| `adl_scale_name` | string | Recommended | Source ADL scale or harmonized scale name. |
| `iadl_total_score` | numeric | Recommended | Total IADL score or harmonized IADL limitation score. |
| `iadl_scale_name` | string | Recommended | Source IADL scale or harmonized scale name. |
| `functional_status_source` | string | Recommended | Source of functional status information, such as self-report, proxy report, clinician assessment, or performance test. |
| `depression_score` | numeric | Optional | Depression symptom score if collected. |
| `depression_scale_name` | string | Optional | Depression instrument name. |
| `neuropsychiatric_symptom_score` | numeric | Optional | Neuropsychiatric or behavioral symptom score if collected. |
| `quality_of_life_score` | numeric | Optional | Patient-reported or proxy-reported quality of life score. |
| `self_rated_health` | string | Optional | Self-rated health category, if collected. |
| `biomarker_available_flag` | boolean | Recommended | Indicates whether biomarker data are available for the visit or person. |
| `amyloid_status` | string | Optional/Restricted | Harmonized amyloid status if available and permitted. |
| `tau_status` | string | Optional/Restricted | Harmonized tau status if available and permitted. |
| `neurodegeneration_marker_status` | string | Optional/Restricted | Harmonized neurodegeneration marker category, if available. |
| `genomics_available_flag` | boolean | Optional/Restricted | Indicates genomic/genetic data are available. Raw genetic data should remain in appropriate controlled repositories. |
| `apoe_e4_status` | string | Optional/Restricted | APOE e4 carrier status if available and approved. |
| `caregiver_interview_flag` | boolean | Recommended | Indicates caregiver interview was conducted at this visit/wave. |
| `caregiver_person_id` | string | Optional/Restricted | Pseudonymized caregiver identifier, if caregiver-level linkage is approved. |
| `caregiver_relationship` | string | Optional | Relationship of caregiver to participant. |
| `caregiver_burden_score` | numeric | Optional | Caregiver burden score, if collected. |
| `caregiver_burden_scale_name` | string | Optional | Name of caregiver burden instrument. |
| `caregiver_hours_per_week` | numeric | Optional | Reported caregiving hours per week. |
| `social_support_score` | numeric | Optional | Social support score if collected. |
| `living_arrangement` | string | Recommended | Study-reported living arrangement at visit/wave. |
| `education_years` | numeric | Optional | Years of education, if collected and releasable. |
| `education_category` | string | Recommended | Harmonized education category. |
| `household_income_category` | string | Optional/Restricted | Harmonized household income category, if collected and permitted. |
| `financial_strain` | string | Optional | Study-reported financial strain category. |
| `health_insurance_status` | string | Optional | Study-reported insurance status, if collected. |
| `sample_weight` | numeric | Optional | Study-provided sample weight for survey analyses. |
| `stratum` | string | Optional/Restricted | Survey design stratum if applicable and releasable. |
| `cluster` | string | Optional/Restricted | Survey design cluster if applicable and releasable. |
| `source_variable_set_version` | string | Yes | Version of the source study data dictionary or variable set. |
| `harmonization_version` | string | Yes | Version of study harmonization logic. |
| `study_visit_quality_flag` | string | Recommended | Recommended values: `pass`, `warning`, `fail`, `partial_visit`, `not_applicable`. |

### Notes

- This file should preserve original study visit timing and context even when harmonized person-month CDE files are created.
- For irregular study visits, link visits to person-months using `visit_month` and `months_from_index`.
- Instrument-specific item-level data may require separate restricted files and mapping metadata.

---

## 4.7 `harmonized_cde_person_time.csv`

### Purpose

`harmonized_cde_person_time.csv` is the central long-format CDE file. It stores harmonized CDE values across sources, people, and time. This structure is flexible enough to represent monthly claims variables, EHR-derived variables, annual study measures, one-time baseline variables, caregiver variables, biomarker/genomic categories, and use-case-specific derived measures.

This file is designed to support cross-sectional and longitudinal analyses without requiring every CDE to be represented as a column in a wide table.

### Grain

One row per person per time interval per CDE.

### Primary key

`person_id`, `time_id`, `cde_id`, `cde_version`

### Column specification

| Column | Type | Required | Description |
|---|---:|---:|---|
| `person_id` | string | Yes | Project-wide pseudonymized person identifier. |
| `caregiver_person_id` | string | Optional/Restricted | Pseudonymized caregiver identifier for caregiver CDEs, if applicable. |
| `time_id` | string | Yes | Stable time identifier, such as `2020-01`, `index_m000`, `visit_003`, or generated interval ID. |
| `time_grain` | string | Yes | Time unit for the CDE value. Recommended values: `baseline`, `person_month`, `person_quarter`, `person_year`, `study_visit`, `event`, `ever`, `time_invariant`. |
| `time_start` | date | Recommended/Restricted | Start date of the interval represented by the CDE value. May be suppressed or generalized. |
| `time_end` | date | Recommended/Restricted | End date of the interval represented by the CDE value. |
| `months_from_index` | integer | Recommended | Relative month from index for monthly or visit-linked values. |
| `cde_id` | string | Yes | Stable unique CDE identifier. Foreign key to `cde_codebook.csv`. |
| `cde_name` | string | Yes | Machine-readable CDE name, such as `ad_adrd_status`, `adl_limitation_count`, or `part_d_anti_dementia_fill`. |
| `cde_label` | string | Recommended | Human-readable CDE label. |
| `cde_domain` | string | Yes | One of the seven RFA domains. |
| `cde_subdomain` | string | Recommended | More specific category, such as `diagnosis`, `cognition`, `function`, `medication`, `caregiver_burden`, or `economic_security`. |
| `cde_value` | string | Recommended | Canonical value as a string. Use for categorical, boolean, and display values. |
| `cde_value_numeric` | numeric | Optional | Numeric value when applicable. |
| `cde_value_text` | string | Optional/Restricted | Text value when necessary. Avoid raw note text in broadly shared files. |
| `cde_value_code` | string | Optional | Standardized code for categorical value, if applicable. |
| `cde_value_unit` | string | Optional | Unit of measurement, such as points, days, dollars, visits, or count. |
| `cde_value_date` | date | Optional/Restricted | Date value for event-date CDEs. May be generalized to month or relative time. |
| `value_type` | string | Yes | Recommended values: `boolean`, `integer`, `numeric`, `categorical`, `ordinal`, `date`, `text`, `json`. |
| `missingness_code` | string | Yes | Missingness/observability code, such as `OBSERVED_NO`, `OBSERVED_YES`, `NOT_OBSERVABLE`, `NOT_COLLECTED`, `UNKNOWN`, or `SUPPRESSED`. |
| `source_priority_rule` | string | Recommended | Rule used to select among multiple sources, such as `NIA_adjudicated_over_EHR_over_claims`. |
| `source_system` | string | Yes | Source used for the value, such as `CMS_CCW`, `NIA_STUDY`, `EHR_STRUCTURED`, `EHR_NLP`, `COMBINED_ALGORITHM`. |
| `source_file` | string | Recommended | Source file or table contributing to the value. |
| `source_variable` | string | Recommended | Source variable name or source concept used to derive the value. |
| `source_code_system` | string | Optional | Code system used, such as ICD-10-CM, CPT, HCPCS, NDC, RxNorm, LOINC, SNOMED CT, or study code. |
| `source_code` | string | Optional | Source code or value set member that triggered the CDE value, if appropriate and shareable. |
| `derived_flag` | boolean | Yes | Indicates whether the CDE value is derived rather than directly observed. |
| `derivation_algorithm_id` | string | Recommended | Foreign key to `algorithm_provenance.csv`. |
| `algorithm_version` | string | Recommended | Version of the algorithm used to generate this value. |
| `nlp_generated_flag` | boolean | Recommended | Indicates whether NLP contributed to the CDE value. |
| `ml_generated_flag` | boolean | Recommended | Indicates whether ML contributed to the CDE value. |
| `generative_ai_generated_flag` | boolean | Recommended | Indicates whether generative AI contributed to the value. Any such use should be validated and documented. |
| `confidence_score` | numeric | Optional | Confidence score from NLP/ML/AI algorithm, usually 0-1. |
| `validation_status` | string | Yes | Recommended values: `not_required`, `pending`, `validated`, `sample_validated`, `failed_validation`, `expert_adjudicated`. |
| `validation_id` | string | Optional | Foreign key to `quality_validation_summary.csv` or detailed validation record. |
| `validity_start_date` | date | Optional | Start date for which this CDE definition/mapping is valid, especially important for changing code systems. |
| `validity_end_date` | date | Optional | End date for which this CDE definition/mapping is valid. |
| `annual_release_version` | string | Yes | Release version in which the CDE value was produced. |
| `provenance_record_id` | string | Recommended | Foreign key to a detailed provenance or mapping record. |
| `created_datetime` | datetime | Yes | Timestamp when the row was generated. |

### Notes

- This long format allows one shared structure for all seven CDE domains.
- Researchers may generate wide analytic files from this file using selected CDEs and time windows.
- Use `missingness_code` to distinguish true absence from non-observability.
- CDEs based on AI, ML, or NLP should always include algorithm ID, version, confidence, and validation status.

---

## 4.8 `cde_codebook.csv`

### Purpose

`cde_codebook.csv` is the authoritative dictionary of harmonized CDEs. It defines each CDE, its scientific rationale, domain, permissible values, data type, time-grain, source systems, source precedence, required observability, value sets, update frequency, and validation expectations.

This file is essential for dissemination, reuse, and annual maintenance.

### Grain

One row per CDE per CDE version.

### Primary key

`cde_id`, `cde_version`

### Column specification

| Column | Type | Required | Description |
|---|---:|---:|---|
| `cde_id` | string | Yes | Stable unique CDE identifier, such as `ADRD_DX_001`. |
| `cde_version` | string | Yes | Version of the CDE definition. |
| `cde_name` | string | Yes | Machine-readable variable name. |
| `cde_label` | string | Yes | Human-readable label. |
| `cde_domain` | string | Yes | One of the seven RFA domains. |
| `cde_subdomain` | string | Recommended | More specific domain category. |
| `cde_definition` | string | Yes | Formal definition of the CDE. |
| `scientific_rationale` | string | Recommended | Explanation of why the CDE is important for AD/ADRD research. |
| `use_case_alignment` | string | Recommended | Use cases or AD/ADRD research questions supported by the CDE. |
| `data_type` | string | Yes | Recommended values: `boolean`, `integer`, `numeric`, `categorical`, `ordinal`, `date`, `text`, `json`. |
| `permissible_values` | string | Required for categorical | List or reference to permitted values and labels. |
| `unit` | string | Optional | Unit of measure for numeric CDEs. |
| `time_varying_flag` | boolean | Yes | Indicates whether the CDE can change over time. |
| `recommended_time_grain` | string | Yes | Recommended time grain, such as `person_month`, `study_visit`, `baseline`, `ever`, or `time_invariant`. |
| `applicable_population` | string | Recommended | Population for which the CDE is meaningful, such as all persons, AD/ADRD cases, caregivers, Part D enrolled persons. |
| `applicable_sources` | string | Yes | Sources from which the CDE can be derived, such as CMS, EHR, NIA study, NLP, or combined algorithm. |
| `required_source_observability` | string | Recommended | Required observability condition, such as Part A/B FFS, Part D, EHR encounter, study visit, or caregiver interview. |
| `source_precedence_order` | string | Recommended | Ordered list of sources used when multiple values exist. |
| `coding_systems` | string | Recommended | Relevant code systems, such as ICD-10-CM, CPT, NDC, RxNorm, LOINC, SNOMED CT, or study item IDs. |
| `code_set_id` | string | Optional | Identifier for related value set or code list. |
| `phenotype_algorithm_summary` | string | Recommended | Short description of derivation algorithm. Example: one inpatient claim or two outpatient claims separated by at least 7 days. |
| `lookback_window_days` | integer | Optional | Default lookback window used for baseline or active condition definition. |
| `washout_window_days` | integer | Optional | Washout period used for incident definitions. |
| `numerator_definition` | string | Optional | Numerator definition for rate/proportion CDEs. |
| `denominator_definition` | string | Optional | Denominator definition for rate/proportion CDEs. |
| `missingness_rules` | string | Yes | Rules for coding missing, not observable, not collected, unknown, and suppressed values. |
| `quality_checks` | string | Recommended | Required quality checks for the CDE. |
| `validation_standard` | string | Recommended | Validation reference standard, such as chart review, study adjudication, expert review, or established claims algorithm. |
| `validation_metrics` | string | Optional | Stored metrics or reference to validation file, such as sensitivity, specificity, PPV, NPV, F1, kappa. |
| `ai_ml_nlp_allowed_flag` | boolean | Recommended | Indicates whether AI/ML/NLP methods are allowed for this CDE. |
| `manual_validation_required_flag` | boolean | Recommended | Indicates whether manual validation is required. |
| `update_frequency` | string | Yes | Recommended values: `annual`, `semiannual`, `as_needed`, `retired`. |
| `cde_steward` | string | Recommended | Person, group, or committee responsible for maintaining the CDE. |
| `approval_status` | string | Yes | Recommended values: `draft`, `under_review`, `approved`, `deprecated`, `retired`. |
| `effective_start_date` | date | Recommended | Date the CDE version becomes valid. |
| `effective_end_date` | date | Optional | Date the CDE version is no longer valid. |
| `public_release_flag` | boolean | Recommended | Indicates whether the CDE definition can be shared publicly. |
| `notes` | string | Optional | Additional comments. |

### Notes

- This file should be the central resource for public-facing CDE documentation.
- Code sets may be stored in separate machine-readable files and referenced through `code_set_id`.
- CDEs should include explicit logic for changing coding schemes over time, especially ICD-9-CM to ICD-10-CM transitions.

---

## 4.9 `source_to_cde_mapping.csv`

### Purpose

`source_to_cde_mapping.csv` documents how raw variables, clinical codes, study items, EHR fields, claims fields, and NLP concepts are transformed into harmonized CDE values. It is the source-to-target crosswalk that allows CDE derivations to be audited, reused, updated, and adapted across NIA-funded studies and RWD infrastructures.

### Grain

One row per source variable/code/rule mapped to a target CDE value.

### Primary key

`mapping_id`

### Column specification

| Column | Type | Required | Description |
|---|---:|---:|---|
| `mapping_id` | string | Yes | Stable unique mapping rule identifier. |
| `cde_id` | string | Yes | Target CDE identifier. Foreign key to `cde_codebook.csv`. |
| `cde_version` | string | Yes | Version of target CDE definition. |
| `source_system` | string | Yes | Source system, such as CMS_CCW, NIA_STUDY, EHR_STRUCTURED, EHR_NLP, or NIA_LINKAGE. |
| `source_dataset` | string | Recommended | Source dataset name, such as MBSF, inpatient claims, carrier claims, PDE, OASIS, HRS, ACT, site EHR, etc. |
| `source_file_or_table` | string | Recommended | Source file or database table. |
| `source_variable_name` | string | Recommended | Source field or variable name. |
| `source_variable_label` | string | Optional | Human-readable source variable label. |
| `source_value` | string | Optional | Source value or category being mapped. |
| `source_code` | string | Optional | Source diagnosis, procedure, drug, lab, or instrument code. |
| `source_code_system` | string | Optional | Code system of `source_code`, such as ICD-10-CM, CPT, HCPCS, NDC, RxNorm, LOINC, or study code. |
| `source_code_description` | string | Optional | Text description of source code or category. |
| `target_cde_value` | string | Yes | Target CDE value after mapping. |
| `target_cde_value_code` | string | Optional | Standardized code for target value. |
| `transform_rule` | string | Yes | Transformation rule, such as direct map, recode, threshold, regex, NLP concept extraction, aggregation, or algorithmic phenotype. |
| `temporal_rule` | string | Recommended | How source timing maps to target timing, such as event month, assessment date, active medication days, baseline lookback, or ever-before-index. |
| `lookback_window_days` | integer | Optional | Lookback window used in mapping. |
| `washout_window_days` | integer | Optional | Washout period used for incident definitions. |
| `claim_position_rule` | string | Optional | Claim diagnosis position rule, such as any diagnosis, principal only, first listed, or excluding rule-out contexts. |
| `encounter_type_rule` | string | Optional | Encounter or claim type restrictions, such as inpatient only, outpatient only, carrier, ED, home health, or hospice. |
| `source_data_required` | string | Recommended | Minimum source data required for valid mapping, such as Part A/B observability or completed study visit. |
| `validity_start_date` | date | Recommended | First date the source mapping is valid. |
| `validity_end_date` | date | Optional | Last date the source mapping is valid. |
| `icd_era` | string | Optional | Recommended values: `ICD9`, `ICD10`, `mixed`, `not_applicable`. |
| `mapping_confidence` | numeric | Optional | Confidence score for mapping, especially when AI/NLP-assisted. |
| `reviewer_id` | string | Optional | Pseudonymized ID or role of reviewer who approved mapping. |
| `approval_status` | string | Yes | Recommended values: `draft`, `reviewed`, `approved`, `deprecated`, `rejected`. |
| `algorithm_id` | string | Optional | Related algorithm identifier from `algorithm_provenance.csv`. |
| `algorithm_version` | string | Optional | Version of related algorithm. |
| `change_reason` | string | Optional | Reason mapping was added or modified, such as new code system, annual update, validation issue, or stakeholder feedback. |
| `notes` | string | Optional | Additional comments. |

### Notes

- This file is critical for annual updates and evolving coding schemes.
- Use validity dates to prevent applying ICD-10 mappings to ICD-9 periods or outdated study instruments to later waves.
- Mapping files are often the most reusable public artifact even when individual-level data cannot be shared.

---

## 4.10 `algorithm_provenance.csv`

### Purpose

`algorithm_provenance.csv` documents every rule-based, AI, ML, NLP, or hybrid algorithm used to create longitudinal files, harmonized variables, phenotypes, or CDEs. It supports reproducibility, transparency, validation, and responsible use of AI/ML/NLP.

### Grain

One row per algorithm or model version.

### Primary key

`algorithm_id`, `algorithm_version`

### Column specification

| Column | Type | Required | Description |
|---|---:|---:|---|
| `algorithm_id` | string | Yes | Stable unique algorithm identifier. |
| `algorithm_version` | string | Yes | Version of the algorithm. |
| `algorithm_name` | string | Yes | Human-readable algorithm name. |
| `algorithm_purpose` | string | Yes | Purpose, such as AD/ADRD phenotype, SDOH NLP extraction, medication class derivation, claims aggregation, or CDE harmonization. |
| `related_cde_ids` | string | Recommended | CDE IDs generated or affected by the algorithm. |
| `method_type` | string | Yes | Recommended values: `rule_based`, `nlp`, `machine_learning`, `generative_ai`, `hybrid`, `manual_abstraction`. |
| `model_name` | string | Optional | Name of ML/NLP/generative AI model, if applicable. |
| `model_version` | string | Optional | Model version or checkpoint. |
| `training_data_source` | string | Optional | Training data source summary. Do not include restricted details in public metadata. |
| `training_period` | string | Optional | Time period represented in training data, if applicable. |
| `feature_sources` | string | Optional | Input feature sources, such as claims diagnoses, notes, medications, study items, or structured EHR. |
| `input_files` | string | Yes | Files or tables used as algorithm inputs. |
| `output_files` | string | Yes | Files or tables generated or modified by the algorithm. |
| `code_repository_path` | string | Recommended | Repository path, package path, or enclave project path for the code. |
| `commit_hash` | string | Recommended | Git commit hash or other immutable code version identifier. |
| `runtime_environment` | string | Recommended | Runtime environment, such as R version, Python version, Spark version, SQL engine, or container image. |
| `package_versions` | string | Recommended | Key package versions or path to environment lock file. |
| `parameter_summary` | string | Optional | Key parameters, thresholds, lookback windows, or model settings. |
| `validation_design` | string | Recommended | Validation approach, such as chart review, expert review, adjudicated study comparison, split-sample validation, external validation, or benchmarking. |
| `validation_sample_size` | integer | Optional | Number of records/persons reviewed in validation. |
| `manual_validation_required_flag` | boolean | Recommended | Indicates whether manual validation is required. |
| `manual_validation_completed_flag` | boolean | Recommended | Indicates whether required manual validation has been completed. |
| `performance_metrics_json` | string | Optional | JSON or path to metrics containing sensitivity, specificity, PPV, NPV, AUROC, F1, calibration, or inter-rater reliability. |
| `bias_fairness_checks` | string | Recommended | Description of subgroup performance checks, such as age, sex, race/ethnicity, geography, source study, or site. |
| `known_limitations` | string | Recommended | Known limitations and cautions for use. |
| `human_reviewed_by` | string | Optional | Role or pseudonymized identifier of human reviewer or committee. |
| `approval_status` | string | Yes | Recommended values: `draft`, `validated`, `approved`, `deprecated`, `retired`, `failed_validation`. |
| `approval_date` | date | Optional | Date algorithm version was approved for use. |
| `retirement_date` | date | Optional | Date algorithm was retired, if applicable. |
| `responsible_team` | string | Recommended | Team responsible for maintaining the algorithm. |
| `contact` | string | Optional | Contact email or group alias if shareable. |
| `access_restrictions` | string | Recommended | Restrictions on code, model, or data access. |
| `license` | string | Optional | License for code/model if releasable. |

### Notes

- Any generative AI use should have explicit validation documentation and should not be treated as authoritative without human or empirical validation.
- Keep a record for simple SQL/rule-based algorithms as well as advanced models.
- This file should connect to `quality_validation_summary.csv` through validation IDs or metric references.

---

## 4.11 `quality_validation_summary.csv`

### Purpose

`quality_validation_summary.csv` stores results of data quality, longitudinal consistency, linkage, phenotype, CDE, privacy, and algorithm validation checks. It supports reproducibility, reviewer confidence, annual maintenance, and transparent reporting of limitations.

### Grain

One row per validation check per file, release, subgroup, or algorithm as applicable.

### Primary key

`validation_id`

### Column specification

| Column | Type | Required | Description |
|---|---:|---:|---|
| `validation_id` | string | Yes | Stable unique validation check identifier. |
| `data_product_release` | string | Yes | Release version being validated. |
| `file_name` | string | Recommended | File being checked. Use `multiple` if the check spans files. |
| `algorithm_id` | string | Optional | Related algorithm, if the check validates an algorithm. |
| `algorithm_version` | string | Optional | Version of the related algorithm. |
| `cde_id` | string | Optional | Related CDE, if the check validates a CDE. |
| `check_name` | string | Yes | Short name of validation check. |
| `check_category` | string | Yes | Recommended values: `schema`, `completeness`, `range`, `longitudinal_continuity`, `linkage`, `claims_logic`, `ehr_logic`, `cde_mapping`, `phenotype`, `ai_ml_nlp`, `privacy`, `reproducibility`. |
| `check_description` | string | Yes | Detailed description of what was checked. |
| `denominator` | integer | Optional | Number of eligible rows/persons/events for the check. |
| `numerator` | integer | Optional | Number of rows/persons/events meeting the check condition. |
| `rate` | numeric | Optional | Computed numerator/denominator rate. |
| `expected_value` | string | Optional | Expected value, range, or threshold. |
| `observed_value` | string | Optional | Observed value or summary. |
| `threshold` | string | Optional | Pass/fail threshold. |
| `pass_fail` | string | Yes | Recommended values: `pass`, `fail`, `warning`, `not_applicable`, `pending`. |
| `severity` | string | Recommended | Recommended values: `low`, `medium`, `high`, `critical`. |
| `affected_rows_count` | integer | Optional | Number of rows affected by issue. |
| `affected_person_count` | integer | Optional | Number of persons affected by issue. |
| `subgroup_variable` | string | Optional | Subgroup variable for stratified validation, such as sex, race, age band, study, site, or geography. |
| `subgroup_value` | string | Optional | Subgroup value. |
| `validation_reference_standard` | string | Optional | Reference standard, such as chart review, study adjudication, manual abstraction, or established algorithm. |
| `sensitivity` | numeric | Optional | Sensitivity if validation design supports it. |
| `specificity` | numeric | Optional | Specificity if validation design supports it. |
| `positive_predictive_value` | numeric | Optional | PPV if validation design supports it. |
| `negative_predictive_value` | numeric | Optional | NPV if validation design supports it. |
| `f1_score` | numeric | Optional | F1 score for classification/extraction algorithms. |
| `inter_rater_reliability` | numeric | Optional | Kappa, ICC, or other inter-rater reliability metric if manual review was used. |
| `remediation_action` | string | Recommended | Corrective action taken or planned. |
| `validation_date` | date | Yes | Date validation check was performed. |
| `reviewer` | string | Optional | Reviewer role, team, or pseudonymized reviewer identifier. |
| `notes` | string | Optional | Additional comments. |

### Notes

- Validation should include overall checks and subgroup checks to detect differential quality by study, site, race/ethnicity, sex, age, language, Medicare coverage type, and other relevant dimensions.
- The file should be updated with each annual release.

---

## 4.12 `research_use_case_cohort.csv`

### Purpose

`research_use_case_cohort.csv` defines analytic cohorts for specific AD/ADRD use cases. The RFA requires stakeholder-driven research questions and use cases. This file allows the core data resource to support multiple analyses without changing the underlying person, enrollment, claims, EHR, study, and CDE files.

Examples of use cases include incident AD/ADRD identification, care transitions after diagnosis, medication patterns, caregiver outcomes, health disparities, SDOH and utilization, biomarker-confirmed disease trajectories, or multimorbidity progression.

### Grain

One row per person per use case. A person may appear in multiple use cases.

### Primary key

`use_case_id`, `person_id`

### Column specification

| Column | Type | Required | Description |
|---|---:|---:|---|
| `use_case_id` | string | Yes | Stable identifier for the research use case. |
| `use_case_name` | string | Yes | Human-readable use case name. |
| `principal_use_case_question` | string | Recommended | Main research question the cohort supports. |
| `person_id` | string | Yes | Project-wide pseudonymized person identifier. |
| `cohort_name` | string | Yes | Name of analytic cohort. |
| `cohort_role` | string | Recommended | Role in cohort, such as `case`, `control`, `exposed`, `unexposed`, `caregiver`, `comparison`. |
| `cohort_entry_date` | date | Recommended/Restricted | Date the person enters the use-case cohort. May be converted to month/relative time. |
| `cohort_entry_month` | date | Recommended | First day of cohort entry month. |
| `cohort_exit_date` | date | Recommended/Restricted | Date the person exits the use-case cohort. |
| `cohort_exit_month` | date | Recommended | First day of cohort exit month. |
| `index_event_type` | string | Yes | Use-case-specific index event definition. |
| `index_date` | date | Recommended/Restricted | Use-case-specific index date. May differ from `person_master.csv` index date. |
| `index_month` | date | Recommended | First day of use-case-specific index month. |
| `inclusion_criteria_met` | string | Yes | Pipe-delimited list or JSON summary of inclusion criteria met. |
| `exclusion_criteria_met` | string | No | Pipe-delimited list or JSON summary of exclusion criteria met. |
| `exclusion_flag` | boolean | Yes | Indicates whether the person is excluded from the use-case analytic cohort. |
| `exclusion_reason` | string | No | Primary reason for exclusion. |
| `baseline_period_start` | date | Recommended | Start of use-case-specific baseline period. |
| `baseline_period_end` | date | Recommended | End of use-case-specific baseline period. |
| `followup_period_start` | date | Recommended | Start of follow-up for the use case. |
| `followup_period_end` | date | Recommended | End of follow-up for the use case. |
| `required_observability_rule` | string | Recommended | Required observability, such as 12 months Part A/B FFS baseline plus Part D follow-up. |
| `baseline_observability_met_flag` | boolean | Recommended | Indicates baseline observability requirements are met. |
| `followup_observability_met_flag` | boolean | Recommended | Indicates follow-up observability requirements are met. |
| `censoring_date` | date | Recommended/Restricted | Date of censoring for the use case. |
| `censoring_reason` | string | Recommended | Reason for censoring, such as death, disenrollment, end of data, event occurred, loss to follow-up, or study end. |
| `treatment_group` | string | Optional | Exposure/treatment group for comparative studies. |
| `comparator_group` | string | Optional | Comparator group label. |
| `outcome_window_start` | date | Optional | Start of outcome ascertainment window. |
| `outcome_window_end` | date | Optional | End of outcome ascertainment window. |
| `primary_outcome_cde_id` | string | Optional | CDE ID for primary outcome. |
| `primary_exposure_cde_id` | string | Optional | CDE ID for primary exposure. |
| `analysis_weight` | numeric | Optional | Analysis weight, survey weight, inverse probability weight, or matching weight. |
| `propensity_score` | numeric | Optional | Propensity score if relevant to the use case. |
| `matched_set_id` | string | Optional | Matched set identifier for matched analyses. |
| `use_case_version` | string | Yes | Version of use-case cohort definition. |
| `created_date` | date | Yes | Date cohort row was created. |
| `notes` | string | Optional | Additional comments. |

### Notes

- Keep use-case logic separate from the core longitudinal files to avoid hard-coding one study design into the shared resource.
- Every use-case cohort should have a corresponding protocol, code file, and validation check.

---

# 5. Optional restricted event-level files

The person-month files are designed for analysis and sharing, but restricted event-level files are often necessary for reproducibility, debugging, and more advanced methods. These files should usually remain inside the secure enclave.

## 5.1 `cms_claim_event.csv`

### Purpose

Stores one row per CMS claim or stay-level event after basic normalization and deduplication. Supports audit trails for `claims_person_month.csv`.

### Key columns

| Column | Type | Description |
|---|---:|---|
| `person_id` | string | Pseudonymized person identifier. |
| `claim_event_id` | string | Pseudonymized claim or derived stay identifier. |
| `source_file` | string | CMS source file family, such as inpatient, outpatient, carrier, SNF, HHA, hospice, DME, PDE, or OASIS. |
| `claim_from_date` | date | Claim start date or service date. Restricted as needed. |
| `claim_through_date` | date | Claim end date. Restricted as needed. |
| `event_month` | date | First day of event month. |
| `claim_type` | string | Normalized claim type. |
| `provider_type` | string | Provider type or specialty if available. |
| `diagnosis_codes` | string | Diagnosis codes or reference to claim diagnosis child table. |
| `procedure_codes` | string | Procedure codes or reference to claim procedure child table. |
| `paid_amount` | numeric | Paid amount using project-defined cost variable. |
| `deduplication_group_id` | string | Grouping ID used to prevent double counting. |
| `mapped_cde_ids` | string | CDEs triggered by this event, if applicable. |
| `algorithm_version` | string | Version of normalization algorithm. |

## 5.2 `part_d_fill_event.csv`

### Purpose

Stores one row per prescription drug event/fill. Supports medication exposure, treatment, and polypharmacy CDEs.

### Key columns

| Column | Type | Description |
|---|---:|---|
| `person_id` | string | Pseudonymized person identifier. |
| `fill_event_id` | string | Pseudonymized fill identifier. |
| `fill_date` | date | Prescription fill date. Restricted as needed. |
| `fill_month` | date | First day of fill month. |
| `ndc` | string | National Drug Code. |
| `rxnorm_ingredient` | string | Mapped RxNorm ingredient. |
| `drug_class` | string | Harmonized drug class. |
| `days_supply` | integer | Days supply. |
| `quantity_dispensed` | numeric | Quantity dispensed. |
| `gross_drug_cost` | numeric | Drug cost measure, if available. |
| `exposure_start_date` | date | Start of derived exposure interval. |
| `exposure_end_date` | date | End of derived exposure interval. |
| `mapped_cde_ids` | string | CDEs triggered by the fill. |
| `algorithm_version` | string | Version of medication mapping algorithm. |

## 5.3 `ehr_note_nlp_extract_event.csv`

### Purpose

Stores one row per NLP-extracted concept from a clinical note. Supports validation and traceability for NLP-derived CDEs.

### Key columns

| Column | Type | Description |
|---|---:|---|
| `person_id` | string | Pseudonymized person identifier. |
| `note_extract_id` | string | Unique extract identifier. |
| `encounter_id_hash` | string | Hashed encounter identifier. |
| `note_date` | date | Note date, restricted as needed. |
| `note_month` | date | First day of note month. |
| `note_type` | string | Note type, such as progress note, discharge summary, social work note. |
| `nlp_concept` | string | Extracted concept, such as cognitive decline, caregiver burden, food insecurity. |
| `assertion_status` | string | Whether concept is present, absent, hypothetical, historical, family history, or uncertain. |
| `temporality` | string | Current, past, future, or unclear. |
| `confidence_score` | numeric | NLP confidence score. |
| `mapped_cde_id` | string | Target CDE ID. |
| `manual_review_status` | string | Validation status for this extract or sample. |
| `algorithm_id` | string | NLP algorithm ID. |
| `algorithm_version` | string | NLP algorithm version. |

---

# 6. Recommended CDE domain implementation examples

The following examples show how the architecture can represent the RFA's seven required domains.

| RFA domain | Example CDEs | Recommended source files | Recommended time grain |
|---|---|---|---|
| Patient and caregiver demographics | age band, sex, race, ethnicity, caregiver relationship, caregiver availability | `person_master.csv`, `nia_study_visit_wave.csv`, EHR registration data | person, baseline, study visit |
| Disease characterization | AD/ADRD status, incident AD/ADRD, MCI, MCC count, specific chronic condition indicators | `claims_person_month.csv`, `ehr_person_month.csv`, `nia_study_visit_wave.csv` | person-month, baseline, study visit |
| Health assessment | cognitive score, ADL limitations, IADL limitations, falls, neuropsychiatric symptoms | NIA study visits, EHR structured assessments, EHR NLP, OASIS | study visit, person-month |
| Biomarkers/genetics/genomics | amyloid status, tau status, APOE e4 status, biomarker availability | NIA study files, EHR labs, specialty biomarker repositories | study visit, event, person |
| Treatment for all conditions | anti-dementia medication, antipsychotics, antidepressants, polypharmacy, procedures, care management | Part D, EHR medications, claims procedures | person-month |
| Patient and caregiver outcomes | hospitalization, ED use, SNF transition, hospice, mortality, quality of life, caregiver burden | CMS claims, EHR, NIA study/caregiver interviews | person-month, study visit |
| SDOH | education, financial strain, food insecurity, housing instability, transportation need, living arrangement | NIA study, EHR structured screening, EHR NLP | baseline, study visit, person-month |

---

# 7. Recommended workflow to create the files

1. **Create secure crosswalks.** Link NIA study participants to CMS and EHR identities inside approved secure infrastructure.
2. **Create `person_master.csv`.** Define the eligible person population and baseline index metadata.
3. **Create the person-month skeleton.** Generate one row per person-month covering baseline and follow-up windows.
4. **Add enrollment and observability.** Use CMS denominator/enrollment files and EHR/study availability indicators to produce `person_month_enrollment_observability.csv`.
5. **Normalize and deduplicate claims.** Convert claims and claim lines into consistent event types; prevent double counting.
6. **Aggregate claims by month.** Produce `claims_person_month.csv`.
7. **Extract EHR structured data and NLP concepts.** Produce `ehr_person_month.csv` and restricted NLP event files if needed.
8. **Prepare NIA study visit/wave file.** Preserve original study timing and harmonize study variables where possible.
9. **Define CDEs and mappings.** Populate `cde_codebook.csv` and `source_to_cde_mapping.csv`.
10. **Generate CDE values.** Produce `harmonized_cde_person_time.csv` using documented source precedence and algorithms.
11. **Validate.** Run schema, observability, linkage, phenotype, CDE, AI/ML/NLP, and privacy checks; store results in `quality_validation_summary.csv`.
12. **Create use-case cohorts.** Define use-case-specific cohorts in `research_use_case_cohort.csv`.
13. **Release and maintain.** Publish code, CDE definitions, documentation, validation summaries, and approved data products; update annually.

---

# 8. Recommended file relationships

```text
person_master.csv
  |-- person_month_enrollment_observability.csv
  |     |-- claims_person_month.csv
  |     |-- ehr_person_month.csv
  |
  |-- nia_study_visit_wave.csv
  |
  |-- harmonized_cde_person_time.csv
          |-- cde_codebook.csv
          |-- source_to_cde_mapping.csv
          |-- algorithm_provenance.csv
          |-- quality_validation_summary.csv

research_use_case_cohort.csv links to person_master.csv and selected CDEs.
dataset_manifest.csv describes every released file.
```

---

# 9. Analysis-ready outputs derived from this architecture

The architecture can support several analysis-ready datasets without changing the core files.

| Analysis output | Description | Source files |
|---|---|---|
| Person-month wide file | One row per person-month with selected CDEs pivoted into columns. | `person_month_enrollment_observability.csv`, `harmonized_cde_person_time.csv` |
| Baseline covariate file | One row per person with baseline demographics, comorbidities, utilization, SDOH, and study measures. | `person_master.csv`, `claims_person_month.csv`, `ehr_person_month.csv`, `nia_study_visit_wave.csv` |
| Survival/time-to-event file | One row per person with index date, event date, censoring date, and covariates. | `research_use_case_cohort.csv`, `harmonized_cde_person_time.csv` |
| Study-visit longitudinal file | One row per person per NIA visit with linked claims/EHR summaries in surrounding windows. | `nia_study_visit_wave.csv`, `claims_person_month.csv`, `ehr_person_month.csv` |
| Caregiver longitudinal file | One row per caregiver/person/time with caregiver burden, relationship, care hours, and patient outcomes. | `person_master.csv`, `nia_study_visit_wave.csv`, `harmonized_cde_person_time.csv` |
| SDOH trajectory file | One row per person-time with education, financial strain, living arrangement, social risks, and outcomes. | `ehr_person_month.csv`, `nia_study_visit_wave.csv`, `harmonized_cde_person_time.csv` |

---

# 10. Quality-control expectations

At minimum, each release should validate:

1. **Schema:** Required columns exist and data types are correct.
2. **Primary keys:** No duplicate primary key rows.
3. **Foreign keys:** All child-file `person_id` values exist in `person_master.csv`.
4. **Longitudinal continuity:** Person-month rows cover expected baseline and follow-up windows.
5. **Observability:** Claims, drug, EHR, and study-derived CDEs are not interpreted without relevant observability.
6. **CMS logic:** Medicare Advantage, Part A, Part B, Part D, dual eligibility, death, and disenrollment are handled correctly.
7. **Claim deduplication:** Event counts do not double-count claim lines, split claims, or transfers.
8. **Coding systems:** ICD-9-CM, ICD-10-CM, CPT, HCPCS, NDC, RxNorm, LOINC, SNOMED CT, and study item mappings are valid for the applicable dates.
9. **AI/ML/NLP validation:** Model-derived CDEs include confidence, validation status, and manual validation where needed.
10. **Privacy:** Direct identifiers are absent from shared files; small cells and restricted fields are suppressed or generalized.
11. **Reproducibility:** Each output links to a pipeline version, code commit, and algorithm provenance record.

---

# 11. Recommended privacy and access levels

| Access level | Contents | Example files |
|---|---|---|
| Public metadata | README, CDE definitions, codebook, synthetic examples, mapping summaries, validation summaries without sensitive counts | `README`, `cde_codebook.csv`, public `dataset_manifest.csv` |
| Controlled access | Deidentified or limited person-level analytic files approved for qualified researchers | Person-month CDE files, use-case cohorts |
| Enclave only | Files with restricted dates, linkage keys, claim IDs, note-derived extracts, granular geography, or sensitive biomarker/genetic data | Event-level claims, NLP extracts, crosswalks |
| Synthetic only | Simulated data with the same schema for training and code testing | Synthetic versions of all analytic files |

---

# 12. Recommended naming and versioning

### File naming

Use consistent names:

```text
<file_id>_<short_name>_<release_version>.<extension>
```

Example:

```text
01_person_master_2026Q1.csv
02_person_month_enrollment_observability_2026Q1.csv
06_harmonized_cde_person_time_2026Q1.parquet
```

### Versioning

Use semantic versioning for file logic and release versions for data refreshes:

```text
file_version: v1.2.0
algorithm_version: adrd_claims_phenotype_v2.1.0
data_product_release: 2026Q1_release
```

### Annual updates

Each annual update should document:

- newly added source data;
- changed source coding systems or source schemas;
- new, revised, deprecated, or retired CDEs;
- changed algorithms or value sets;
- validation results;
- known limitations;
- backward compatibility notes.

---

# 13. Minimal viable release

A minimal first-year release could include:

1. `dataset_manifest.csv`
2. `person_master.csv`
3. `person_month_enrollment_observability.csv`
4. `claims_person_month.csv`
5. `nia_study_visit_wave.csv`
6. `harmonized_cde_person_time.csv`
7. `cde_codebook.csv`
8. `source_to_cde_mapping.csv`
9. `algorithm_provenance.csv`
10. `quality_validation_summary.csv`
11. code to create the longitudinal CMS CCW files
12. synthetic test data with the same schema

`ehr_person_month.csv` should be included as soon as EHR data access and harmonization are operational, because EHR data are central to health assessment, SDOH, caregiver, and NLP-derived CDEs.

---

# 14. Example CDE rows

Example rows for `harmonized_cde_person_time.csv`:

| person_id | time_id | time_grain | months_from_index | cde_id | cde_name | cde_domain | cde_value | source_system | derived_flag | validation_status |
|---|---|---|---:|---|---|---|---|---|---:|---|
| P000001 | index_m000 | person_month | 0 | ADRD_DX_001 | ad_adrd_status | Disease characterization | prevalent_ad_adrd | COMBINED_ALGORITHM | true | validated |
| P000001 | index_m006 | person_month | 6 | TREAT_RX_002 | anti_dementia_drug_fill | Treatment for all conditions | OBSERVED_YES | CMS_CCW | true | not_required |
| P000001 | visit_003 | study_visit | 14 | HEALTH_COG_001 | cognitive_global_score | Health assessment | 23 | NIA_STUDY | false | expert_adjudicated |
| P000001 | visit_003 | study_visit | 14 | SDOH_LIVE_001 | living_arrangement | Social determinants of health | with_family | NIA_STUDY | false | not_required |
| P000001 | index_m012 | person_month | 12 | OUTCOME_HOSP_001 | any_inpatient_hospitalization | Patient and caregiver health outcomes | OBSERVED_NO | CMS_CCW | true | not_required |

---

# 15. Recommended implementation notes for NIH/CMS enclave work

1. Create and store linkage crosswalks separately from analytic files.
2. Never export raw CMS beneficiary IDs, medical record numbers, direct identifiers, or raw note text unless explicitly permitted under the data use agreement.
3. Build person-month skeletons using approved date handling rules.
4. Keep exact dates in enclave-only files when required; use relative months in shared analytic files.
5. Treat Medicare Advantage months carefully; FFS claims may be incomplete unless valid encounter data are included.
6. Require Part D observability for claims-based prescription drug measures.
7. Separate event-level files from analytic person-month summaries.
8. Preserve original NIA study visit structure rather than forcing all study variables into monthly rows.
9. Use `missingness_code` to distinguish no observed event from no data.
10. Maintain all code, mapping files, algorithm metadata, and validation outputs with each release.

---

# 16. Suggested directory structure

```text
project_root/
  README_longitudinal_file_architecture.md
  data_dictionary/
    dataset_manifest_schema.csv
    person_master_schema.csv
    person_month_enrollment_observability_schema.csv
    claims_person_month_schema.csv
    ehr_person_month_schema.csv
    nia_study_visit_wave_schema.csv
    harmonized_cde_person_time_schema.csv
    cde_codebook_schema.csv
    source_to_cde_mapping_schema.csv
    algorithm_provenance_schema.csv
    quality_validation_summary_schema.csv
    research_use_case_cohort_schema.csv
  value_sets/
    ad_adrd_icd_codes.csv
    mcc_condition_codes.csv
    medication_class_ndc_rxnorm.csv
    cognitive_testing_cpt_hcpcs.csv
    sdoh_screening_mappings.csv
  code/
    01_create_person_master.sql
    02_create_person_month_observability.sql
    03_aggregate_cms_claims_person_month.sql
    04_extract_ehr_person_month.sql
    05_harmonize_nia_study_visits.R
    06_generate_cde_person_time.py
    07_validate_release.py
  synthetic_data/
    person_master_synthetic.csv
    person_month_enrollment_observability_synthetic.csv
    claims_person_month_synthetic.csv
    ehr_person_month_synthetic.csv
    nia_study_visit_wave_synthetic.csv
    harmonized_cde_person_time_synthetic.csv
  validation/
    quality_validation_summary.csv
    validation_protocol.md
  docs/
    cde_user_guide.md
    cms_ccw_longitudinal_file_creation_guide.md
    ehr_nlp_validation_guide.md
    release_notes.md
```

---

# 17. Summary

The recommended longitudinal file architecture for this project is a linked suite of files centered on a person-month observability skeleton and a long-format harmonized CDE file. CMS CCW claims support longitudinal diagnosis, treatment, utilization, cost, and outcomes measurement. EHR data support clinical assessments, SDOH, caregiver information, disease severity, and NLP-derived variables. NIA study visits preserve high-quality longitudinal research measures. CDE, mapping, provenance, and validation files make the resource reusable, auditable, and maintainable across annual releases.

