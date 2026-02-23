# HubSpot Contact Import Walkthrough

Step-by-step guide for importing CSV contact lists into HubSpot as static segments.

## Overview

HubSpot's import wizard lets you upload a CSV file and automatically create a static "contacts segment" — a fixed list of contacts you can target with emails. This is what we use to create A/B test groups.

## Step 1: Start the Import

1. Go to **CRM > Contacts** in HubSpot
2. Click the **Import** button (top right)
3. Select **Start an import**
4. Choose **File from computer**
5. Select **One file** and **One object**
6. Choose **Contacts** as the object type
7. Click **Next**

## Step 2: Upload Your CSV

1. Drag and drop your CSV file or click to browse
2. HubSpot accepts `.csv` files with UTF-8 encoding
3. The file must have a header row — the skill generates files with an `Email` header
4. Click **Next** after upload

### Troubleshooting file issues

- **"File format not supported"**: Ensure the file ends in `.csv` and is comma-separated (not semicolon)
- **"No data found"**: Check the file isn't empty and has both a header row and data rows
- **Encoding errors**: Re-save the file as UTF-8 CSV if you see garbled characters

## Step 3: Map Columns

1. HubSpot auto-detects the `Email` column and maps it to the **Email** contact property
2. Verify the mapping shows: `Email` → `Email` with a green checkmark
3. If it doesn't auto-map, click the dropdown next to your column and select **Email**
4. Click **Next**

### Tips

- The skill generates single-column CSVs (Email only) — mapping is straightforward
- If you have additional columns (name, company), map them to the corresponding HubSpot properties
- Unmapped columns are ignored during import

## Step 4: Import Details

This is the critical step for A/B testing:

1. **Name your import**: Use a descriptive name that ties to the test variant
   - Pattern: `{Campaign} - Variant {Letter}` (e.g., "February Survey - Variant A")
   - This name also becomes the segment name, so make it recognizable

2. **Create a contacts segment**: **CHECK THIS BOX** — this is what creates the static list you'll send emails to. If you skip this, you only import contacts without a targetable segment.

3. **Duplicate handling**: HubSpot matches on Email by default
   - Existing contacts are updated (not duplicated)
   - New email addresses create new contact records

4. **Accept the data processing agreement**: Check the consent box

5. Click **Finish import**

## Step 5: Verify the Import

1. After import completes, HubSpot shows a summary with:
   - Number of contacts created
   - Number of contacts updated
   - Any errors or skipped rows
2. Go to **CRM > Lists** to find your new segment
3. Click the segment to verify the contact count matches your CSV

## Naming Conventions

Use consistent naming across all test groups for easy identification:

| Import Name | Segment Name (auto-created) | Email Variant |
|---|---|---|
| February Survey - Variant A | February Survey - Variant A | Subject line A |
| February Survey - Variant B | February Survey - Variant B | Subject line B |
| February Survey - Variant C | February Survey - Variant C | Subject line C |
| February Survey - Winner Send | February Survey - Winner Send | Winning variant |

## Repeating for Multiple Groups

For A/B/C split tests, repeat the entire import process for each CSV file:

1. Import `variant_a.csv` → creates segment "Variant A"
2. Import `variant_b.csv` → creates segment "Variant B"
3. Import `variant_c.csv` → creates segment "Variant C"

Each import is independent. You can run them back-to-back — no need to wait between imports.

## Common Gotchas

- **Forgot to check "Create a contacts segment"**: You'll need to re-import the same file with the checkbox enabled. The duplicate handling ensures contacts aren't duplicated.
- **Wrong file uploaded**: Simply re-import the correct file. Contacts matched by email get updated, not duplicated.
- **Segment not showing up in email send**: Make sure you're looking under **Lists** (not filters). Static segments appear alongside active lists.
- **Contacts not receiving emails**: Check that imported contacts have `Marketing contact status = Marketing contact`. Non-marketing contacts can't receive marketing emails regardless of list membership.
