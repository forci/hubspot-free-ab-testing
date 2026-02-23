---
name: hubspot-ab-test-lists
version: 1.1.0
description: >
  Create A/B (or A/B/C/…) split test contact lists for HubSpot email campaigns.
  Designed for HubSpot Starter and Free plans that lack native A/B testing.
  Filters eligible marketing contacts, randomizes them into equal groups, and
  produces import-ready CSV files for HubSpot static segments.
  Trigger on: "split test", "A/B test emails", "create test segments",
  "randomize contacts for email test" in a HubSpot context.
license: MIT
---

# HubSpot A/B Test Lists

Create randomized contact segments for split-testing email campaigns on HubSpot Starter/Free plans.

## When to Use

- You're on HubSpot **Starter** or **Free** and need to compare email variants (subject lines, body copy, send times)
- You want to send variant A to one group, variant B to another, then send the winner to everyone else
- Works with any number of variants (A/B, A/B/C, etc.)

## Prerequisites

- A HubSpot account with contacts
- A contact export CSV from HubSpot (or HubSpot MCP access to pull contacts via API)
- Email variants already drafted (or planned) in HubSpot

## Workflow

### Step 1: Identify Email Variants

Ask the user to list their email variants. Clarify what differs between them (subject line, body, CTA, send time).

Display them for confirmation:

```
Your test variants:
1. Variant A — "{subject line A}"
2. Variant B — "{subject line B}"
3. Variant C — "{subject line C}" (if applicable)

Does this look right?
```

Confirm the number of groups matches the number of variants.

### Step 2: Get the Contact List

Offer two paths:

**Path A — CSV Export (recommended for most users):**

1. In HubSpot, go to **CRM > Contacts**
2. Apply any filters you want (e.g., a specific list, lifecycle stage, or all contacts)
3. Click **Export** (top right) > **Export view**
4. Choose CSV format, wait for the email, download the file
5. Ask the user for the file path

**Path B — HubSpot MCP/API:**

If HubSpot MCP is connected, pull contacts directly:

```
Use hubspot-list-objects or hubspot-search-objects to fetch contacts with:
- Email
- Marketing contact status (hs_marketable_status)
- Email subscription statuses
```

This path skips the CSV export and feeds directly into filtering.

### Step 3: Filter Eligible Contacts

Generate a Python script at runtime that filters contacts for email eligibility.

The script must:

1. Read the CSV using `csv.DictReader`
2. Apply these filters (keep contacts that pass ALL):
   - Has a non-empty email address
   - `Marketing contact status` equals `"Marketing contact"` (column may be named `Marketing contact status` or `hs_marketable_status`)
   - NOT opted out of email (check `Opted out of email: Marketing Information` or relevant subscription columns — value should NOT be `"Opted Out"`)
   - NOT globally unsubscribed (check `Unsubscribed from all email` — value should NOT be `"True"` or `"Yes"`)
   - NOT unengaged — filter out contacts where `Sends Since Last Engagement` is high (>5) or `Marketing emails opened`/`Marketing emails clicked` are both 0. This prevents HubSpot's "Don't send to unengaged contacts" toggle from unevenly trimming groups at send time.
3. Deduplicate by email (case-insensitive, keep first occurrence)
4. Print a summary: total contacts, eligible contacts, filtered-out breakdown (including unengaged count)

Example script structure (generate this, do NOT store it as a file):

```python
import csv
import sys

def filter_contacts(csv_path):
    total = 0
    eligible = []
    no_email = 0
    non_marketing = 0
    opted_out = 0
    unsubscribed = 0
    seen_emails = set()

    with open(csv_path, 'r', encoding='utf-8-sig') as f:
        reader = csv.DictReader(f)
        headers = reader.fieldnames

        # Detect column names (HubSpot exports vary)
        email_col = next((h for h in headers if h.lower().strip() == 'email'), None)
        marketing_col = next((h for h in headers if 'marketing contact status' in h.lower() or h == 'hs_marketable_status'), None)
        unsub_col = next((h for h in headers if 'unsubscribed from all' in h.lower()), None)
        optout_cols = [h for h in headers if 'opted out' in h.lower()]

        if not email_col:
            print("ERROR: No 'Email' column found. Available columns:", headers)
            sys.exit(1)

        for row in reader:
            total += 1
            email = row.get(email_col, '').strip().lower()

            if not email:
                no_email += 1
                continue
            if email in seen_emails:
                continue
            seen_emails.add(email)

            if marketing_col and row.get(marketing_col, '').strip() != 'Marketing contact':
                non_marketing += 1
                continue

            if unsub_col and row.get(unsub_col, '').strip().lower() in ('true', 'yes'):
                unsubscribed += 1
                continue

            is_opted_out = False
            for col in optout_cols:
                if row.get(col, '').strip().lower() in ('opted out', 'true', 'yes'):
                    is_opted_out = True
                    break
            if is_opted_out:
                opted_out += 1
                continue

            eligible.append(email)

    print(f"Total rows: {total}")
    print(f"Eligible contacts: {len(eligible)}")
    print(f"Filtered out:")
    print(f"  No email: {no_email}")
    print(f"  Non-marketing contacts: {non_marketing}")
    print(f"  Unsubscribed from all: {unsubscribed}")
    print(f"  Opted out: {opted_out}")
    print(f"  Duplicates: {total - no_email - non_marketing - unsubscribed - opted_out - len(eligible)}")

    return eligible

eligible = filter_contacts(sys.argv[1])
```

**Important**: Adapt the column-detection logic to the actual headers in the user's CSV. HubSpot export headers vary by account language and configuration.

