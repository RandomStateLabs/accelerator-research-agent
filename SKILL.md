---
name: accelerator-research-agent
description: Research accelerator portfolio companies using Firecrawl and Tavily MCPs. Generates structured CSV and markdown reports with systematic impact scoring. Pure research workflow - outputs data only, no tracking system integration.
---

# Accelerator Research Agent

A Claude Desktop skill for researching accelerator portfolio companies with systematic impact analysis using proven MCP tools.

## When to Use This Skill

Activate this skill when:
- User asks to **"research companies from [accelerator name]"**
- User wants to **"analyze [accelerator] portfolio"**
- User mentions accelerator names: YC, Techstars, Fast Forward, 500 Global, a16z, Sequoia Arc
- User needs to **"score companies for impact"** or **"evaluate mission alignment"**
- User requests **"scrape [accelerator] portfolio"** or **"generate company reports"**

## Prerequisites - Required MCP Servers

This skill uses TWO MCP servers that have been tested and proven effective:

### 1. Firecrawl MCP (Required)
**Purpose**: AI-native web scraping for JavaScript-heavy portfolio pages
**Why Critical**: Returns LLM-ready markdown, handles React/Next.js automatically
**Free Tier**: 500 credits/month
**Paid Tier**: $30/month starter (recommended for production)

### 2. Tavily MCP (Required)
**Purpose**: AI-optimized search for company research and discovery
**Why Critical**: 100 RPM free tier, citation-backed results, built for LLMs
**Free Tier**: 100 requests/minute (6,000/hour!)
**Paid Tier**: $50/month for scale

**Setup Guide**: See `references/mcp-setup-guide.md` for detailed MCP configuration.

## Core Research Workflow (3 Phases)

This workflow generates CSV and markdown reports ONLY. It does NOT create tracking issues or manage pipelines.

### Phase 1: Portfolio Scraping

**Goal**: Extract complete company list from accelerator portfolio page

**Primary Tool**: `firecrawl_extract` (Validated: 100% success rate in live testing)

