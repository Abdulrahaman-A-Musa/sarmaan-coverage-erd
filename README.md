# SARMAAN Coverage – Physical Data Model (ERD)

Interactive physical entity-relationship diagram for the **SARMAAN Coverage survey** tables in the Databricks catalog `sarmaan2data`.

**Live page:** https://YOUR-USERNAME.github.io/sarmaan-coverage-erd/

The page shows every table in the Coverage pipeline with all of its columns, data types, primary and foreign keys, and the relationships between tables, for each layer of the lakehouse (bronze, silver, gold).

---

## Contents

1. [How data flows](#how-data-flows)
2. [What is in each layer](#what-is-in-each-layer)
3. [What changes from bronze to silver](#what-changes-from-bronze-to-silver)
4. [What changes from silver to gold](#what-changes-from-silver-to-gold)
5. [Tables and column counts](#tables-and-column-counts)
6. [Keys and relationships](#keys-and-relationships)
7. [Using the ERD page](#using-the-erd-page)
8. [Updating the page](#updating-the-page)

---

## How data flows

```
KoboToolbox  ──►  bronze  ──►  silver  ──►  gold  ──►  analysis and dashboards
 (survey)        (step 1)   (steps 2, 4)  (step 5)
```

| Layer | Filled by | In one line | Replaces |
|---|---|---|---|
| `bronze` | Step 1 | Raw copy of every Kobo submission | Postgres `raw_data` |
| `silver` | Steps 2 and 4 | Cleaned, renamed and keyed, with every quality issue logged | – (new) |
| `gold` | Step 5 | Approved, analysis-ready data with proper data types | Postgres `sarmaan2data` |

**Who should use which layer**

- **Analysts, dashboards and reports:** use **gold**.
- **Data quality follow-up with field teams:** use **silver**, especially `coverage_validation_issues`.
- **Audit, or checking exactly what Kobo sent:** use **bronze**.

---

## What is in each layer

### Bronze: raw

- **Every submission, approved or not**, exactly as downloaded from Kobo.
- **Every column is `STRING`**, so nothing is lost or converted on the way in.
- **Kobo's own export column names** are kept, e.g. `starttime`, `start_geopoint_latitude`, `_submission__uuid`, `_submission__validation_status`.
- **All form helper fields** are kept: notes, error flags, confirmation questions and Kobo metadata (`deviceid`, `submitted_by`, `version`, `tags`, `meta_rootuuid`).
- **Tables link on Kobo's internal IDs**: `index_uuid` on the household and `_parent_index_submission__uuid` on the child tables.

### Silver: cleaned and keyed

- **Every submission, approved or not**, still stored as `STRING`.
- **Columns are renamed to the questionnaire codes** from the step 2 mapping file, e.g. `q1_States`, `q23_has_electricity`, `q90_Offered_drugs`.
- **New shared keys** link the household to its child tables:
  - `concatenated_id` on every table.
  - One unique ID per child table row, e.g. `child_id_child_uuid` or `net_id_net_uuid`.
- **Location fields are copied onto every child table** (`q1_States`, `q2_LGAs`, `q3_Wards`, `q4_Community`, `q6_HH_number`, `unique_code`), so each child table can be filtered by location on its own.
- **A `cycle` column** on the household table records the survey round.
- **A new table, `coverage_validation_issues`**, logs every failed check from step 4. Each row records the run (`run_id`, `logged_at`), where the problem is (`sheet`, `row_index`, `row_id`), which check failed (`check_type`, `rule`), and what was wrong (`column_name`, `value`, `issue`).

### Gold: approved and analysis-ready

- **Approved submissions only.** Anything not yet approved stays in silver.
- **Plain-English column names**, e.g. `state`, `electricity`, `child_drug`.
- **Proper data types**: `DATE`, `TIMESTAMP_NTZ`, `INT`, `SMALLINT`, `BIGINT`, `DOUBLE` and `STRING`.
- **Yes/no answers stored as 0/1 in the raw data are shown as no/yes.**
- **Kobo metadata and form helper fields are removed**, leaving only survey answers, location and keys.
- **The same keys as silver** (`concatenated_id` and the per-table unique IDs), so silver and gold rows can be traced to each other.

---

## What changes from bronze to silver

| Change | Example |
|---|---|
| **Renamed** from Kobo export names to questionnaire codes | `starttime` → `start`; `start_geopoint` → `gps_location` |
| **Keys added** | `concatenated_id`; `child_id_childd_uuid`, `net_id_net_uuid`, `child_id_child_uuid` |
| **Location fields copied** onto the child tables | `q1_States`, `q2_LGAs`, `q3_Wards`, `q4_Community`, `q6_HH_number`, `unique_code` |
| **Added** | `cycle` (household); `coverage_validation_issues` (new table) |
| **Removed** | Kobo's `meta_rootuuid` |

**Net effect:** 458 → 482 columns (+24), and 4 → 5 tables.

---

## What changes from silver to gold

Most silver columns are **renamed** in gold, not removed. Gold drops 84 columns in total.

### Renamed: questionnaire codes become plain names

| Silver | Gold |
|---|---|
| `q1_States`, `q2_LGAs`, `q3_Wards`, `q4_Community` | `state`, `lga`, `ward`, `community` |
| `q6_HH_number`, `unique_code` | `household_no`, `household_code` |
| `q23_has_electricity`, `q24_own_radio` | `electricity`, `radio` |
| `q79_Have_mosquitonet`, `q80_Howmany_mosquitonet` | `household_nets`, `nets_count` |
| `q82_Howgot_net`, `q84_sleepunder_net` | `net_source`, `net_sleep` |
| `q88_Ageof_child`, `q89_Sexof_child`, `q90_Offered_drugs` | `child_age`, `child_gender`, `child_drug` |
| `q104a_KnowAZM_Radio` | `mda_awareness_radio` |
| `q137_willingspend_Malaria` | `malaria_prevention_cost` |

### Kept with the same name

| Table | Columns |
|---|---|
| `coverage_household` | `start`, `end`, `consent`, `noconsent_reasons`, `othernoconsent_reasons`, `total_eligible`, `no_eligible`, `cycle`, `formatted_date`, `concatenated_id`, `index` |
| All child tables | `concatenated_id` and the table's unique ID |

### Removed: not needed for analysis

| Group | Silver columns removed | Tables |
|---|---|---|
| Kobo submission metadata | `submissiontime`, `validationstatus`, `notes`, `status`, `submittedby`, `version`, `tags`, `id` | All four |
| Parent table name | `_parent_table_name` / `parenttablename` | Child tables |
| Device and login | `deviceid`, `concat_user`, `error_invalid_login` | Household |
| Form error flags | `error_missing_name`, `error_missing_phone`, `error_wrong_date_time` | Household |
| Confirmation questions | `confirm_user_info`, `confirm_Data_Collector_info`, `confirm_lga`, `confirm_ward`, `confirm_community`, `check_location` | Household |
| Form notes and section markers | `note1`, `note2`, `note3`, `note1_001`, `note7`, `note8`, `notezz`, `note1p`, `end_interview_note`, `ac_module`, `e_children` | Household |
| Extra GPS fields | `gps_location`, `_gps_latitude`, `gps_longitude`, `gps_altitude`, `gps_precision`, `q9_GPS`, `q9_GPS_Altitude`, `q9_GPS_Precision` | Household |

Gold keeps one GPS point per household, as `latitude` and `longitude`.

### Column change per table

| Table | Silver | Gold | Removed |
|---|---:|---:|---:|
| `coverage_household` | 309 | 265 | 44 |
| `coverage_children_1_59` | 94 | 83 | 11 |
| `coverage_net_info` | 44 | 35 | 9 |
| `coverage_all_children` | 25 | 15 | 10 |
| `coverage_validation_issues` | 10 | – | stays in silver |
| **Total** | **482** | **398** | **84** |

The full column-by-column mapping, silver name to gold name, is in the step 5 mapping file (`step_5_map_file.xlsx`).

---

## Tables and column counts

| Table | What one row is | Bronze | Silver | Gold |
|---|---|---:|---:|---:|
| `coverage_household` | One household visit | 311 | 309 | 265 |
| `coverage_children_1_59` | One child aged 1–59 months, with drug offer and uptake | 88 | 94 | 83 |
| `coverage_net_info` | One mosquito net in the household | 39 | 44 | 35 |
| `coverage_all_children` | One child listed in the household | 20 | 25 | 15 |
| `coverage_validation_issues` | One failed data quality check | – | 10 | – |
| **Total** | | **458** | **482** | **398** |

---

## Keys and relationships

`coverage_household` is the parent table. Each child table links back to it with a foreign key. One household can have zero or many rows in each child table.

```
coverage_household (1) ──< coverage_all_children   (0..many)
                   (1) ──< coverage_children_1_59  (0..many)
                   (1) ──< coverage_net_info       (0..many)

coverage_validation_issues   (silver only, standalone log, no keys)
```

**Silver and gold**

| Table | Primary key | Foreign key → parent |
|---|---|---|
| `coverage_household` | `concatenated_id` | – |
| `coverage_all_children` | `child_id_childd_uuid` | `concatenated_id` → `coverage_household.concatenated_id` |
| `coverage_net_info` | `net_id_net_uuid` | `concatenated_id` → `coverage_household.concatenated_id` |
| `coverage_children_1_59` | `child_id_child_uuid` | `concatenated_id` → `coverage_household.concatenated_id` |

**Bronze**

| Table | Primary key | Foreign key → parent |
|---|---|---|
| `coverage_household` | `index_uuid` | – |
| `coverage_all_children` | `child_id_submission__uuid` | `_parent_index_submission__uuid` → `coverage_household.index_uuid` |
| `coverage_net_info` | `net_id_submission__uuid` | `_parent_index_submission__uuid` → `coverage_household.index_uuid` |
| `coverage_children_1_59` | `child_idd_submission__uuid` | `_parent_index_submission__uuid` → `coverage_household.index_uuid` |

Primary and foreign keys are declared in Databricks as informational constraints. The engine does not enforce them; the loader's validation checks do.

---

## Using the ERD page

- **Schema switch:** view the `bronze`, `silver` or `gold` model.
- **Entity boxes:** key columns are listed first, then every other column with its position in the table and its data type.
- **Relationship lines:** crow's-foot notation, one household to zero or many child rows.
- **Search:** find a column across all tables, e.g. `uuid`, `mda_` or `gps`.
- **Expand every table:** shows full column lists without scrolling.

The page is one self-contained `index.html` with no external data files. It also works offline: download it and double-click.

---

## Updating the page

1. Regenerate the column data from the mapping files with `coverage/docs/generate_erd.py` (step 1, 2 and 5 map files).
2. Rebuild `index.html` with the new data.
3. In this repository, click **Add file → Upload files**, upload the new `index.html` and commit. GitHub Pages republishes within a minute or two, and the link stays the same.
4. If column counts or keys changed, update the tables in this README.

---

## Repository contents

| File | Purpose |
|---|---|
| `index.html` | The interactive ERD page served by GitHub Pages |
| `README.md` | This file |

---

## Note on visibility

This repository is public so it can be served by free GitHub Pages. It contains table names, column names, data types and keys only, with **no survey responses or personal data**.

*Maintained by: [Your name], SARMAAN programme data team*
