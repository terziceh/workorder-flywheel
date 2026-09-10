# 05 — Silver Work Order Transformations

## Objective

Transform the validated Bronze work-order snapshot into a cleaner, analysis-ready Silver dataset while preserving the historical maintenance information needed for downstream analytics and machine learning.

Silver focuses on business-data cleaning and validation. Model-specific text processing, keyword extraction, feature engineering, and prediction logic are intentionally deferred to Gold.

Related issue: [#5 — Build the Silver cleaning and data-quality pipeline](https://github.com/terziceh/workorder-flywheel/issues/5).

## Why Silver Exists

Bronze preserves the historical source data with minimal transformation. Silver is where the first business-facing cleaning occurs.

```text
Bronze
Raw + traceable
        ↓
Silver
Clean + standardized + validated
        ↓
Gold
Analytics + model-ready datasets
```

## Input

The Silver pipeline reads the validated Bronze Delta table. The historical dataset includes work-order and phase information such as identifiers, descriptions, creation dates, facility and property context, problem and category information, shops, work codes, locations, assets, equipment, failure codes, and ingestion metadata.

Operational records remain private and are not published in this repository.

## Transformation Strategy

The Silver pipeline intentionally uses simple native Spark transformations. The first implementation performs the following steps:

1. Rename ambiguous source fields.
2. Trim unnecessary whitespace.
3. Convert blank strings to null.
4. Preserve identifiers as strings.
5. Standardize selected categorical fields.
6. Preserve maintenance descriptions.
7. Parse the work-order creation timestamp.
8. Create data-quality flags.
9. Identify and remove confirmed exact duplicate exports.
10. Write and validate the Silver Delta table.

The pipeline avoids Python UDFs and model-specific transformations so the cleaning process remains simple and scalable.

## 1. Clarify Source Column Names

Bronze validation identified several source-export fields whose names did not clearly describe their business meaning. Silver assigns descriptive names for the work-order and phase fields.

Example:

```text
description1  -> work_order_description
description14 -> phase_description
shop12        -> work_order_shop
shop16        -> phase_shop
```

The rename changes field names only. Source values are not modified.

## 2. Normalize Missing Values and Whitespace

String fields are trimmed to remove unnecessary surrounding whitespace. Empty or whitespace-only values are converted to null.

This creates a consistent representation of missing information without inventing replacement values. Optional fields remain nullable, so missing asset, equipment, failure-code, or other supplemental information does not automatically cause a record to be removed.

## 3. Preserve Business Identifiers

Identifiers such as work order, phase, property, location, asset, and equipment are preserved as text. This prevents identifier values from being unintentionally treated as numeric measurements and protects formatting such as leading zeros.

## 4. Standardize Business Labels

Selected categorical fields are trimmed and standardized to consistent casing. These include status, region, facility, problem code, type, category, job priority, organization, work-order shop, phase shop, work code, and failure code.

Silver does not attempt to determine whether a historical work code or other business label is correct. Historical label validation is a downstream analytical and modeling problem.

## 5. Preserve Maintenance Descriptions

Work-order and phase descriptions are intentionally not aggressively cleaned. Maintenance descriptions naturally contain abbreviations, equipment terminology, spelling variation, technician language, room and location references, operational context, punctuation, and formatting differences.

That information may be useful for future work-code and asset prediction. Silver therefore performs only basic whitespace normalization.

NLP transformations such as lowercasing, punctuation removal, stop-word processing, keyword extraction, TF-IDF preparation, and `model_text` creation are deferred to Gold.

## 6. Parse Creation Dates

The source creation-date field is converted into a Spark timestamp while the source value remains available during validation. A separate flag identifies populated source values that fail timestamp conversion.

This allows downstream analysis to use a proper timestamp without silently hiding invalid source values.

## 7. Data-Quality Flags

Silver creates explicit quality indicators for important conditions such as missing work-order identifiers, missing phases, missing descriptions, missing work codes, failed creation-date parsing, and exact repeated records.

Missing work codes are retained because they may represent useful inference cases for the future recommendation system. Missing asset values are also retained because asset identification is one of the primary downstream modeling goals.

## 8. Duplicate Handling

Bronze validation identified a small set of exact repeated source records. These records were reviewed separately before Silver transformation.

Silver does not deduplicate on work-order number because one work order may legitimately contain multiple phases. Instead, duplicate detection compares the complete business record while excluding ingestion metadata.

After the identified repeats were confirmed as duplicate export records, Silver retains one copy of each business record and removes only the additional exact copies. This preserves legitimate multi-phase work orders while removing confirmed repeated exports.

## 9. Silver Validation

Before writing the final table, the pipeline reviews overall quality-flag counts, individual flags, work-code coverage, asset coverage, equipment coverage, location coverage, failed date conversions, and confirmed exact repeats.

After duplicate removal, the dataset is checked again to confirm that no exact duplicate business groups remain. The final Silver table is then written as Delta and its saved record count is validated against the transformed DataFrame.

## Silver Output

The first implementation produces one primary cleaned table:

```text
work_order_ai.silver.workorders_clean
```

Conceptually:

```text
Historical Work Orders
        ↓
Bronze
Raw + lineage
        ↓
Validation
Missingness + grain + duplicates + dates
        ↓
Silver
Clean business data
        ↓
Gold
Analytics + ML features
```

## What Silver Does Not Do

The following are intentionally deferred:

- work-code correction
- asset correction
- keyword extraction
- `model_text` creation
- TF-IDF transformation
- embeddings
- model features
- training/test datasets
- work-code prediction
- asset prediction
- automated label correction

These transformations depend on the analytical or modeling objective and therefore belong in Gold or the ML layer.

## Next Step — Gold

Gold will convert the cleaned Silver data into datasets designed for specific analytical and machine-learning use cases.

The first Gold work will investigate relationships between descriptions, work codes, facility/location context, and asset history. A model-specific text field can be created in Gold while preserving the original Silver descriptions.

For example:

```text
Original description
"ROOM IS WARM AND AIR HANDLER IS MAKING NOISE..."

        ↓

model_text
"warm air handler noise"
```

This separation keeps Silver reusable while allowing Gold datasets to evolve with the modeling strategy.

## Current Limitations

The first Silver implementation processes the current historical snapshot using a full-refresh workflow.

The following production capabilities are intentionally deferred:

- incremental ingestion
- automated recurring-file processing
- separate quarantine tables
- formal reference-data validation
- reusable pipeline packaging
- automated data-quality test suites
- orchestration and monitoring

These capabilities can be added if the project moves from proof of concept to a recurring production workflow.
