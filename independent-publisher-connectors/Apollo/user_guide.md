# Apollo Enrichment Connector - User Guide

This guide explains how to use the Apollo Enrichment connector to automatically fill in contact and company information in your Power Automate flows.

## What Does This Connector Do?

The Apollo connector looks up information about people and companies using Apollo.io's database of over 275 million B2B contacts. Give it an email address or company domain, and it returns:

**For Contacts:**
- Full name, job title, seniority level
- Phone numbers (direct dial, mobile)
- LinkedIn profile URL
- Current company information
- Location (city, state, country)

**For Accounts/Companies:**
- Industry, employee count, revenue
- Company description
- Social media links (LinkedIn, Twitter, Facebook)
- Funding history and investors
- Technology stack
- Headquarters address
- Company logo URL

## Available Actions

| Action | What It Does | When to Use |
|--------|--------------|-------------|
| **Enrich Contact** | Look up one person by email | Button clicks, form submissions, new lead triggers |
| **Enrich Account** | Look up one company by domain | Button clicks, new account triggers |
| **Bulk Enrich Contacts** | Look up up to 10 people at once | Scheduled batch jobs, importing lists |
| **Bulk Enrich Accounts** | Look up up to 10 companies at once | Scheduled batch jobs, importing lists |

## When to Use Individual vs Bulk

**Use Individual (Enrich Contact / Enrich Account) when:**
- A user clicks a button to enrich one record
- A new lead or account is created and you want to enrich it immediately
- You need to handle errors for each record separately

**Use Bulk when:**
- You're running a scheduled job to enrich many records
- You're importing a list and want to enrich everything
- Speed matters and you want fewer API calls

Both use the same number of Apollo credits (1 per record). The difference is efficiency - Bulk makes fewer API calls, which is faster and better for rate limits.

---

## Common Scenarios

### Scenario 1: Enrich a Contact When a Button is Clicked

**Goal:** User clicks "Enrich" button on a Contact form, and the contact's information is updated with Apollo data.

**Setup:**
1. Create a Custom API in your Dataverse solution (e.g., `fw_EnrichContact`)
2. Create a flow with trigger: **When an action is performed**
   - Table: Contacts
   - Action Name: `fw_EnrichContact`
3. Add action: **Get a row by ID** (Dataverse)
   - Table: Contacts
   - Row ID: `triggerBody()?['InputParameters/Target/@id']`
4. Add action: **Enrich Contact** (Apollo)
   - Email: `outputs('Get_a_row_by_ID')?['body/emailaddress1']`
5. Add action: **Update a row** (Dataverse)
   - Map Apollo fields to Contact fields

### Scenario 2: Enrich an Account When a Button is Clicked

**Goal:** User clicks "Enrich" button on an Account form, and the account's information is updated with Apollo data.

**Flow Structure:**
```
Trigger: When an action is performed (fw_EnrichAccount)
    ↓
Get Account row
    ↓
Extract domain from Website URL
    ↓
Enrich Account (Apollo)
    ↓
Update Account with Apollo data
    ↓
(Optional) Download and save company logo
```

**Extracting the Domain:**

If your Website field contains full URLs like `https://www.microsoft.com/products`, you need to extract just `microsoft.com`. Use this expression:

```
if(
  empty(outputs('Get_Account')?['body/websiteurl']),
  '',
  first(split(replace(replace(replace(toLower(outputs('Get_Account')?['body/websiteurl']),'https://',''),'http://',''),'www.',''),'/'))
)
```

This handles:
- Empty values (returns blank instead of error)
- https:// and http://
- www. prefix
- Trailing paths like /products

### Scenario 3: Automatically Enrich New Leads

**Goal:** When a new lead is created, automatically look up their information.

**Flow Structure:**
```
Trigger: When a row is added (Leads table)
    ↓
Condition: Email is not empty
    ↓
Enrich Contact (Apollo)
    ↓
Update Lead with Apollo data
```

**Tip:** Also enrich the company by extracting the domain from their email:
```
split(triggerOutputs()?['body/emailaddress1'], '@')[1]
```
This turns `john@microsoft.com` into `microsoft.com`.

### Scenario 4: Nightly Batch Enrichment

**Goal:** Every night, enrich accounts that haven't been enriched yet.

