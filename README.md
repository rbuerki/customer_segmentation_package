# xxx Segments

This is a python package designed for the monthly calculation / update of the different segmentation models for xxx loyalty members:

- RFM scorings and segment calculation
- Affinties clustering
- Lifecycle segmentation

The aim of the package is to perform all of these calculations in one run and based on the same source data so that there are no inconsistencies resulting from different data loads, source tables etc. The results are written to a local CSV file and to a table in the analytics DB.

## Build

This package requires **Python 3.6** or higher. It runs in the "xxx_segments" conda environment which can be installed according to the `xxx_segments.yml` file. (It is worth to note that sk-learn has been fixed to version 0.21.2 because of a pickled clustering model that was developed with this version.)

## Run

In a terminal or command window, navigate to the top-level project directory (the one that contains this README).

Launch with:

``` python
python xxx_segments
```

To speed up the process during development and testing two CLI options can be passed to the run command:

- `-p` / `--preloaded` : Skips the stored procedure execution for loading the transaction data and thus fetches preloaded transaction data from the analytics DB. This is handy if you make changes to the code only.
- `-n` / `--nowriteback` : Skips the final insertion of the results into the database. They are still stored to a CSV file tough. This allows for experimentation without overwriting existing results.

## Output

The script results in a pandas DataFrame consisting of a row for every xxx loyalty member (identified by MemberAK and MemberSK) with the different metrics and segment scorings. These are:

| "yearmon" | "MemberAK" | "MemberSK" | "recency" | "frequency" | "monetary" | "RFM-Score" | "RFM_3" | "RFM_Segment" | "Affinität_Segment" | "Lifecycle_Segment" | "R" | "F" | "M" |
| --------- | ---------- | ---------- | --------- | ----------- | ---------- | ----------- | ------- | ------------- | ------------------- | ------------------- | --- | --- | --- |
| ...       | ...        | ...        | ...       | ...         | ...        | ...         | ...     |...            |...                  |...                  | ... | ... | ... |


- `yearmon` A 4 digit int (yymm) indicating the final month of the period the actual results were updated for.*
- `recency`: Number of days since last purchase (up to 3 years back)
- `frequency`: Number of days with a transaction within the last 1 year
- `monetary`: Sum of turnover within last 2 years
- `rfm_score`: 3 digit RFM-Score according to "PKZ-Logic"
- `rfm_3`: "Classic" 3 digit RFM-Score consisting of concatentated R-F-M quantile classifications
- `rfm_segment`: RFM-segment classification (based on "rfm_3")
- `cluster_name`: Segment classification based on products purchased
- `lifecycle_segment`: Segment classification on temporal components and frequencies

*The reference date / end date for all calculations is always _the last day of the last full month for which transaction data has been loaded_.

### Upsert / Merge into DB

These results are merged into a DB-table containing the results of all past runs / months. The `yearmon` column together with the `MemberAK` form a composite primary key. See the table create statement for the details and exact column structure:

```SQL
CREATE TABLE xxx_analytics.dbo.xxx_segments_hist (
  yearmon smallint,
  MemberAK int,
  MemberSK int NOT NULL,
  recency smallint,
  frequency smallint,
  monetary money,
  R tinyint,
  F tinyint,
  M tinyint,
  RFM-Score smallint,
  RFM_3 smallint,
  RFM_Segment nvarchar(30),
  Affnität_Segment nvarchar(40),
  Lifecycle_Segment nvarchar(30),
  PRIMARY KEY (yearmon, MemberAK)
)
```

The merge logic is as follows:

- INSERT rows if the primary key does not yet exist (appends results for the first run of a new month to the table)
- UPDATE row entries if the primary key already exists (for example if you rerun the script for a given month, it updates the scores that have changed)

**Backfills**: This logic makes it possible to "backfill" results for earlier periods by running the script with past transaction data. But one has to be aware that the loyality member pool and the contents of the helper tables in the analytics DB (on Warengruppen and Sites) will be loaded with actual values. (This means for example that more recently created members will be considered for modelling and will be classified as "lost inactive" for earlier periods.)

_(Actual work-around for backfills: Use the stored procedure "manual date" and set the dates manually in the transaction query.)_


### Save "Export-File" to CSV

A subset of the result columns is also written to a CSV file in the local `data/EXPORT/` folder (or whatever SAVE_PATH is defined in the settings.py). The file name includes a 4-digit yearmon code and a timestamp. These files are meant for iLoy to import the CRM-scores into the PROD DB.

This CSV has the following columns:

| "MemberAK" | "MemberSK" | "R" | "F" | "M" | "RFM-Score" | "RFM_3" | "RFM_Segment" | "Affinität_Segment" | "Lifecycle_Segment" |
| ---------- | ---------- | --- | --- | --- | ----------- | ------- | ------------- | ------------------- | ------------------- |
| ...        | ...        | ... | ... | ... | ...         | ...     |...            |...                  |...                  |


[THE REST OF THIS README IS NOT DISCLOSED]# customer_segmentation_package