### Step 4: Randomize into Groups

Extend the script to split eligible contacts into N equal groups:

```python
import random

random.seed(42)  # Reproducible results
random.shuffle(eligible)

n_variants = {N}  # Number of variants from Step 1
group_size = len(eligible) // n_variants
groups = []
for i in range(n_variants):
    start = i * group_size
    end = start + group_size
    groups.append(eligible[start:end])

# Remaining contacts (for winner send later)
remaining = eligible[n_variants * group_size:]

for i, group in enumerate(groups):
    letter = chr(65 + i)  # A, B, C, ...
    print(f"Group {letter}: {len(group)} contacts")
print(f"Remaining (winner send): {len(remaining)} contacts")
```

Report group sizes to the user. All test groups should be equal. Remaining contacts go into the winner-send pool.

### Step 5: Create Import CSVs

Write one CSV per group. Each file has a single `Email` column:

```python
import os

output_dir = os.path.dirname(csv_path)  # Same directory as source CSV

for i, group in enumerate(groups):
    letter = chr(65 + i)
    filename = f"{test_name}_variant_{letter}.csv"
    filepath = os.path.join(output_dir, filename)
    with open(filepath, 'w', newline='') as f:
        writer = csv.writer(f)
        writer.writerow(['Email'])
        for email in group:
            writer.writerow([email])
    print(f"Written: {filepath} ({len(group)} contacts)")

# Also write the remaining contacts for the winner send
if remaining:
    filepath = os.path.join(output_dir, f"{test_name}_remaining.csv")
    with open(filepath, 'w', newline='') as f:
        writer = csv.writer(f)
        writer.writerow(['Email'])
        for email in remaining:
            writer.writerow([email])
    print(f"Written: {filepath} ({len(remaining)} contacts)")
```

Ask the user for a descriptive `test_name` prefix (e.g., `february_survey`).

Tell the user where the files were saved.

### Step 6: Import to HubSpot

**Path A — Browser-guided import:**

For each CSV file, walk the user through:

1. Go to **Data Management > Import** (or CRM > Contacts > Import)
2. Click **Start an import**
3. Select: File from computer > One file > One object > Contacts
4. Upload the CSV file
5. Map the `Email` column to the Email property (usually auto-detected)
6. Name the import using the **variant's distinguishing trait**, not just a letter: `{Campaign Name} - {what makes this variant different}`. Example: `Survey Test A - with form link`, `Survey Test B - without form link`. Confirm each name with the user before importing — a mislabeled segment causes confusion when sending and evaluating.
7. **Check "Create a contacts segment"** — this creates the static list for email targeting
8. Accept the data processing agreement
9. Click **Finish import**

Repeat for each variant CSV.

See `references/hubspot-import-walkthrough.md` for detailed guidance and troubleshooting.

**Path B — Manual import:**

Provide the CSV file paths and tell the user to import them manually into HubSpot, reminding them to check "Create a contacts segment" for each import.

### Step 7: Send Emails

After all segments are imported, guide the user to send each email variant to its corresponding segment:

```
Send mapping:
- "{Variant A name}" → send to segment "{Campaign} - Variant A"
- "{Variant B name}" → send to segment "{Campaign} - Variant B"
- "{Variant C name}" → send to segment "{Campaign} - Variant C" (if applicable)
```

In HubSpot's email editor, under **Send to**, select the corresponding static list/segment.

**Critical: Uncheck "Don't send to unengaged contacts"** in the send settings. This filter runs AFTER segment selection and removes different numbers from each group (e.g., 3 from group A but only 2 from group B), destroying equal group sizes and invalidating the test. The filtering script already excluded unengaged contacts before splitting, so this toggle is redundant and harmful.

### Step 8: Evaluate & Send Winner

After the test period (recommend **3-5 business days** for reliable data):

1. Compare metrics across variants:
   - **Open rate** — which subject line performs best
   - **Click rate** — which content/CTA drives action
   - **Reply rate** — which variant generates responses (if relevant)
2. Declare a winner based on the primary metric the user cares about
3. Import the `remaining.csv` file as a new segment (follow Step 6 again)
4. Send the winning email variant to the remaining contacts segment

## Common Mistakes

- **Leaving "Don't send to unengaged contacts" checked**: This is the #1 test-killer. HubSpot's unengaged filter runs at send time and removes different numbers from each group, making them unequal (e.g., 17/20 vs 18/20 vs 20/20). Always uncheck it — the filtering script already excludes unengaged contacts before splitting.
- **Mislabeling segments during import**: Name each segment by its distinguishing trait (`Survey Test A - with form link`), not generically. If variant B is "without form link", don't name its segment "with form link". Double-check names before finishing each import — renaming segments after the fact is possible but easy to forget.
- **Sending to non-marketing contacts**: HubSpot silently skips non-marketing contacts. The filtering step catches this, but double-check if your eligible count seems low.
- **Skipping unsubscribe checks**: Importing opted-out contacts into a segment doesn't override their opt-out — HubSpot still blocks the send — but it inflates your segment size and skews metrics.
- **Peeking at results too early**: Open/click tracking needs time. Checking after 2 hours gives misleading data. Wait at least 3 business days.
- **Unequal group sizes**: The randomization ensures equal groups. If you manually edit the CSVs, keep groups the same size for valid comparison.
- **Forgetting "Create a contacts segment"**: Without this checkbox, HubSpot imports the contacts but doesn't create a targetable list. You'd need to re-import.
- **Not using the remaining list for the winner send**: The whole point is to test on a subset, then send the winner to everyone else. Don't forget the final send.