**Flow Structure:**
```
Trigger: Recurrence (Daily at 2 AM)
    ↓
List rows: Accounts where Last Enriched is empty (top 100)
    ↓
Select: Extract domains from Website URLs
    ↓
Compose: chunk(body('Select'), 10)
    ↓
Apply to each chunk:
    ↓
    Bulk Enrich Accounts
    ↓
    Apply to each result: Update Account
    ↓
    Delay: 1 second
```

The `chunk()` function splits your list into groups of 10 (Apollo's limit per bulk request).

---

## Working with Apollo's Output

### Contact Output Fields

| Apollo Field | Description | Example |
|--------------|-------------|---------|
| `person.name` | Full name | John Smith |
| `person.first_name` | First name | John |
| `person.last_name` | Last name | Smith |
| `person.title` | Job title | Senior Account Executive |
| `person.seniority` | Seniority level | senior |
| `person.linkedin_url` | LinkedIn profile | https://linkedin.com/in/johnsmith |
| `person.city` | City | San Francisco |
| `person.state` | State | California |
| `person.country` | Country | United States |
| `person.phone_numbers` | Array of phone numbers | (see below) |
| `person.organization.name` | Company name | Microsoft |

**Getting the first phone number:**
```
first(body('Enrich_Contact')?['person']?['phone_numbers'])?['sanitized_number']
```

### Account Output Fields

| Apollo Field | Description | Example |
|--------------|-------------|---------|
| `organization.name` | Company name | Microsoft Corporation |
| `organization.industry` | Primary industry | computer software |
| `organization.estimated_num_employees` | Employee count | 221000 |
| `organization.annual_revenue` | Revenue (number) | 198000000000 |
| `organization.annual_revenue_printed` | Revenue (formatted) | $198B |
| `organization.phone` | Main phone | +1-425-882-8080 |
| `organization.linkedin_url` | Company LinkedIn | https://linkedin.com/company/microsoft |
| `organization.logo_url` | Logo image URL | https://... |
| `organization.short_description` | Company description | Microsoft develops... |
| `organization.city` | HQ city | Redmond |
| `organization.state` | HQ state | Washington |
| `organization.country` | HQ country | United States |
| `organization.founded_year` | Year founded | 1975 |
| `organization.technology_names` | Tech stack array | ["Salesforce", "AWS", ...] |

---

## Tips and Best Practices

### Handle Empty Results

Not every lookup returns a match. Always check if the result exists before using it:

```
if(empty(body('Enrich_Contact')?['person']), '', body('Enrich_Contact')?['person']?['title'])
```

Or use a Condition action to check if `person` is not null before updating.

### Don't Enrich the Same Record Twice

Add a "Last Enriched" date field to your table. Update it when you enrich, and filter it out in future batch jobs:

**Filter for List rows:**
```
fw_lastenricheddate eq null or fw_lastenricheddate lt 2024-01-01
```

### Respect Rate Limits

Add a **Delay** action (1-2 seconds) between bulk calls in loops to avoid hitting Apollo's rate limits.

### Save the Apollo ID

Apollo returns an `id` for both people and organizations. Save this to a field in your CRM - you can use it for future lookups without consuming credits for matching.

### Downloading Company Logos

To save the company logo to your Account's image field:

1. Add **HTTP** action (GET):
   - URI: `body('Enrich_Account')?['organization']?['logo_url']`
2. Add **Upload a file or image** (Dataverse):
   - Table: Accounts
   - Row ID: Your account ID
   - Column: Default Image (entityimage)
   - Content: `body('HTTP')`
   - Content name: `logo.png`

---

## Troubleshooting

**"No match found"**
- Apollo doesn't have data for every email/domain
- Try a different email for the same person
- Verify the domain is correct (company domain, not personal email domain)

**"Rate limit exceeded" (429 error)**
- Add Delay actions between calls
- Reduce batch size
- Check your Apollo plan's rate limits

**Empty fields in results**
- Not all companies have all data points
- Check for null/empty before mapping to your CRM fields

**Bulk operation returns partial results**
- Some records may not match - check each result in the response array
- The results array order matches your input order

---

## Need Help?

- **Apollo API Documentation:** https://apolloio.github.io/apollo-api-docs/
- **Apollo Support:** Contact Apollo.io support for API-specific questions
- **Connector Issues:** Open an issue on the GitHub repository
