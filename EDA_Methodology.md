# EDA Methodology

## Overview

This document describes the step-by-step methodology adopted to transform a raw, 
unstructured dataset of Apple job postings into a clean, structured dataset ready 
for analysis and SQL import.

---

## Step 1 — Initial Data Inspection

The first step was a preliminary visual inspection of the raw dataset.  
The data was exported to an Excel file to quickly assess the structure and content 
of each column.

**Problem encountered — Excel Formula Injection:**  
Some text fields began with special characters (`=`, `+`, `-`, `@`), which Excel 
interprets as formulas, corrupting the display. A sanitization function was applied 
before export to force Excel to treat these fields as plain text.

---

## Step 2 — Job Title Standardization (Role Category)

The `title` column contained hundreds of free-text variations of job roles.  
The goal was to create a new `Role_Category` column to group titles into macro 
categories: **SD** (Software Developer), **SE** (Software Engineer), and **OTHER**.

Two approaches were tested:

- **Attempt 1 — Strict regex matching:** Used word boundaries to match full words 
  like "developer" and "engineer". This left a high number of roles unclassified 
  as OTHER due to abbreviations and partial words.

- **Attempt 2 — Flexible partial matching:** Lowered the matching threshold to 
  substrings like "dev" and "eng". Results did not improve significantly.

**Reflection:** This step raised a deeper analytical question about whether the 
SD vs SE distinction was meaningful enough to justify the effort. This led to 
a broader reconsideration of the analytical approach, prioritizing a thorough 
understanding of each variable before any transformation.

---

## Step 3 — Variable Analysis and Problem Scoping

Before proceeding with any further feature engineering, each column was carefully 
analyzed to understand its meaning and the challenges it presented:

- **`title`:** Highly heterogeneous free-text. Meaningful classification would 
  require keyword-based segmentation by domain (iOS, Java, back-end, etc.).
- **`location`:** Multiple naming conventions for the same city. 
  Required standardization.
- **`minimum_qual`, `preferred_qual`, `responsibilities`, `education_experience`:** 
  Unstructured multi-line text blocks. Extremely difficult to parse without 
  AI-based methods.

This analysis also clarified the business questions the dataset could answer and 
who could benefit from the investigation (e.g., job seekers, competing companies, 
researchers).

---

## Step 4 — Skill Extraction from Unstructured Text

Three approaches were evaluated to extract structured skills from the raw 
qualification fields.

### Approach 1 — Keyword Matching

A curated list of technical skills (programming languages, tools, frameworks, 
cloud platforms, etc.) was compiled. Each job description was scanned for exact 
or partial keyword matches using regex with word boundaries.

**Result:** Over 1,500 out of ~2,200 rows returned fewer than 3 skills. 
Expanding the dictionary reduced empty records by ~200 units, but coverage 
remained insufficient. The method was deemed invalid for this use case.

### Approach 2 — Basic LLM Extraction

A basic prompt was sent to an LLM (GPT-4o-mini) asking it to freely extract 
technical skills from each job description as a comma-separated list.

**Result:** Improved coverage, but the unstructured output was inconsistent and 
difficult to aggregate reliably.

### Approach 3 — LLM with Structured Outputs ✓ (Selected Method)

A Pydantic schema was defined to enforce a strict output format from the LLM. 
For each job posting, the model was required to return:

- `programming_languages` — list of programming languages mentioned
- `tools_and_technologies` — frameworks, platforms, and tools
- `soft_skills` — interpersonal and organizational skills
- `years_experience_min` — minimum years of experience as an integer
- `experience_level` — one of: `junior`, `mid`, `senior`, `not specified`

**Result:** The most complete and consistent method. Successfully processed 
2,148 rows in approximately 76 minutes at a total cost of ~$3.00 USD.

**Key lesson learned:** The output schema should be defined before extraction, 
not after. Constraining the LLM output to a fixed list of categories 
(using `Literal` types in Pydantic) would have eliminated the need for 
the normalization step described below.

---

## Step 5 — Normalization via LLM Bucketing

Due to the generic prompt used during extraction, the raw output contained 
an excessive number of unique values:

- `tools_and_technologies`: ~4,352 unique values
- `programming_languages`: ~200 unique values
- `soft_skills`: ~2,190 unique values

These included duplicates, synonyms, and overly specific terms.

**Solution — Two-stage LLM bucketing:**

1. A first LLM call defined 10–15 standard macro-categories (buckets) 
   for each column (e.g., "Cloud Computing", "DevOps", "Machine Learning/AI").
2. Subsequent calls mapped every unique extracted value to one of those buckets, 
   processing values in chunks of 300 to stay within token limits.

Values that could not be mapped to any bucket were assigned to `Other`.

---

## Step 6 — Geographical Classification

The `location` column contained hundreds of unique location strings with 
inconsistent formatting. An LLM was used to classify each unique location 
into one of three macro-regions:

- **North America**
- **Europe**
- **Asia**

Locations in South America were mapped to North America, and Oceania to Asia, 
as per the defined classification rules. The mapping was processed in chunks 
of 50 locations per API call to ensure reliability.

---

## Step 7 — Database Preparation and SQL Import

The cleaned and normalized dataset was prepared for import into a MySQL database 
managed via Beekeeper Studio.

Transformation steps applied:
- Renamed the index column to `id` to serve as primary key.
- Retained only the relevant columns: `id`, `programming_languages`, 
  `tools_and_technologies`, `years_experience_min`, `experience_level`, `location`.
- Converted Python list fields into clean comma-separated strings.
- Imported the skills data into the `skills` table via SQLAlchemy.
- Updated the `location` column row by row using the geographical mapping 
  generated in the previous step.

The final cleaned dataset was also exported as `clean_applejobs.csv` for use 
in subsequent analysis.

---

## Step 8 — Exploratory Analysis (North America Focus)

Using the cleaned dataset, a targeted analysis was performed on job postings 
located in North America. The analysis addressed three key questions:

1. **Which programming languages are most in demand?**  
   Frequency of language mentions was calculated across all North America postings 
   and expressed as a percentage of total mentions.

2. **Which tools and technologies are priority skills?**  
   The same frequency analysis was applied to the tools and technologies column.

3. **What is the expected level of experience?**  
   The distribution of experience levels (junior, mid, senior) was computed, 
   excluding "not specified" entries. The average minimum years of experience 
   was also calculated as a summary statistic.

Results were visualized using Matplotlib and Seaborn, and saved to the 
`data/plots/` directory.
