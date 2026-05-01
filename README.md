# emotion_jira_dataset

## Overview 

This repository makes the [Jira Emotion 2016 Dataset](https://web.archive.org/web/20210224080543/http://ansymore.uantwerpen.be/system/files/uploads/artefacts/alessandro/MSR16/archive3.zip) compatible with [Kaiaulu](https://github.com/sailuh/kaiaulu) for sentiment analysis of developer communication across open source software projects.

The emotion-labeled data comes from Ortu et al. (2016), ["The Emotional Side of Software Developers in JIRA"](https://doi.org/10.1145/2901739.2903505), and covers issue comments from four open source communities: Apache, Spring, JBoss, and CodeHaus.

---

## Dataset

Download the [dataset archive](https://web.archive.org/web/20210224080543/http://ansymore.uantwerpen.be/system/files/uploads/artefacts/alessandro/MSR16/archive3.zip) and extract it. The archive contains:

- `Groups/group1/` — 16 CSV files (one per rater), 392 comments, 6 emotions
- `Groups/group2/` — 3 CSV files (one per rater), 1,600 comments, 3 emotions
- `Groups/group3/` — 4 CSV files (one per emotion), ~4,000 sentences
- `emotion_dataset_jira.sql` — PostgreSQL database dump of the full Jira dataset

---

## Database Setup

The Jira database is PostgreSQL. Before running the notebooks, load the SQL dump into a local PostgreSQL instance.

### 1. Create the database

```bash
psql -U postgres -c "CREATE DATABASE jira;"
```

### 2. Import the SQL dump

The SQL file references a database user (`jira_user`) and database name (`jira_dataset`) that may not exist on your system. Use the command below to strip those lines out and load only the table definitions and data:

```bash
grep -v "DROP DATABASE\|CREATE DATABASE\|ALTER DATABASE\|\\\\connect\|OWNER TO jira_user" \
  /path/to/archive3/emotion_dataset_jira.sql | \
  psql -U postgres -d jira -v ON_ERROR_STOP=0 2>/dev/null
```

### 3. Import `jira_issue_report` separately

Due to encoding issues in some comment text, the `jira_issue_report` table may not load in the step above. Extract and import it separately:

```bash
grep "INSERT INTO jira_issue_report" \
  /path/to/archive3/emotion_dataset_jira.sql \
  > /path/to/archive3/jira_issue_report_only.sql

psql -U postgres -d jira -f /path/to/archive3/jira_issue_report_only.sql 2>/dev/null
```

### 4. Import `jira_user` separately

The `jira_user` table may also fail to load in the bulk import due to encoding issues. Extract and import it separately:

```bash
grep "INSERT INTO jira_user" \
  /path/to/archive3/emotion_dataset_jira.sql \
  > /path/to/archive3/jira_user_only.sql

psql -U postgres -d jira -f /path/to/archive3/jira_user_only.sql 2>/dev/null
```

You should see ~11,600 rows in `jira_user` after this step.

### 5. Verify the load

```bash
psql -U postgres -d jira -c "
SELECT relname AS table_name, n_live_tup AS row_count
FROM pg_stat_user_tables
ORDER BY n_live_tup DESC;"
```

You should see `jira_issue_comment` with ~1.8M rows, `jira_issue_report` with ~690K rows, and `jira_user` with ~11,600 rows.

---

## Setup

### 1. Create and activate a virtual environment

```bash
python -m venv .venv
source .venv/bin/activate
```

### 2. Install dependencies

```bash
pip install pandas numpy matplotlib sqlalchemy psycopg2-binary
```

---

## Notebooks

| Notebook | Description |
|---|---|
| `1_emotion_csv_inter_rater_agreement.ipynb` | Analyzes inter-rater agreement across all three groups and selects reliable comment IDs based on an adjustable threshold |
| `2_load_emotion_labels_to_postgres.ipynb` | Loads reliable emotion labels into PostgreSQL and verifies the join to `jira_issue_comment` and `jira_issue_report` |
| `3_explore_relevant_jira_projects.ipynb` | Explore which communities and Apache projects have labeled comments, confirm the join path, and extract project keys and date ranges needed for Kaiaulu |
| `4_generate_kaiaulu_configs.ipynb` | Auto-generates one Kaiaulu YAML config file per Apache project key that has emotion-labeled comments |
| `5_add_emotion_labels_to_kaiaulu.ipynb` | Joins emotion labels to Kaiaulu's downloaded Jira comment data and exports a labeled CSV per project |

---

## References

- Ortu et al. (2016). [The Emotional Side of Software Developers in JIRA](https://doi.org/10.1145/2901739.2903505). MSR 2016.
- Murgia et al. (2014). [Do Developers Feel Emotions?](https://doi.org/10.1145/2597073.2597086). MSR 2014.
