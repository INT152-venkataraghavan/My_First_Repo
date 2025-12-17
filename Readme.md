This is a professional README for your email cleaning and normalization Python script.

#  Email Cleaning and Normalization Script

This Python project provides a robust solution for cleaning, normalizing, and validating email addresses in a dataset. It handles various common data quality issues such as typos, inconsistent formatting, dynamic null values, and missing TLDs (Top-Level Domains) by leveraging configuration files and facility-based inference.

##  Features

  * **Configuration-Driven:** Uses a `config.json` file to manage paths for all input and mapping files.
  * **Email Extraction:** Extracts emails from common wrapper formats (e.g., `<email@domain.com>`, `mailto:email@domain.com`).
  * **Normalization:**
      * Removes excessive spaces and standardizes spacing around `@` and `.`
      * Applies **token replacements** (e.g., replacing " at " with "@").
  * **Typo Correction:**
      * Corrects common **TLD typos** (e.g., `.con` to `.com`).
      * Replaces common separators (like `(at)`, `[at]`) with `@` if no `@` exists.
      * Handles repeated punctuation (`..`, `@@`).
  * **Null Handling:** Identifies and treats custom null strings (e.g., 'NA', 'none', 'n/a') as true nulls.
  * **Facility-Based TLD Inference:** For emails missing a TLD (e.g., `user@stanford`), it infers the most common TLD from other valid emails within the same `facility_id`.
  * **Detailed Output:** Generates a summary of cleaning results and a detailed output CSV with the original email, formatted email, validation status, and a breakdown of changes applied.

-----

##  Prerequisites

  * Python 3.x
  * The `pandas` library (`pip install pandas`)

-----

##  Getting Started

### 1\. Project Structure

Ensure your project directory contains the following files:

```
.
├── email_cleaner.py     # The main script
├── config.json          # Configuration file
├── tld_fixes.csv        # TLD typo mapping
├── token_replacements.csv # Token replacement mapping
├── separators_to_at.csv # Separator list for '@' replacement
└── null_values.csv      # List of dynamic null values
```

### 2\. Configuration Files

The script relies on several CSV files for its logic. Examples are provided below.

#### `config.json`

This file specifies the paths to all other configuration and input files.

```json
{
    "input_path": "input.csv",
    "tld_fixes_csv": "tld_fixes.csv",
    "token_replacements_csv": "token_replacements.csv",
    "separators_to_at_csv": "separators_to_at.csv",
    "null_values_csv": "null_values.csv"
}
```

#### `input.csv`

The main dataset must contain the email column (default: `e_mail`) and the facility ID column (hardcoded: `facility_id`).

| e\_mail | facility\_id | other\_data |
| :--- | :--- | :--- |
| john.doe at [gmail.com] | FAC001 | ... |
| jane@stanford | FAC002 | ... |
| email@https://www.google.com/url?sa=E\&source=gmail\&q=gmaill.com | FAC001 | ... |
| NA | FAC003 | ... |

#### `tld_fixes.csv`

Used by `correct_common_typos` to fix common Top-Level Domain mistakes.

| wrong | right |
| :--- | :--- |
| https://www.google.com/url?sa=E\&source=gmail\&q=gmaill.com | gmail.com |
| com.co | com |
| co. | com |

#### `token_replacements.csv`

Used by `normalize_raw_email` to replace non-standard tokens with standard email characters.

| token | replacement |
| :--- | :--- |
| (dot) | . |
| \[DOT\] | . |
| @ | at |

#### `separators_to_at.csv`

Used by `replace_separators_with_at_when_reasonable` to substitute separators with `@` when the email is missing one.

| sep |
| :--- |
| at |
| (at) |
| \[at\] |

#### `null_values.csv`

Used by `is_dynamic_null` to identify custom null strings.

| value |
| :--- |
| NA |
| N/A |
| none |
| - |

### 3\. Execution

Run the main script from your terminal:

```bash
python email_cleaner.py
```

## Output

The script will generate an output CSV file named like `email_reformatted_YYYYMMDD_HHMMSS.csv` and print a summary to the console.

### Console Summary Example

```
{'total_rows': 1000, 'valid_before': 750, 'valid_after': 950, 'newly_fixed': 200, 'invalid_after': 50, 'null_original': 20, 'null_formatted': 70}
```

  * **`total_rows`**: Total number of email entries processed.
  * **`valid_before`**: Emails that were valid **before** cleaning.
  * **`valid_after`**: Emails that are valid **after** cleaning.
  * **`newly_fixed`**: The number of invalid emails successfully cleaned and validated.
  * **`invalid_after`**: Emails that remain invalid or were not fixable.
  * **`null_original`**: Emails marked as null based on the `null_values.csv`.
  * **`null_formatted`**: Total emails resulting in a `None` formatted value (original nulls + those that became null after cleaning).

### Output CSV (`email_reformatted_...csv`)

The output CSV will contain the following columns, providing a detailed audit trail of the cleaning process:

| Column Name | Description |
| :--- | :--- |
| `e_mail` | The original raw email address. |
| `formatted_email` | The final, cleaned, and validated email (lowercase). `None` if invalid or a null value. |
| `valid_before` | `1` if the original email was valid, `0` otherwise. |
| `valid_after` | `1` if the `formatted_email` is valid, `0` otherwise. |
| `invalid_email` | The resulting email if it was still invalid after cleaning (for inspection). |
| `null_value` | `1` if marked as a dynamic null. |
| `removed_spaces` | `1` if excessive spaces were removed. |
| `extracted_from_wrappers` | `1` if extracted from a format like `<...>` or `mailto:`. |
| `domain_change` | `1` if TLD inference was applied. |
| `typo_fix` | `1` if a typo (TLD fix, `@@` fix, or separator-to-`@` fix) was applied. |
| `punctuation` | `1` if leading/trailing punctuation was stripped. |
| `token_replacement` | `1` if tokens like `(dot)` or `at` were replaced. |
| `no_change` | `1` if the cleaning functions were run but resulted in no recorded change note. |

-----

## Code Deep Dive

The core logic is managed by the following functions:

  * **`load_...` functions**: Handle the loading of all configuration CSV files into memory (as Dictionaries or Lists).
  * **`extract_email_from_wrappers`**: Uses regex to clean up emails surrounded by brackets, parentheses, or `mailto:`.
  * **`normalize_raw_email`**: Applies token replacements and general space cleanup.
  * **`correct_common_typos`**: Applies TLD fixes and fixes repeated punctuation/at-signs.
  * **`replace_separators_with_at_when_reasonable`**: Attempts to fix missing `@` by replacing a known separator.
  * **`infer_tld_from_facility`**: The advanced function that infers a missing TLD based on other records for the same `facility_id`.
  * **`validate_email`**: The strict regex validator that determines the final validity.
  * **`process_dataframe_minimal`**: The main orchestration function that iterates over the dataset, applies all cleaning steps, and tracks the resulting changes.