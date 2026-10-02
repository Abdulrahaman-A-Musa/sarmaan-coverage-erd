SARMAAN Coverage – Physical Data Model (ERD)

Interactive physical entity-relationship diagram for the SARMAAN Coverage survey tables in the Databricks catalog sarmaan2data.

The page shows every table in the Coverage pipeline with all of its columns, data types, primary and foreign keys, and the relationships between tables, for each layer of the lakehouse (bronze, silver, gold).

What the page shows
Entity boxes: one per table. Key columns come first, then every other column with its position in the table and its data type.
Relationships: crow's-foot notation. One household links to zero or many rows in each child table.
Schema switch: view the bronze, silver or gold model.
Column search: finds a column across all tables, e.g. uuid or mda_.
Expand every table: shows full column lists without scrolling.

The page is a single self-contained index.html with no external data files. It also works offline.

Pipeline layers
Layer	Filled by	Contents	Replaces
bronze	Step 1	Every Kobo submission as downloaded, all columns STRING	Postgres raw_data
silver	Steps 2 and 4	All submissions with readable column names and shared keys, plus the validation issues log	–
gold	Step 5	Approved submissions only, typed columns, 0/1 shown as no/yes	Postgres sarmaan2data
Tables and column counts
Table	Bronze	Silver	Gold
coverage_household	311	309	265
coverage_children_1_59	88	94	83
coverage_net_info	39	44	35
coverage_all_children	20	25	15
coverage_validation_issues	–	10	–
Total	458	482	398
Keys and relationships

coverage_household is the parent table. Each child table links back to it on a foreign key.

Silver and gold

Table	Primary key	Foreign key → parent
coverage_household	concatenated_id	–
coverage_all_children	child_id_childd_uuid	concatenated_id → coverage_household.concatenated_id
coverage_net_info	net_id_net_uuid	concatenated_id → coverage_household.concatenated_id
coverage_children_1_59	child_id_child_uuid	concatenated_id → coverage_household.concatenated_id
coverage_validation_issues (silver only)	–	– (standalone log table)

Bronze

Table	Primary key	Foreign key → parent
coverage_household	index_uuid	–
coverage_all_children	child_id_submission__uuid	_parent_index_submission__uuid → coverage_household.index_uuid
coverage_net_info	net_id_submission__uuid	_parent_index_submission__uuid → coverage_household.index_uuid
coverage_children_1_59	child_idd_submission__uuid	_parent_index_submission__uuid → coverage_household.index_uuid

Primary and foreign keys are declared in Databricks as informational constraints. The engine does not enforce them; the loader's validation checks do.

Updating the page
Regenerate the column data from the mapping files (coverage/docs/generate_erd.py, using the step 1, 2 and 5 map files).
Rebuild index.html with the new data.
In this repository, click Add file → Upload files, upload the new index.html and commit. GitHub Pages republishes within a minute or two, and the link stays the same.
Repository contents
File	Purpose
index.html	The interactive ERD page served by GitHub Pages
README.md	This file
Note on visibility

This repository is public so it can be served by free GitHub Pages. It contains table names, column names, data types and keys only, with no survey responses or personal data.

Maintained by: SARMAAN programme data team
