# 🤖 AI Job Application Automation Pipeline

> Automated LinkedIn job scraping, AI-powered filtering, resume customization, and application tracking — built with n8n, Claude AI, and Google Workspace.

**From 8 manual applications/day → 40+ automated applications/day**

---

## 🖼️ Workflow Preview

![n8n Workflow](workflows/Screenshot%202026-06-23%20152437.png)

---

## 🎯 What This Does

This pipeline automates the entire job application process:

1. **Scrapes LinkedIn** — pulls 150+ fresh job postings daily from verified tech companies
2. **AI Filters** — Claude reviews each job for relevance, visa sponsorship, seniority fit
3. **Customizes Resume** — Claude tailors your resume to each job with ATS keywords
4. **Creates Documents** — generates a Google Doc resume per job automatically
5. **Logs Everything** — appends all job data to a shared Google Sheet for team tracking

---

## 📊 Results

| Metric | Before (Manual) | After (Automated) |
|---|---|---|
| Applications/day | 8 | 40+ |
| Time per application | 30 minutes | 2 minutes |
| Monthly applications | 160 | 1,200+ |
| Cost per resume | ~$0 (your time) | ~$0.009 |
| Monthly cost | 0 (but 4hrs/day) | ~$15 |

---

## 🏗️ Architecture

```
Manual/Scheduled Trigger
         ↓
Scrape Jobs (Apify LinkedIn Scraper)
         ↓
Remove Duplicates + Filter Staffing Companies (Code Node)
         ↓
Loop Over Items (Batch Size = 1)
         ├→ Check Relevance (Claude AI)
         ├→ Parse Response (Code Node)
         ├→ Filter (Verdict = true, Sponsorship ≠ No)
         ├→ Customize Resume (Claude AI)
         ├→ Clean HTML Output (Code Node)
         ├→ Create Google Doc
         ├→ Upload Resume HTML to Drive
         ├→ Create Daily Sheet Tab
         └→ Append Row to Google Sheets
```

---

## 🛠️ Tech Stack

| Component | Tool |
|---|---|
| Workflow Automation | n8n (self-hosted) |
| Job Scraping | Apify (LinkedIn Jobs Scraper) |
| AI Filtering & Resume | Claude Haiku (claude-haiku-4-5) |
| Document Generation | Google Docs API |
| Application Tracking | Google Sheets API |
| Resume Storage | Google Drive |

---

## 🚀 Getting Started

### Prerequisites

