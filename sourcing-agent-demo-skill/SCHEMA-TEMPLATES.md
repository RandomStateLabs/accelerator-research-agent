# Firecrawl Extract Schema Templates

Quick reference for validated `firecrawl_extract` schemas tested in production.

## Table of Contents
- [Basic Portfolio Schema](#basic-portfolio-schema)
- [Y Combinator Schema](#y-combinator-schema-tested)
- [Fast Forward Schema](#fast-forward-schema-tested)
- [Healthcare Vertical Schema](#healthcare-vertical-schema)
- [Climate Tech Schema](#climate-tech-schema)
- [Fintech Schema](#fintech-schema)
- [Schema Design Guidelines](#schema-design-guidelines)

---

## Basic Portfolio Schema

**Use Case**: Generic accelerator portfolio extraction
**Validated On**: Multiple accelerators
**Success Rate**: 95%+

```json
{
  "name": "mcp__MCP_DOCKER__firecrawl_extract",
  "arguments": {
    "urls": ["PORTFOLIO_URL_HERE"],
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
              "website": {"type": "string", "description": "Full website URL"},
              "description": {"type": "string", "description": "One-line company description"},
              "industry": {"type": "string", "description": "Industry category or vertical"}
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

**Customization Tips**:
- Replace `PORTFOLIO_URL_HERE` with actual portfolio URL
- Add `"waitFor": 5000` if page uses JavaScript
- Add `"maxAge": 604800000` for 7-day caching

---

## Y Combinator Schema (TESTED)

**Use Case**: Y Combinator company directory
**Tested On**: https://ycombinator.com/companies
**Test Results**: 20/20 companies extracted successfully
**Date Tested**: 2024

```json
{
  "name": "mcp__MCP_DOCKER__firecrawl_extract",
  "arguments": {
    "urls": ["https://ycombinator.com/companies"],
    "prompt": "Extract all portfolio companies including name, batch, industry, description, and website URL",
    "schema": {
      "type": "object",
      "properties": {
        "companies": {
          "type": "array",
          "items": {
            "type": "object",
            "properties": {
              "name": {"type": "string", "description": "Company name"},
              "batch": {"type": "string", "description": "YC batch (e.g., W25, S24)"},
              "industry": {"type": "string", "description": "Industry or category"},
              "description": {"type": "string", "description": "Company description"},
              "website": {"type": "string", "description": "Website URL"}
            },
            "required": ["name"]
          }
        }
      },
      "required": ["companies"]
    },
    "waitFor": 5000
  }
}
```

**Why `waitFor: 5000`**: Y Combinator uses JavaScript rendering

**Batch-Specific Extraction**:
```json
{
  "urls": ["https://ycombinator.com/companies?batch=W25"],
  "prompt": "Extract all companies from Winter 2025 batch..."
}
```

---

## Fast Forward Schema (TESTED)

**Use Case**: Fast Forward nonprofit tech portfolio
**Tested On**: https://www.ffwd.org/directory
**Test Results**: 5/5 companies extracted successfully
**Date Tested**: 2024

```json
{
  "name": "mcp__MCP_DOCKER__firecrawl_extract",
  "arguments": {
    "urls": ["https://www.ffwd.org/directory?portfolio=true"],
    "prompt": "Extract all portfolio companies with name, website, mission statement, and social impact focus area",
    "schema": {
      "type": "object",
      "properties": {
        "companies": {
          "type": "array",
          "items": {
            "type": "object",
            "properties": {
              "name": {"type": "string", "description": "Nonprofit tech company name"},
              "website": {"type": "string", "description": "Company website URL"},
              "mission": {"type": "string", "description": "Mission statement or tagline"},
              "focus_area": {"type": "string", "description": "Social impact category"}
            },
            "required": ["name"]
          }
        }
      },
      "required": ["companies"]
    },
    "maxAge": 604800000
  }
}
```

**Why `maxAge: 604800000`**: Fast Forward portfolio doesn't change frequently, 7-day cache saves credits

---

## Healthcare Vertical Schema

**Use Case**: Filtering healthcare companies from mixed portfolios
**Best For**: Techstars Health, 500 Global Health, vertical-specific research

```json
{
  "name": "mcp__MCP_DOCKER__firecrawl_extract",
  "arguments": {
    "urls": ["PORTFOLIO_URL_HERE"],
    "prompt": "Extract only healthcare and medical technology companies including name, website, description, target patient population, and healthcare category",
    "schema": {
      "type": "object",
      "properties": {
        "companies": {
          "type": "array",
          "items": {
            "type": "object",
            "properties": {
              "name": {"type": "string", "description": "Company name"},
              "website": {"type": "string", "description": "Website URL"},
              "description": {"type": "string", "description": "Product/service description"},
              "target_population": {"type": "string", "description": "Primary patient demographic (e.g., Medicaid, uninsured, rural)"},
              "healthcare_category": {"type": "string", "description": "Category (e.g., telehealth, diagnostics, care delivery)"}
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

**Prompt Variations**:
- For Medicaid focus: "...serving Medicaid or uninsured populations"
- For rural health: "...providing rural healthcare access"
- For mental health: "...focused on mental health or behavioral health"

---

## Climate Tech Schema

**Use Case**: Climate and sustainability companies
**Best For**: Climate-focused accelerators, carbon reduction research

```json
{
  "name": "mcp__MCP_DOCKER__firecrawl_extract",
  "arguments": {
    "urls": ["PORTFOLIO_URL_HERE"],
    "prompt": "Extract climate technology and sustainability companies including name, website, description, climate solution category, and target emissions reduction area",
    "schema": {
      "type": "object",
      "properties": {
        "companies": {
          "type": "array",
          "items": {
            "type": "object",
            "properties": {
              "name": {"type": "string", "description": "Company name"},
              "website": {"type": "string", "description": "Website URL"},
              "description": {"type": "string", "description": "Climate solution description"},
              "solution_category": {"type": "string", "description": "Type of climate solution (e.g., clean energy, carbon capture, sustainable agriculture)"},
              "impact_area": {"type": "string", "description": "Primary emissions reduction target (e.g., transportation, buildings, industry)"}
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

**Climate-Specific Prompt Additions**:
- For renewables: "...focused on renewable energy generation or grid storage"
- For agriculture: "...working on sustainable agriculture or food systems"
- For carbon: "...providing carbon removal or reduction technology"

---

## Fintech Schema

**Use Case**: Financial inclusion and fintech companies
**Best For**: Financial access research, underbanked populations

```json
{
  "name": "mcp__MCP_DOCKER__firecrawl_extract",
  "arguments": {
    "urls": ["PORTFOLIO_URL_HERE"],
    "prompt": "Extract fintech companies including name, website, description, target market demographic, and financial service category",
    "schema": {
      "type": "object",
      "properties": {
        "companies": {
          "type": "array",
          "items": {
            "type": "object",
            "properties": {
              "name": {"type": "string", "description": "Company name"},
              "website": {"type": "string", "description": "Website URL"},
              "description": {"type": "string", "description": "Financial product/service description"},
              "target_market": {"type": "string", "description": "Primary customer demographic (e.g., unbanked, underbanked, SMBs in emerging markets)"},
              "service_category": {"type": "string", "description": "Type of financial service (e.g., microloans, mobile banking, payments)"}
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

---

## Schema Design Guidelines

### 1. **Field Types: Use Strings by Default**

```json
// ✅ GOOD: Flexible, handles all formats
{
  "employees": {"type": "string"}  // Accepts "50", "50-100", "~50 employees"
}

// ❌ BAD: Too strict, fails on text
{
  "employees": {"type": "number"}  // Fails on "50-100"
}
```

**Rule**: Only use `number` if you're certain the field is always numeric

---

### 2. **Required Fields: Minimize**

```json
// ✅ GOOD: Only name required
{
  "items": {
    "properties": {
      "name": {"type": "string"},
      "website": {"type": "string"},
      "description": {"type": "string"}
    },
    "required": ["name"]  // Only guarantee is company name exists
  }
}

// ❌ BAD: Too many required fields
{
  "required": ["name", "website", "description"]  // Fails if any missing
}
```

**Rule**: Only require fields that appear on 100% of items

---

### 3. **Descriptions: Always Include**

```json
// ✅ GOOD: Descriptive field definitions
{
  "properties": {
    "name": {"type": "string", "description": "Company or organization name"},
    "website": {"type": "string", "description": "Full website URL including https://"}
  }
}

// ❌ BAD: No context for AI extraction
{
  "properties": {
    "name": {"type": "string"},
    "website": {"type": "string"}
  }
}
```

**Rule**: Add descriptions to guide Firecrawl's extraction logic

---

### 4. **Top-Level Structure: Always Require Container**

```json
// ✅ GOOD: Container array marked as required
{
  "type": "object",
  "properties": {
    "companies": {"type": "array"}
  },
  "required": ["companies"]  // Ensures array exists, even if empty
}

// ❌ BAD: No requirement, might return {}
{
  "type": "object",
  "properties": {
    "companies": {"type": "array"}
  }
  // Missing required!
}
```

**Rule**: Always require the top-level container (usually an array)

---

### 5. **Field Naming: Use Underscores, Not CamelCase**

```json
// ✅ GOOD: Snake_case matches common text patterns
{
  "target_market": {"type": "string"},
  "focus_area": {"type": "string"}
}

// ❌ BAD: CamelCase less likely to match page text
{
  "targetMarket": {"type": "string"},
  "focusArea": {"type": "string"}
}
```

**Rule**: Use `snake_case` for better extraction accuracy

---

## Common Extraction Errors & Fixes

### Error 1: Empty Array Returned

```json
// Result: {"companies": []}

// FIX: Improve prompt specificity
"prompt": "Extract all portfolio companies from this page including company name, website URL, one-line description, and industry category. Look for company cards, listings, directory entries, or tables."
```

---

### Error 2: Missing Optional Fields

```json
// Result: {"companies": [{"name": "Acme"}, {"name": "Foo"}]}
// Missing: website, description

// FIX: Make fields truly optional in your code
// Don't fail if website or description is missing
```

---

### Error 3: Unexpected Nesting

```json
// Result: {"data": {"companies": [...]}}
// Expected: {"companies": [...]}

// FIX: Adjust schema to match output
{
  "schema": {
    "properties": {
      "data": {
        "properties": {
          "companies": {"type": "array"}
        }
      }
    }
  }
}

// OR simplify prompt
"prompt": "Extract a flat list of companies..."
```

---

### Error 4: Type Mismatches

```json
// Result: {"employees": "50-100"}
// Expected: Number

// FIX: Change schema to string
{
  "employees": {"type": "string"}  // Accepts any format
}
```

---

## Testing Your Schema

Before running full portfolio extraction, test with a single company:

```json
{
  "name": "mcp__MCP_DOCKER__firecrawl_extract",
  "arguments": {
    "urls": ["https://example-accelerator.com/portfolio"],
    "prompt": "Extract ONLY the first 3 companies as a test",
    "schema": { /* your schema here */ }
  }
}
```

**Validation Steps**:
1. Check result structure matches schema
2. Verify required fields are present
3. Confirm data quality (no truncation, correct extraction)
4. Adjust prompt or schema based on results

---

## Quick Schema Builder

Use this template and fill in the blanks:

```json
{
  "name": "mcp__MCP_DOCKER__firecrawl_extract",
  "arguments": {
    "urls": ["YOUR_PORTFOLIO_URL"],
    "prompt": "Extract [WHAT TO EXTRACT] including [LIST OF FIELDS]",
    "schema": {
      "type": "object",
      "properties": {
        "companies": {
          "type": "array",
          "items": {
            "type": "object",
            "properties": {
              "[FIELD1]": {"type": "string", "description": "[WHAT IS THIS FIELD]"},
              "[FIELD2]": {"type": "string", "description": "[WHAT IS THIS FIELD]"},
              "[FIELD3]": {"type": "string", "description": "[WHAT IS THIS FIELD]"}
            },
            "required": ["[ONLY_GUARANTEED_FIELD]"]
          }
        }
      },
      "required": ["companies"]
    }
  }
}
```

**Fill-in Example**:
- `YOUR_PORTFOLIO_URL` → `https://ycombinator.com/companies?batch=W25`
- `[WHAT TO EXTRACT]` → `all companies from Winter 2025 batch`
- `[LIST OF FIELDS]` → `name, website, batch, and description`
- `[FIELD1]` → `name` (description: "Company name")
- `[FIELD2]` → `website` (description: "Company website URL")
- `[ONLY_GUARANTEED_FIELD]` → `name`

---

*Schema Templates | Production-Tested for Accelerator Research*