**Fallback Tool**: `firecrawl_scrape` (if extract doesn't work for specific sites)

**Why Firecrawl**: Accelerator sites are static HTML/JavaScript pages. Firecrawl handles JavaScript rendering automatically without browser automation complexity.

---

#### Phase 1A: Structured Extraction (PREFERRED METHOD)

**Tool**: `firecrawl_extract`

**Why This is Primary**: Live testing validated 100% success rate with structured schemas on Y Combinator (20/20 companies) and Fast Forward (5/5 companies). Returns structured JSON directly instead of markdown requiring manual parsing.

**Best Practice MCP Call**:
```json
{
  "name": "mcp__MCP_DOCKER__firecrawl_extract",
  "arguments": {
    "urls": ["https://www.ffwd.org/directory?portfolio=true"],
    "prompt": "Extract all portfolio companies including company name, website URL, one-line description, and industry vertical",
    "schema": {
      "type": "object",
      "properties": {
        "companies": {
          "type": "array",
          "items": {
            "type": "object",
            "properties": {
              "name": {"type": "string", "description": "Company name"},
              "website": {"type": "string", "description": "Company website URL"},
              "description": {"type": "string", "description": "One-line description"},
              "industry": {"type": "string", "description": "Industry vertical or category"}
            },
            "required": ["name"]
          }
        }
      },
      "required": ["companies"]
    }
  }
}
```

**Parameters Explained**:
- `urls` - Array of URLs to extract from (can pass multiple portfolio pages)
- `prompt` - Natural language instruction for what to extract
- `schema` - JSON Schema defining exact output structure
- `required` - Ensures critical fields are always present

**Output**: Structured JSON array ready for Phase 2 research
```json
{
  "companies": [
    {
      "name": "Noora Health",
      "website": "https://noorahealth.org",
      "description": "Training family caregivers for hospital patients",
      "industry": "Healthcare"
    },
    {
      "name": "Tala",
      "website": "https://tala.co",
      "description": "Microloans for unbanked populations",
      "industry": "Fintech"
    }
  ]
}
```

**Validated Schemas from Testing**:

**Y Combinator Portfolio** (tested with 20 companies):
```json
{
  "schema": {
    "type": "object",
    "properties": {
      "companies": {
        "type": "array",
        "items": {
          "type": "object",
          "properties": {
            "name": {"type": "string"},
            "batch": {"type": "string"},
            "industry": {"type": "string"},
            "description": {"type": "string"},
            "website": {"type": "string"}
          }
        }
      }
    }
  }
}
```

**Fast Forward Portfolio** (tested with 5 companies):
```json
{
  "schema": {
    "type": "object",
    "properties": {
      "companies": {
        "type": "array",
        "items": {
          "type": "object",
          "properties": {
            "name": {"type": "string"},
            "website": {"type": "string"},
            "mission": {"type": "string"},
            "focus_area": {"type": "string"}
          }
        }
      }
    }
  }
}
```

**More Schema Templates**: See `SCHEMA-TEMPLATES.md` for ready-to-use schemas for:
- Healthcare/Medicaid-focused companies
- Climate tech and sustainability
- Financial inclusion and fintech
- Vertical-specific extraction patterns
- Common troubleshooting examples

---

#### Phase 1B: Markdown Extraction (FALLBACK METHOD)

**Tool**: `firecrawl_scrape`

**When to Use**:
- `firecrawl_extract` fails for a specific site structure
- You need full page context for manual parsing
- Portfolio page has non-standard layout

**Best Practice MCP Call**:
```json
{
  "name": "mcp__MCP_DOCKER__firecrawl_scrape",
  "arguments": {
    "url": "https://www.ffwd.org/directory?portfolio=true",
    "formats": ["markdown"],
    "onlyMainContent": true,
    "maxAge": 604800000
  }
}
```

**Parameters Explained**:
- `formats: ["markdown"]` - Returns clean markdown (not raw HTML)
- `onlyMainContent: true` - Strips navigation, footer, ads
- `maxAge: 604800000` - 7-day cache (saves credits on re-runs)

**Output**: Clean markdown requiring manual parsing

---

#### Phase 1C: Multi-Page Discovery

**For Multi-Page Portfolios**:

If portfolio has pagination, use `firecrawl_map` first to discover all pages:

```json
{
  "name": "mcp__MCP_DOCKER__firecrawl_map",
  "arguments": {
    "url": "https://ycombinator.com/companies",
    "search": "portfolio",
    "limit": 100
  }
}
```

Then pass all discovered URLs to `firecrawl_extract` as an array:
```json
{
  "name": "mcp__MCP_DOCKER__firecrawl_extract",
  "arguments": {
    "urls": [
      "https://ycombinator.com/companies?batch=W25",
      "https://ycombinator.com/companies?batch=S24",
      "https://ycombinator.com/companies?batch=W24"
    ],
    "prompt": "Extract companies from each batch page",
    "schema": { /* same schema as above */ }
  }
}
```

---

### Phase 2: Company Research

**Goal**: Deep research on each company to gather founder info, mission, metrics

**Tool**: `tavily-search` (primary), `tavily-extract` (fallback)

**Why Tavily**:
- 100 requests/minute free tier (can research 100 companies/hour)
- Returns citation-backed, LLM-ready data
- Aggregates multiple sources automatically
- Built specifically for AI research workflows

**Research Questions to Answer**:
- What problem does this company solve?
- Who are the founders and their backgrounds?
- What is their target market/primary beneficiaries?
- Do they serve underserved communities? (for impact scoring)
- Key metrics: users, revenue, funding raised, employee count

**Best Practice MCP Call** (per company):
```json
{
  "name": "mcp__MCP_DOCKER__tavily-search",
  "arguments": {
    "query": "[Company Name] founders mission target market impact metrics",
    "max_results": 5,
    "search_depth": "advanced",
    "include_raw_content": true
  }
}
```

**Parameters Explained**:
- `max_results: 5` - Top 5 most relevant sources (balances depth vs tokens)
- `search_depth: "advanced"` - More comprehensive results (use "basic" for speed)
- `include_raw_content: true` - Get full page text, not just snippets

**Batch Processing Strategy**:
- Research 5-10 companies at a time
- Allow user to review results before continuing
- Prevents context overflow on large portfolios (50+ companies)

**Fallback Option**:

If you already have company website URLs, use `tavily-extract`:

```json
{
  "name": "mcp__MCP_DOCKER__tavily-extract",
  "arguments": {
    "urls": ["https://noorahealth.org", "https://tala.co"],
    "extract_depth": "advanced"
  }
}
```

**Output**: Structured research data with founder names, mission statements, target markets, metrics

---

### Phase 3: Impact Scoring & Report Generation

**Goal**: Apply systematic impact rubric and generate deliverable reports

**Tool**: None (pure analysis + formatting)

#### Impact Scoring Framework

**5-Tier Impact Rubric** (Default: Low-income US impact)

**⭐⭐⭐⭐⭐ Tier 1 - Direct Impact**
- Primary beneficiaries ARE underserved populations
- Product designed exclusively/primarily for them
- Impact is central to business model
- Clear measurement of reach to target communities

**Example**: Noora Health (trains family caregivers for low-income patients in India)

**⭐⭐⭐⭐ Tier 2 - Strong Indirect Impact**
- Serves broad market including significant underserved segment (30-70%)
- Creates economic opportunities in underserved areas
- Removes barriers to essential services

**Example**: Watsi (crowdfunded healthcare for many low-income patients)

**⭐⭐⭐ Tier 3 - Moderate Systemic Impact**
- Addresses root causes affecting underserved communities
- Environmental/sustainability with distributional benefits
- Enables nonprofits/governments to serve underserved

**Example**: Pivot Bio (sustainable agriculture reduces costs for all farmers)

**⭐⭐ Tier 4 - Tangential Impact**
- Vague social good mission without targeting
- Freemium model but unclear if underserved use it
- Could benefit underserved, not designed for them

**Example**: Duolingo (free language learning, not targeted to underserved)

**⭐ Tier 5 - Minimal Alignment**
- Purely commercial focus
- Target market is affluent/enterprise
- No social mission

**Example**: Peloton ($1,500 bike + $40/month subscription)

**Scoring Process**:
1. Read company research notes
2. Identify **primary beneficiaries** (who actually uses this?)
3. Assess **directness of impact pathway**:
   - Direct: Company → Product → Underserved User → Benefit
   - Indirect: Company → Product → Intermediary → Underserved User
4. Assign tier with **2-3 sentence reasoning**

**Full rubric with examples**: See `references/impact-scoring.md`

#### Report Generation

**CSV Output Format**:
```csv
Company Name,Website,Founded,Founders,Mission,Impact Tier,Impact Reasoning,Target Market,Employees,Funding Stage,Funding Amount
Noora Health,noorahealth.org,2014,"Shahed Alam, Edith Elliott",Train family caregivers for hospital patients,1,"Direct service to low-income patients who can't afford professional post-discharge care. Operates exclusively in safety-net hospitals.",Low-income patients India,150,Series B,$23M
Tala,tala.co,2011,Shivani Siroya,Microloans for unbanked populations,1,"Primary beneficiaries are financially excluded individuals. Product specifically designed for those traditional banks won't serve.",Emerging markets unbanked,400,Series D,$350M
```

**Markdown Report Format**:
```markdown
# [Accelerator Name] Research Report

**Generated**: [YYYY-MM-DD]
**Total Companies Researched**: [X]
**Research Methodology**: Firecrawl scraping + Tavily deep search

## Impact Distribution

- ⭐⭐⭐⭐⭐ Tier 1 (Direct Impact): [X] companies ([X]%)
- ⭐⭐⭐⭐ Tier 2 (Strong Indirect): [X] companies ([X]%)
- ⭐⭐⭐ Tier 3 (Moderate Systemic): [X] companies ([X]%)
- ⭐⭐ Tier 4 (Tangential): [X] companies ([X]%)
- ⭐ Tier 5 (Minimal): [X] companies ([X]%)

## Tier 1 Companies (Highest Impact)

### [Company Name]
**Website**: [URL]
**Founded**: [Year]
**Founders**: [Names]
**Mission**: [One-line description]
**Target Market**: [Primary beneficiaries]
**Impact Score**: ⭐⭐⭐⭐⭐ Tier 1

**Why Tier 1**: [2-3 sentence reasoning explaining why primary beneficiaries are underserved populations and impact is direct]

**Key Metrics**:
- Employees: [X]
- Funding: [Stage], $[Amount]
- Users/Reach: [X if available]

---

[Repeat for each Tier 1 company]

## Tier 2 Companies (Strong Impact)

[Same format as Tier 1]

## Research Methodology

**Data Collection**:
- Portfolio extraction: Firecrawl MCP (AI-native scraping)
- Company research: Tavily MCP (100 RPM free tier, citation-backed)

**Analysis Framework**:
- 5-tier impact rubric focused on low-income US populations
- Evaluation criteria: Primary beneficiaries, directness of impact pathway, measurement

**Limitations**:
- Research based on publicly available information
- Impact scores are analytical judgments, not quantitative measurements
- May not reflect companies that have pivoted since last public update
```

**File Naming Convention**:
- CSV: `[accelerator-name]-portfolio-[YYYY-MM-DD].csv`
- Markdown: `[accelerator-name]-research-report-[YYYY-MM-DD].md`

---

## Example Usage Scenarios

### Scenario 1: Basic Research (10 companies)

**User Request**:
```
"Research 10 climate tech companies from YC W25 batch"
```

**Your Workflow**:
1. **Extract**: `firecrawl_extract` on YC W25 portfolio page with climate filter
   ```json
   {
     "name": "mcp__MCP_DOCKER__firecrawl_extract",
     "arguments": {
       "urls": ["https://ycombinator.com/companies?batch=W25"],
       "prompt": "Extract all climate tech companies including name, website, batch, description, and industry",
       "schema": {
         "type": "object",
         "properties": {
           "companies": {
             "type": "array",
             "items": {
               "type": "object",
               "properties": {
                 "name": {"type": "string"},
                 "website": {"type": "string"},
                 "batch": {"type": "string"},
                 "description": {"type": "string"},
                 "industry": {"type": "string"}
               },
               "required": ["name"]
             }
           }
         }
       }
     }
   }
   ```
2. **Filter**: Parse JSON output, select first 10 climate tech companies
3. **Research**: `tavily-search` on each company (10 searches)
4. **Score**: Apply climate tech impact rubric (Tier 1 = direct emissions reduction)
5. **Report**: Generate CSV + markdown with top-tier companies highlighted

**Time Estimate**: 2-3 minutes for 10 companies
**API Cost**: $0 (free tiers)
**Why Extract**: Returns structured JSON, no manual parsing needed

---

### Scenario 2: Full Portfolio Analysis (50+ companies)

**User Request**:
```
"Analyze the complete Fast Forward 2024 portfolio with impact scoring"
```

**Your Workflow**:
1. **Discover**: `firecrawl_map` to find all portfolio pages (if multi-page)
   ```json
   {
     "name": "mcp__MCP_DOCKER__firecrawl_map",
     "arguments": {
       "url": "https://www.ffwd.org/directory",
       "search": "portfolio",
       "limit": 100
     }
   }
   ```
2. **Extract**: `firecrawl_extract` on discovered pages with structured schema
   ```json
   {
     "name": "mcp__MCP_DOCKER__firecrawl_extract",
     "arguments": {
       "urls": ["https://www.ffwd.org/directory?portfolio=true"],
       "prompt": "Extract all portfolio companies with name, website, mission, and focus area",
       "schema": {
         "type": "object",
         "properties": {
           "companies": {
             "type": "array",
             "items": {
               "type": "object",
               "properties": {
                 "name": {"type": "string"},
                 "website": {"type": "string"},
                 "mission": {"type": "string"},
                 "focus_area": {"type": "string"}
               },
               "required": ["name"]
             }
           }
         }
       }
     }
   }
   ```
3. **Research**: `tavily-search` in batches of 10 companies
   - Batch 1: Research companies 1-10, present results
   - Batch 2: Research companies 11-20, present results
   - Continue until complete
4. **Score**: Apply impact rubric to all companies
5. **Report**: Generate comprehensive CSV + markdown

**Time Estimate**: 10-15 minutes for 50 companies
**API Cost**: ~$5-10 if using paid tiers
**Why Extract**: Structured JSON output scales better for large portfolios

---

### Scenario 3: Vertical-Focused Research

**User Request**:
```
"Find healthcare companies from Techstars 2024 serving Medicaid populations"
```

**Your Workflow**:
1. **Extract**: `firecrawl_extract` on Techstars 2024 portfolio with healthcare filter
   ```json
   {
     "name": "mcp__MCP_DOCKER__firecrawl_extract",
     "arguments": {
       "urls": ["https://www.techstars.com/portfolio"],
       "prompt": "Extract healthcare companies from 2024 cohort including name, website, description, and target patient population",
       "schema": {
         "type": "object",
         "properties": {
           "companies": {
             "type": "array",
             "items": {
               "type": "object",
               "properties": {
                 "name": {"type": "string"},
                 "website": {"type": "string"},
                 "description": {"type": "string"},
                 "target_population": {"type": "string"}
               },
               "required": ["name"]
             }
           }
         }
       }
     }
   }
   ```
2. **Filter**: From JSON output, identify healthcare companies
3. **Deep Research**: `tavily-search` with queries like:
   - "[Company] Medicaid uninsured patient focus"
   - "[Company] healthcare access low-income"
4. **Score**: Apply healthcare access rubric (Tier 1 = Medicaid/uninsured focus)
5. **Report**: Generate filtered report showing only Tier 1-2 companies

---

## Tool-Specific Best Practices

### Firecrawl Best Practices

**Priority Hierarchy** (use in this order):
1. ✅ **`firecrawl_extract`** - PRIMARY (100% success rate in testing)
2. ✅ **`firecrawl_scrape`** - Fallback (when extract doesn't work)
3. ✅ **`firecrawl_map`** - Discovery (for multi-page portfolios)
4. ❌ **`firecrawl_crawl`** - AVOID (expensive, overkill)

**When to Use `firecrawl_extract`** (PRIMARY):
- ✅ Portfolio pages with structured company data
- ✅ When you need JSON output with specific fields
- ✅ Tested and validated on YC, Fast Forward, Techstars
- ✅ Eliminates manual parsing errors
- ✅ Same credit cost as scrape, but better output

**Schema Design Best Practices**:
```json
{
  "schema": {
    "type": "object",
    "properties": {
      "companies": {
        "type": "array",
        "items": {
          "type": "object",
          "properties": {
            "name": {"type": "string", "description": "Required: Company name"},
            "website": {"type": "string", "description": "Optional: Full URL"},
            "description": {"type": "string", "description": "Optional: One-liner"}
          },
          "required": ["name"]  // Only require fields that always exist
        }
      }
    },
    "required": ["companies"]
  }
}
```

**Extract vs Scrape Decision Tree**:
```
Do you need structured data (JSON)?
  ├─ YES → Use firecrawl_extract with schema
  └─ NO  → Need full page markdown?
           ├─ YES → Use firecrawl_scrape
           └─ NO  → Use tavily-extract as fallback
```

**When to Use `firecrawl_scrape`** (FALLBACK):
- ✅ Extract failed for specific site structure
- ✅ Need full page context for manual parsing
- ✅ Portfolio has non-standard layout
- ✅ 1 credit per scrape, fast and efficient

**When to Use `firecrawl_map`** (DISCOVERY):
- ✅ Multi-page portfolios with pagination
- ✅ Unknown site structure (need to find all URLs)
- ✅ Discovering company profile URLs before extraction

**When to Use `firecrawl_crawl`**:
- ❌ **Avoid for accelerator research** (expensive, overkill)
- Only use if portfolio requires deep multi-level scraping across domains

**Error Handling**:
- If `firecrawl_extract` returns empty/partial data:
  1. First: Add more descriptive prompt ("Extract companies with name, website URL, and description")
  2. Second: Simplify schema (remove optional fields, keep only required)
  3. Third: Fallback to `firecrawl_scrape` with manual parsing
- If `firecrawl_scrape` returns empty content:
  1. Add `waitFor: 3000` parameter (wait 3 seconds for JavaScript)
  2. Increase to `waitFor: 5000` if still empty
  3. Final fallback: `tavily-extract` on same URL

---

### Tavily Best Practices

**When to Use `tavily-search`**:
- ✅ Company research (primary use case)
- ✅ Finding founder backgrounds
- ✅ Discovering recent news/funding
- ✅ Multi-source validation

**When to Use `tavily-extract`**:
- ✅ You already have company website URLs
- ✅ Need clean content from specific pages
- ✅ Fallback when Firecrawl fails

**When to Use `tavily-crawl`**:
- ❌ **Not recommended for this workflow**
- Use `tavily-search` instead (faster, more relevant)

**Query Optimization**:
- Good: "Noora Health founders mission target market impact"
- Better: "Noora Health Shahed Alam low-income patients India healthcare"
- Include specific keywords that indicate impact focus

**Rate Limits**:
- Free tier: 100 RPM (6,000/hour)
- Can research 100 companies/hour for FREE
- If exceeding limits, use `search_depth: "basic"` for speed

---

## Token Management & Efficiency

### Reducing Token Usage

1. **Batch Processing**: Research 5-10 companies, review, continue
2. **Firecrawl**: Always use `onlyMainContent: true`
3. **Tavily**: Use `max_results: 5` (not 10 or 20)
4. **Cache**: Leverage `maxAge` to reuse Firecrawl results

### Context Management

**If reaching token limits**:
- Reduce batch size from 10 to 5 companies
- Use `search_depth: "basic"` instead of "advanced"
- Generate report in stages (Tier 1 first, then Tier 2-3)

---

## Customizing the Impact Rubric

The default rubric focuses on **low-income US impact**. Adapt for different theses:

### Climate Tech Rubric

- **Tier 1**: Direct emissions reduction with measurable CO2 impact
- **Tier 2**: Clean energy infrastructure (solar, wind, grid storage)
- **Tier 3**: Circular economy, sustainable materials
- **Tier 4**: Sustainability marketing, vague "green" positioning
- **Tier 5**: Greenwashing, no real climate impact

### Healthcare Access Rubric

- **Tier 1**: Medicaid/uninsured patient focus
- **Tier 2**: Rural healthcare delivery, telemedicine for underserved
- **Tier 3**: Provider cost reduction (hospitals, clinics)
- **Tier 4**: General telehealth, no specific access focus
- **Tier 5**: Luxury/concierge medicine ($5K+/year memberships)

### Financial Inclusion Rubric

- **Tier 1**: Unbanked/underbanked populations (microloans, mobile money)
- **Tier 2**: Small business lending in emerging markets
- **Tier 3**: Financial literacy tools, accessible banking
- **Tier 4**: General fintech, no specific inclusion focus
- **Tier 5**: High-net-worth services, wealth management

See `references/impact-scoring.md` for detailed customization guide with examples.

---

## Output Specifications

### What This Skill Generates

- ✅ **CSV file** with all company data (Excel/Google Sheets compatible)
- ✅ **Markdown research report** with detailed analysis
- ✅ **Impact tier distribution** summary
- ✅ **Documented scoring rationale** (2-3 sentences per company)

### What This Skill Does NOT Do

- ❌ Create tracking issues (use separate Linear skill)
- ❌ Integrate with CRM/Airtable
- ❌ Send email alerts or notifications
- ❌ Manage deal pipeline or follow-ups
- ❌ Use Coresignal for enrichment (too expensive, doesn't cover new startups)
- ❌ Use Playwright for scraping (overkill, Firecrawl handles JavaScript)

### Next Steps After Report Generation

1. **Review** CSV and markdown reports
2. **Import** CSV to your preferred system:
   - Airtable for database view
   - Google Sheets for collaboration
   - Linear (via separate tracking skill)
3. **Filter** to Tier 1-2 companies for outreach
4. **Customize** impact rubric for next research run

---

## Troubleshooting

### "MCP server not found"
**Problem**: Claude Desktop can't find Firecrawl or Tavily MCP servers

**Solutions**:
1. Check `claude_desktop_config.json` has both MCPs configured
2. Restart Claude Desktop completely (quit and reopen)
3. Verify Docker Desktop is running (if using Docker MCP)
4. Test individual MCP with simple query: `mcp__MCP_DOCKER__firecrawl_scrape` on google.com

---

### `firecrawl_extract` Returns Empty/Partial Data

**Problem**: Extract function returns `{"companies": []}` or missing fields

**Solution Sequence** (try in order):

**Step 1: Improve the Prompt**
```json
// ❌ Bad: Vague prompt
"prompt": "Extract companies"

// ✅ Good: Specific, detailed prompt
"prompt": "Extract all portfolio companies from this page including company name, website URL, one-line description, and industry category. Look for company cards, listings, or directory entries."
```

**Step 2: Simplify the Schema**
```json
// ❌ Bad: Too many required fields
{
  "schema": {
    "properties": {
      "companies": {
        "items": {
          "properties": {
            "name": {"type": "string"},
            "website": {"type": "string"},
            "description": {"type": "string"}
          },
          "required": ["name", "website", "description"]  // Too strict!
        }
      }
    }
  }
}

// ✅ Good: Only require what always exists
{
  "schema": {
    "properties": {
      "companies": {
        "items": {
          "properties": {
            "name": {"type": "string"},
            "website": {"type": "string"},
            "description": {"type": "string"}
          },
          "required": ["name"]  // Only name is guaranteed
        }
      }
    }
  }
}
```

**Step 3: Add waitFor for JavaScript Pages**
```json
// Some portfolio pages load via JavaScript
{
  "name": "mcp__MCP_DOCKER__firecrawl_extract",
  "arguments": {
    "urls": ["https://ycombinator.com/companies"],
    "prompt": "Extract companies...",
    "schema": { /* ... */ },
    "waitFor": 5000  // Wait 5 seconds for page to fully load
  }
}
```

**Step 4: Fallback to Scrape**
If extract still fails, use `firecrawl_scrape` and parse manually:
```json
{
  "name": "mcp__MCP_DOCKER__firecrawl_scrape",
  "arguments": {
    "url": "https://example.com/portfolio",
    "formats": ["markdown"],
    "onlyMainContent": true
  }
}
```

---

### Firecrawl Returns Empty Content (Scrape Function)

**Problem**: `firecrawl_scrape` returns blank or minimal content

**Solutions**:
1. Add `waitFor: 3000` parameter (wait for JavaScript to load)
2. Increase to `waitFor: 5000` if still empty
3. Check `onlyMainContent: true` isn't stripping too much
4. Final fallback: Use `tavily-extract` on same URL

---

### Extract Returns Wrong Data Structure

**Problem**: JSON doesn't match expected schema or has unexpected nesting

**Cause**: Firecrawl AI interpreted page structure differently than expected

**Solution**: Adjust schema to match actual output
```json
// If you get: {"data": {"companies": [...]}}
// Instead of: {"companies": [...]}

// Adjust schema to:
{
  "schema": {
    "type": "object",
    "properties": {
      "data": {
        "type": "object",
        "properties": {
          "companies": {
            "type": "array",
            // ...
          }
        }
      }
    }
  }
}
```

Or simplify prompt: "Extract a flat array of companies"

---

### Tavily Rate Limit Hit

**Problem**: Error message "Rate limit exceeded"

**Details**: Free tier = 100 requests/minute (6,000/hour)

**Solutions**:
1. **Short term**: Add 1-second delay between searches
2. **Medium term**: Use `search_depth: "basic"` instead of "advanced" (faster, uses fewer resources)
3. **Long term**: Upgrade to paid tier ($50/month for unlimited within higher rate limits)

---

### Companies Missing Key Data

**Problem**: Some companies lack founder info, metrics, or descriptions

**Causes**:
- Stealth startups with minimal public presence
- Early-stage companies without press coverage
- Companies that pivoted recently

**Solutions**:
1. Mark as "Insufficient public data" in report
2. Use `tavily-extract` directly on company website (if URL available)
3. Try alternative search queries:
   - "[Company] founder LinkedIn"
   - "[Company] Crunchbase profile"
   - "[Company] AngelList"
4. Recommend manual research for high-priority Tier 1 candidates

---

### Schema Validation Errors

**Problem**: "Invalid schema" or extraction fails silently

**Common Issues**:

**Issue 1: Missing Required Top-Level Property**
```json
// ❌ Bad: No required array
{
  "schema": {
    "type": "object",
    "properties": {
      "companies": {"type": "array"}
    }
    // Missing: "required": ["companies"]
  }
}

// ✅ Good: Explicitly require companies array
{
  "schema": {
    "type": "object",
    "properties": {
      "companies": {"type": "array"}
    },
    "required": ["companies"]
  }
}
```

**Issue 2: Inconsistent Field Names**
```json
// ❌ Bad: Using camelCase when page has spaces
{
  "properties": {
    "companyName": {"type": "string"}  // Page shows "Company Name"
  }
}

// ✅ Good: Match page structure or use flexible names
{
  "properties": {
    "name": {"type": "string"}
  }
}
```

**Issue 3: Over-Specified Types**
```json
// ❌ Bad: Expecting number when page shows text
{
  "properties": {
    "employees": {"type": "number"}  // Page shows "50-100"
  }
}

// ✅ Good: Use string for ambiguous data
{
  "properties": {
    "employees": {"type": "string"}  // Accepts "50-100" or "50"
  }
}
```

---

## Cost Analysis

### Free Tier Research (0-500 companies/month)

**Tools**:
- Firecrawl: 500 credits free
- Tavily: 100 RPM (6,000/hour) free

**Cost**: $0/month

**Limitations**:
- 500 Firecrawl scrapes (enough for 10-20 accelerator portfolios)
- Unlimited Tavily searches within rate limits

---

### Paid Tier Research (500+ companies/month)

**Tools**:
- Firecrawl Starter: $30/month (5,000 credits)
- Tavily Pro: $50/month (unlimited within higher rate limits)

**Cost**: $80/month total

**What You Get**:
- 5,000 Firecrawl scrapes (100+ accelerator portfolios)
- Unlimited Tavily searches (higher rate limits)
- Production-ready for scale

---

## Validation Checklist

Before running research, confirm:

### MCP Configuration
- [ ] Firecrawl MCP configured in `claude_desktop_config.json`
- [ ] Tavily MCP configured in `claude_desktop_config.json`
- [ ] Docker Desktop running (if using Docker MCP deployment)
- [ ] Both MCPs tested with simple queries

### MCP Test Commands (Run These First)
```json
// Test 1: Firecrawl Extract (PRIMARY)
{
  "name": "mcp__MCP_DOCKER__firecrawl_extract",
  "arguments": {
    "urls": ["https://example.com"],
    "prompt": "Extract the main heading",
    "schema": {
      "type": "object",
      "properties": {
        "heading": {"type": "string"}
      }
    }
  }
}
// Expected: Should return {"heading": "Example Domain"}

// Test 2: Tavily Search
{
  "name": "mcp__MCP_DOCKER__tavily-search",
  "arguments": {
    "query": "Y Combinator",
    "max_results": 3
  }
}
// Expected: Should return 3 search results about Y Combinator
```

### Schema Preparation
- [ ] Schema uses validated patterns from Phase 1A (Y Combinator or Fast Forward schemas)
- [ ] Only `name` field is marked as required (other fields optional)
- [ ] Schema descriptions added for clarity
- [ ] Top-level `companies` array is marked as required

### Research Configuration
- [ ] Impact rubric reviewed and adapted for thesis (if needed)
- [ ] Batch size determined (recommend 5-10 companies for context management)
- [ ] Output format confirmed (CSV + markdown)
- [ ] Target accelerator URL verified (portfolio page exists)

### Performance Optimization
- [ ] `maxAge: 604800000` added to Firecrawl calls (7-day cache)
- [ ] `search_depth: "basic"` or "advanced" chosen based on need
- [ ] Batch processing strategy planned (10 companies at a time for 50+ portfolios)

---

## Learning Resources

- **Schema Templates**: `SCHEMA-TEMPLATES.md` - Ready-to-use extraction schemas
- **MCP Setup Guide**: `references/mcp-setup-guide.md` (if exists)
- **Impact Scoring Framework**: `references/impact-scoring.md` (if exists)
- **Tool Reference**: `references/tool-reference.md` (if exists)
- **Example Reports**: `references/example-outputs.md` (if exists)

---

## Quick Start Command

**Copy-paste this to test your setup**:
```json
{
  "name": "mcp__MCP_DOCKER__firecrawl_extract",
  "arguments": {
    "urls": ["https://www.ffwd.org/directory?portfolio=true"],
    "prompt": "Extract the first 3 portfolio companies including name, website, and mission",
    "schema": {
      "type": "object",
      "properties": {
        "companies": {
          "type": "array",
          "items": {
            "type": "object",
            "properties": {
              "name": {"type": "string"},
              "website": {"type": "string"},
              "mission": {"type": "string"}
            },
            "required": ["name"]
          }
        }
      },
      "required": ["companies"]
    }
  }
}
```

**Expected Result**: JSON with 3 companies from Fast Forward portfolio

---

*Accelerator Research Agent | Production-Ready with Firecrawl Extract + Tavily Search*