- [n8n](https://n8n.io) (self-hosted or cloud)
- [Apify account](https://apify.com) (free tier works)
- [Anthropic API key](https://console.anthropic.com)
- Google account with Drive + Sheets + Docs access

### Setup

**1. Self-host n8n for FREE using Docker**

> 💡 This is how to run n8n completely free — no subscription needed. Just Docker on your local machine.

Install Docker: [docker.com/get-started](https://www.docker.com/get-started)

Then run this single command:
```bash
docker run -it --rm \
  --name n8n \
  -p 5678:5678 \
  -v ~/.n8n:/home/node/.n8n \
  n8nio/n8n
```

Then open your browser and go to:
```
http://localhost:5678
```

That's it — n8n is running locally for free! 🎉

> **Why self-host?** n8n cloud costs $20+/month. Running it in Docker on your own machine is 100% free. Your workflows run whenever your computer is on.

**2. Clone this repo**
```bash
git clone https://github.com/Haneesh-Nellore/job-automation-pipeline.git
cd job-automation-pipeline
```

**3. Set up Google Workspace**
- Create a Google Sheet for tracking (note the Sheet ID)
- Create a Google Drive folder for resumes (note the Folder ID)
- Create a master resume Google Doc (note the Doc ID)
- Set up OAuth2 credentials for Google APIs in n8n

**3. Set up Apify**
- Sign up at apify.com
- Get your API token
- Use the `apify/linkedin-jobs-scraper` actor

**4. Set up Claude**
- Get your Anthropic API key
- Add it to n8n credentials

**5. Import the workflow**
- Open n8n → Workflows → Import
- Upload `workflows/n8n_workflow_template.json`
- Update all credential references

**6. Configure your profile**

Edit `config/candidate_profile.json`:
```json
{
  "name": "Your Name",
  "email": "your@email.com",
  "phone": "+1 (xxx)-xxx-xxxx",
  "location": "City, State",
  "years_experience": 4,
  "visa_status": "F1 OPT / H1B / Citizen / etc",
  "skills": {
    "languages": ["Java", "Python", "TypeScript"],
    "frameworks": ["Spring Boot", "FastAPI", "Node.js"],
    "cloud": ["AWS", "Azure", "GCP"],
    "devops": ["Docker", "Kubernetes", "Terraform"]
  }
}
```

**7. Update your master resume**
- Open `templates/resume_template.html`
- Fill in your experience, education, and projects
- Upload it to your Google Drive master resume doc

**8. Run it!**
- Trigger manually first to test
- Verify 10 jobs produce 10 different resumes
- Schedule for daily runs

---

## 📁 Project Structure

```
job-automation-pipeline/
├── README.md
├── config/
│   └── candidate_profile.example.json   # Your profile config (copy & fill)
├── workflows/
│   └── n8n_workflow_template.json        # Import into n8n
├── nodes/
│   ├── remove_duplicates.js              # Filter staffing companies
│   ├── parser.js                         # Parse Claude JSON response
│   └── clean_output.js                  # Extract HTML from Claude response
├── prompts/
│   ├── check_relevance.md               # Claude relevance filter prompt
│   └── customise_resume.md              # Claude resume customization prompt
└── templates/
    └── resume_template.html             # ATS-optimized resume HTML template
```

---

## 🧠 Node Code

### Remove Duplicates + Filter Staffing Companies

```javascript
const scrapedJobs = $input.all();

// Add or remove companies as needed
const staffingCompanies = [
  'infosys', 'wipro', 'tcs', 'tata consultancy', 'cognizant',
  'hcl', 'capgemini', 'hexaware', 'mastech', 'tekystems',
  'apex systems', 'insight global', 'robert half', 'randstad',
  'beaconfire', 'beacon hill', 'kforce', 'modis', 'stefanini',
  'luxoft', 'epam', 'globallogic', 'mphasis', 'persistent systems',
  'ltimindtree', 'mindtree', 'revature', 'collabera'
];

return scrapedJobs.filter(item => {
  const company = (item.json.companyName || '').toLowerCase();
  return !staffingCompanies.some(bad => company.includes(bad));
});
```

### Parse Claude Response

```javascript
const items = $input.all();
const results = [];

for (const item of items) {
  try {
    const text = item.json.content[0].text;
    const cleaned = text.replace(/```json|```/g, '').trim();
    const parsed = JSON.parse(cleaned);
    const verdict = parsed.verdict === true || parsed.verdict === "true" ? "true" : "false";

    results.push({
      json: {
        verdict,
        sponsorshipMentioned: parsed.sponsorshipMentioned ?? 'Not Mentioned',
        reason: parsed.reasonForVerdict ?? 'N/A'
      }
    });
  } catch(e) {
    results.push({
      json: {
        verdict: "false",
        sponsorshipMentioned: "Not Mentioned",
        reason: "parse error",
        error: e.message
      }
    });
  }
}

return results;
```

### Clean HTML Output

```javascript
const items = $input.all();
const results = [];

for (const item of items) {
  try {
    let html = '';
    if (item.json.content && Array.isArray(item.json.content)) {
      html = item.json.content[0].text;
    } else if (item.json.text) {
      html = item.json.text;
    }

    // Extract only the HTML part
    const htmlStart = html.indexOf('<html');
    if (htmlStart !== -1) html = html.substring(htmlStart);

    results.push({ json: { resumeHtml: html } });
  } catch(e) {
    results.push({
      json: {
        resumeHtml: '<html><body>Error generating resume</body></html>',
        error: e.message
      }
    });
  }
}

return results;
```

---

## 🤖 Claude Prompts

### Check Relevance Prompt

```
You are a technical recruiter. Return raw JSON only.

REJECT (verdict false):
- Job outside USA
- Requires US citizenship, green card, permanent residency, or clearance
- Senior, Staff, Principal, Lead, Director, VP roles
- Internship only
- Empty or too-short description
- PhD required
- 7+ years required

PASS (verdict true):
- Entry, Associate, Mid-level, Junior roles (0-6 years)
- Remote / hybrid / onsite all fine
- No sponsorship mention = fine (assume possible)
- Transferable skills count heavily
- If role mentions ANY: Java, Python, microservices, AWS, Azure,
  Kubernetes, REST APIs → AUTO PASS

SPONSORSHIP:
- Yes = H1B / OPT / CPT / F1 / will sponsor
- No = citizenship / GC / clearance required
- Not Mentioned = everything else

RULE: Be VERY inclusive. Maximize realistic opportunities. When in doubt, PASS.

Return JSON only (no backticks, no markdown):
{"verdict": "true or false", "sponsorshipMentioned": "Yes / No / Not Mentioned", "reasonForVerdict": "one sentence"}
```

### Customize Resume Prompt

```
Customize this resume for the job below.
Integrate missing ATS keywords naturally.
Keep human engineer tone.
Cut irrelevant content. Keep only what's relevant.

Rules:
- Rewrite bullets using context-setting style
- Integrate top missing ATS keywords naturally
- Quantified achievements where possible
- Complete 2-page resume, never truncate
- HTML only, no backticks, start with <html>
- Include ONLY skills from job description (20-30 max)

Job: {{ $json.title }} at {{ $json.companyName }}

Job Description:
{{ $json.descriptionText }}

My Resume:
[PASTE YOUR MASTER RESUME CONTENT HERE]
```

> ⚠️ **Important:** Use `{{ $json.title }}` NOT `{{ $('Scrape jobs').first().json.title }}` — the `.first()` always grabs the first job, not the current loop item!

---

## 📋 Google Sheets Tracking Columns

| Column | Description |
|---|---|
| Status | Not Applied / In Progress / Applied / Skipped |
| Job Post Link | LinkedIn URL |
| Job Title | Role name |
| Seniority Level | Entry / Mid / Senior |
| Posted At | When job was posted |
| Company Name | Employer |
| Company Website | Company URL |
| Salary | If listed |
| Description | Full job description |
| No. Applicants | Competition level |
| Resume URL | Google Doc link |
| Application URL | Direct apply link |
| Sponsorship | Yes / No / Not Mentioned |
| Verdict Reason | Why Claude passed/rejected |
| Date Added | When scraped |
| Applied By | Team member name |
| Last Updated | Last status change |
| Notes | Free text |

---

## ⚠️ Known Issues & Fixes

### Loop Processing Same Job Repeatedly

**Symptom:** All generated resumes are identical

**Root Cause:** Claude prompts using `.first()` which always grabs the first job

```javascript
// ❌ Wrong — always grabs first job
{{ $('Scrape jobs').first().json.title }}

// ✅ Correct — grabs current loop item
{{ $json.title }}
```

### Google Docs Margins via API

**Status:** Currently using CSS margins instead of API

```css
body { margin: 0.7in; }
```

---

## 💰 Cost Breakdown

| Component | Cost |
|---|---|
| Claude Haiku (per resume) | ~$0.009 |
| Apify LinkedIn Scraper | Free tier (limited) |
| n8n (self-hosted) | Free |
| Google Workspace | Free |
| **Monthly total (40 resumes/day)** | **~$11–15** |

---

## 🗺️ Roadmap

- [x] LinkedIn job scraping
- [x] AI relevance filtering
- [x] Resume customization
- [x] Google Docs generation
- [x] Google Sheets tracking
- [ ] Fix loop variable bug
- [ ] Scheduled daily runs
- [ ] Cover letter generation
- [ ] Indeed + Glassdoor support
- [ ] Interview outcome tracking
- [ ] ML model for better filtering

---

## 🤝 Contributing

Contributions welcome! This is designed to be useful for any job seeker, not just software engineers.

To adapt for your profile:
1. Update `config/candidate_profile.example.json` with your skills
2. Update the resume template with your experience
3. Adjust the Claude prompts for your target roles
4. Update the staffing company filter list as needed

---

## 📄 License

MIT License — free to use, adapt, and share.

---

## 👨‍💻 Author

Built by [Haneesh Nellore](https://github.com/Haneesh-Nellore) — Senior Software Engineer transitioning into AI/LLM engineering.

- 📧 nellorehaneesh123@gmail.com
- 💼 [LinkedIn](https://linkedin.com/in/haneeshnellore)
- 🔧 [aws-serverless-patterns](https://github.com/Haneesh-Nellore/aws-serverless-patterns)
- 🩺 [VitalAI](https://github.com/Haneesh-Nellore/VitalAI)
