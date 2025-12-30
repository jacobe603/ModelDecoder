# AAON Model String Parser - Technical Architecture

This document describes the architecture of the Model String Parser web application, enabling implementation of additional model code sets (e.g., different HVAC equipment lines).

---

## Table of Contents

1. [Overview](#overview)
2. [Data Structures](#data-structures)
3. [Core Functions](#core-functions)
4. [UI Components](#ui-components)
5. [Adding New Model Types](#adding-new-model-types)
6. [Position Mapping System](#position-mapping-system)
7. [Dependency Validation](#dependency-validation)
8. [Extension Points](#extension-points)

---

## Overview

The application is a single-file HTML application with:
- **Inline CSS** (~640 lines) - All styling
- **Inline JavaScript** (~1400 lines) - All logic and data
- **No external dependencies** - Pure vanilla JS, no frameworks

### Key Features
1. **Parse Mode**: Decode a model string into human-readable components
2. **Reverse Search**: Find codes by searching descriptions
3. **Position Visualization**: Highlight which characters in the model string correspond to each code
4. **Dependency Validation**: Warn when code combinations are invalid
5. **Capacity Data Enhancement**: Dynamic data lookup based on related codes

---

## Data Structures

### 1. Code Definitions (`codeDefinitions`)

The primary data structure mapping category codes to their descriptions.

```javascript
const codeDefinitions = {
    CATEGORY_KEY: {
        name: "Human Readable Name",      // Display name for the category
        position: "POS",                   // Position identifier (shown in UI)
        group: "GroupName",                // Grouping for navigation (e.g., "Model", "Cooling", "Heating", "Features")
        codes: {
            "0": "Description for code 0",
            "A": "Description for code A",
            "B": "Description for code B",
            // ... all valid codes for this category
        }
    },
    // ... more categories
};
```

**Example:**
```javascript
VOLTAGE: {
    name: "Voltage",
    position: "VLT",
    group: "Model",
    codes: {
        "1": "230V/1Φ/60Hz",
        "2": "230V/3Φ/60Hz",
        "3": "460V/3Φ/60Hz",
        "8": "208V/3Φ/60Hz"
    }
}
```

### 2. Navigation Structure (`navStructure`, `featureNavStructure`)

Defines the sidebar navigation order and hierarchy.

```javascript
const navStructure = [
    { key: 'CATEGORY_KEY', name: 'Display Name', indent: false },
    { key: 'SUBCATEGORY', name: 'Subcategory Name', indent: true },
    // ...
];
```

- `key`: Must match a key in `codeDefinitions`
- `name`: Display name in sidebar
- `indent`: Whether to indent (for visual hierarchy)

### 3. Position Map (`positionMap`)

Maps each category to character positions in the model string.

```javascript
const positionMap = {
    'CATEGORY_KEY': [startIndex, endIndex],  // endIndex is exclusive
    'GEN': [0, 2],      // Characters 0-1 (e.g., "RN")
    'SIZE': [3, 6],     // Characters 3-5 (e.g., "025")
    'VOLTAGE': [7, 8],  // Character 7 (e.g., "3")
    // ...
};
```

**Important**: Indices are based on the full model string including separators (`-` and `:`).

### 4. Reference Model String (`REFERENCE_MODEL_STRING`)

Used for position visualization in reverse search.

```javascript
const REFERENCE_MODEL_STRING = 'RN-025-3-0-BB02-384:A000-D0B-DEH-0BA-0D0000L-00-00B00000B';
```

### 5. Dependency Rules (`dependencyRules`)

Defines validation rules for code interdependencies.

```javascript
const dependencyRules = [
    {
        condition: {
            category: 'CATEGORY_A',
            codes: ['0', '1'],           // Trigger when CATEGORY_A is 0 or 1
            and: {                        // Optional: compound condition
                category: 'CATEGORY_B',
                codes: ['X']
            }
        },
        affects: 'CATEGORY_C',            // Category that must follow rules
        validCodes: ['A', 'B', 'C'],       // Only these codes are valid
        message: 'Explanation of the rule',
        hint: 'Suggested fix',
        info: false,                      // true = informational only, no validation
        enables: 'CATEGORY_D'             // Optional: indicates feature enablement
    }
];
```

### 6. Supplemental Data (e.g., `heatingCapacityData`)

Additional lookup tables for dynamic data enhancement.

```javascript
const heatingCapacityData = {
    "1": { gasInput: 60.0, gasOutput: 48.6, electric208: 7.5, electricOther: 10 },
    "2": { gasInput: 90.0, gasOutput: 72.0, electric208: 15.0, electricOther: 20 },
    // ...
};
```

---

## Core Functions

### Parsing Functions

#### `parseModelString()`
Main entry point for parsing. Called on input change.

```javascript
function parseModelString() {
    const input = document.getElementById('model-input').value.toUpperCase().trim();

    // 1. Normalize input (handle different dash types)
    const normalized = input.replace(/[–—]/g, '-');

    // 2. Split into model part and feature part (separated by ':')
    const [modelPart, featurePart] = normalized.split(':');

    // 3. Parse each segment and build results array
    const results = [];
    const modelSegments = modelPart.split('-').filter(s => s);

    // 4. Parse model segments (GEN, SIZE, VOLTAGE, etc.)
    if (modelSegments.length >= 1) {
        results.push({ category: 'GEN', code: modelSegments[0], ...lookup('GEN', modelSegments[0]) });
    }
    // ... continue for each segment

    // 5. Parse feature segments (after the colon)
    if (featurePart) {
        const featureSegments = featurePart.split('-').filter(s => s);
        // Parse each feature segment...
    }

    // 6. Enhance results (e.g., add capacity data)
    enhanceHeatingCapacity(results);

    // 7. Validate dependencies
    const warnings = validateDependencies(results);

    // 8. Render UI
    renderResults(results, warningCategories);
    updateParseVisualization();
}
```

#### `lookup(category, code)`
Retrieves description for a code.

```javascript
function lookup(category, code) {
    const def = codeDefinitions[category];
    if (!def) return { name: 'Unknown', description: 'Unknown category', position: category };

    return {
        name: def.name,
        position: def.position,
        description: def.codes[code] || 'Unknown code'
    };
}
```

### Search Functions

#### `performSearch()`
Full-text search across all codes.

```javascript
function performSearch() {
    const query = document.getElementById('search-input').value.toLowerCase().trim();
    const matches = [];

    for (const [catKey, catDef] of Object.entries(codeDefinitions)) {
        for (const [code, description] of Object.entries(catDef.codes)) {
            if (description.toLowerCase().includes(query) ||
                catDef.name.toLowerCase().includes(query)) {
                matches.push({
                    category: catKey,
                    position: catDef.position,
                    name: catDef.name,
                    code: code,
                    description: description
                });
            }
        }
    }

    // Render results and visualization
}
```

### Visualization Functions

#### `buildModelStringVisualization(highlightCategories)`
Creates HTML with highlighted positions.

```javascript
function buildModelStringVisualization(highlightCategories) {
    const str = REFERENCE_MODEL_STRING; // or current input
    const highlightPositions = new Set();

    // Collect positions to highlight
    highlightCategories.forEach(cat => {
        const pos = positionMap[cat];
        if (pos) {
            for (let i = pos[0]; i < pos[1]; i++) {
                highlightPositions.add(i);
            }
        }
    });

    // Build HTML
    let html = '';
    for (let i = 0; i < str.length; i++) {
        const char = str[i];
        if (highlightPositions.has(i)) {
            html += `<span class="highlight">${char}</span>`;
        } else if (char === '-' || char === ':') {
            html += `<span class="separator">${char}</span>`;
        } else {
            html += char;
        }
    }
    return html;
}
```

### Validation Functions

#### `validateDependencies(results)`
Checks all dependency rules and returns warnings.

```javascript
function validateDependencies(results) {
    const warnings = [];
    const codeMap = {};

    // Build lookup map
    results.forEach(r => codeMap[r.category] = r.code);

    // Check each rule
    dependencyRules.forEach(rule => {
        // Check if condition is met
        if (!codeMap[rule.condition.category] ||
            !rule.condition.codes.includes(codeMap[rule.condition.category])) {
            return;
        }

        // Check compound condition if present
        if (rule.condition.and) {
            // ... validate compound condition
        }

        // Check affected category
        if (rule.affects && codeMap[rule.affects]) {
            if (!rule.validCodes.includes(codeMap[rule.affects])) {
                warnings.push({
                    type: 'warning',
                    message: rule.message,
                    hint: rule.hint,
                    // ... other details
                });
            }
        }
    });

    return warnings;
}
```

### Enhancement Functions

#### `enhanceHeatingCapacity(results)`
Adds dynamic data to results based on related codes.

```javascript
function enhanceHeatingCapacity(results) {
    const b1Result = results.find(r => r.category === 'B1');
    const b2Result = results.find(r => r.category === 'B2');
    const voltageResult = results.find(r => r.category === 'VOLTAGE');

    // Determine heating type and voltage
    const isElectric = b1Result?.code === '1';
    const isGas = ['2', '3', '4', '5', '6', '7', '8', '9'].includes(b1Result?.code);

    // Look up capacity and append to description
    const capacityData = heatingCapacityData[b2Result?.code];
    if (capacityData && isGas) {
        b2Result.description += ` — ${capacityData.gasInput} MBH Input`;
    }
}
```

---

## UI Components

### HTML Structure

```html
<div class="app-container">
    <!-- Title Bar -->
    <div class="title-bar">...</div>

    <!-- Model String Input -->
    <div class="model-string-bar">
        <input id="model-input" oninput="parseModelString()">
        <div class="examples-row"><!-- Example buttons --></div>
        <div class="model-string-viz"><!-- Position visualization --></div>
    </div>

    <!-- Tab Navigation -->
    <div class="tabs-bar">
        <button onclick="switchTab('parse')">Parse Model String</button>
        <button onclick="switchTab('search')">Reverse Search</button>
    </div>

    <!-- Main Content -->
    <div class="main-content">
        <!-- Parse Panel -->
        <div class="parse-panel" id="parse-panel">
            <div class="sidebar"><!-- Navigation --></div>
            <div class="results-panel">
                <div class="validation-panel"><!-- Warnings --></div>
                <table class="results-table"><!-- Results --></table>
            </div>
        </div>

        <!-- Search Panel -->
        <div class="search-panel" id="search-panel">
            <input id="search-input" oninput="debouncedSearch()">
            <div class="model-string-viz"><!-- Position visualization --></div>
            <div class="search-results"><!-- Results --></div>
        </div>
    </div>

    <!-- Status Bar -->
    <div class="status-bar">...</div>
</div>
```

### Key CSS Classes

| Class | Purpose |
|-------|---------|
| `.model-string-viz-code` | Monospace display for model string |
| `.highlight` | Blue background for highlighted characters |
| `.separator` | Muted color for `-` and `:` characters |
| `.results-table` | Main results table styling |
| `.selected-row` | Blue outline for selected row |
| `.has-warning` | Yellow background for warning rows |
| `.validation-panel` | Yellow warning panel |
| `.options-list` | Expandable available options |

---

## Adding New Model Types

### Step 1: Create Model-Specific Data

Create separate data objects for each model type:

```javascript
// Model Type A (existing RN Series)
const codeDefinitions_RN = { ... };
const navStructure_RN = [ ... ];
const featureNavStructure_RN = [ ... ];
const positionMap_RN = { ... };
const dependencyRules_RN = [ ... ];
const REFERENCE_MODEL_STRING_RN = '...';

// Model Type B (new model series)
const codeDefinitions_XY = { ... };
const navStructure_XY = [ ... ];
const featureNavStructure_XY = [ ... ];
const positionMap_XY = { ... };
const dependencyRules_XY = [ ... ];
const REFERENCE_MODEL_STRING_XY = '...';
```

### Step 2: Add Model Type Selector UI

Add a dropdown before the input:

```html
<div class="model-type-selector">
    <label for="model-type">Model Series:</label>
    <select id="model-type" onchange="switchModelType(this.value)">
        <option value="RN">RN Series (Rooftop Units)</option>
        <option value="XY">XY Series (Other Equipment)</option>
    </select>
</div>
```

### Step 3: Create Switching Function

```javascript
// Active configuration references
let codeDefinitions = codeDefinitions_RN;
let navStructure = navStructure_RN;
let featureNavStructure = featureNavStructure_RN;
let positionMap = positionMap_RN;
let dependencyRules = dependencyRules_RN;
let REFERENCE_MODEL_STRING = REFERENCE_MODEL_STRING_RN;

function switchModelType(modelType) {
    switch(modelType) {
        case 'RN':
            codeDefinitions = codeDefinitions_RN;
            navStructure = navStructure_RN;
            featureNavStructure = featureNavStructure_RN;
            positionMap = positionMap_RN;
            dependencyRules = dependencyRules_RN;
            REFERENCE_MODEL_STRING = REFERENCE_MODEL_STRING_RN;
            break;
        case 'XY':
            codeDefinitions = codeDefinitions_XY;
            navStructure = navStructure_XY;
            featureNavStructure = featureNavStructure_XY;
            positionMap = positionMap_XY;
            dependencyRules = dependencyRules_XY;
            REFERENCE_MODEL_STRING = REFERENCE_MODEL_STRING_XY;
            break;
    }

    // Clear current results and rebuild UI
    document.getElementById('model-input').value = '';
    buildSidebar([]);
    document.getElementById('results-body').innerHTML = '';
    updateParseVisualization();

    // Update example buttons dynamically
    updateExampleButtons(modelType);
}
```

### Step 4: Update Parse Logic for Different Formats

If different model types have different string formats:

```javascript
function parseModelString() {
    const modelType = document.getElementById('model-type').value;

    switch(modelType) {
        case 'RN':
            parseRNModelString(input);
            break;
        case 'XY':
            parseXYModelString(input);
            break;
    }
}

function parseRNModelString(input) {
    // RN format: RN-025-3-0-BB02-384:A000-D0B-DEH-0BA-0D0000L-00-00B00000B
    const [modelPart, featurePart] = input.split(':');
    // ... existing parsing logic
}

function parseXYModelString(input) {
    // XY format: XY-100-A-B-C-D (different structure)
    const segments = input.split('-');
    // ... custom parsing logic for this format
}
```

---

## Position Mapping System

### How Position Mapping Works

The `positionMap` maps each category to character indices in the full model string:

```
Model String: R N - 0 2 5 - 3 - 0 - B B 0 2 - 3 8 4 : A 0 0 0 - ...
Index:        0 1 2 3 4 5 6 7 8 9 10 11 12 13 14 15 16 17 18 19 20 21 22 23 ...

positionMap:
  'GEN':     [0, 2]    → "RN" (indices 0-1)
  'SIZE':    [3, 6]    → "025" (indices 3-5)
  'VOLTAGE': [7, 8]    → "3" (index 7)
  'INTERPROT': [9, 10] → "0" (index 9)
  'A1':      [11, 12]  → "B" (index 11)
  'A2':      [12, 13]  → "B" (index 12)
  ...
```

### Creating Position Maps for New Models

1. Write out the full reference model string
2. Mark each character's index (including separators)
3. Identify start/end indices for each category
4. The end index is exclusive (like JavaScript slice)

---

## Dependency Validation

### Rule Types

1. **Simple Restriction**: If A=X, then B must be one of [Y, Z]
2. **Compound Condition**: If A=X AND C=W, then B must be one of [Y, Z]
3. **Informational**: If A=X, show info message (no validation)
4. **Feature Enablement**: If A=X, then feature B becomes applicable

### Rule Structure

```javascript
{
    condition: {
        category: 'B1',           // Category to check
        codes: ['0'],             // Codes that trigger this rule
        and: {                    // Optional compound condition
            category: 'A2',
            codes: ['6', '7']
        }
    },
    affects: 'B2',                // Category that's constrained
    validCodes: ['0'],            // Only valid codes when rule applies
    message: 'No Heating (B1=0) requires B2 to be 0',
    hint: 'B2 should be 0 when there is no heating',
    info: false                   // true = info only, false = validation
}
```

---

## Extension Points

### Adding New Features

1. **New Data Enhancement**: Follow `enhanceHeatingCapacity()` pattern
2. **New Validation Rules**: Add to `dependencyRules` array
3. **New UI Column**: Update `renderResults()` and table header
4. **New Search Filters**: Modify `performSearch()` logic

### Recommended Refactoring for Multi-Model Support

```javascript
// Configuration object pattern
const ModelConfigs = {
    RN: {
        name: 'RN Series - Rooftop Units',
        codeDefinitions: { ... },
        navStructure: [ ... ],
        featureNavStructure: [ ... ],
        positionMap: { ... },
        dependencyRules: [ ... ],
        referenceString: '...',
        parseFunction: parseRNModelString,
        examples: [
            { label: '25T Gas Heat', value: '...' },
            { label: '8T ERU', value: '...' }
        ]
    },
    XY: {
        name: 'XY Series - Other Equipment',
        // ... same structure
    }
};

let activeConfig = ModelConfigs.RN;

function switchModelType(type) {
    activeConfig = ModelConfigs[type];
    rebuildUI();
}
```

---

## File Structure (Current)

```
/ModelDecoder
├── index.html          # Complete single-file application
├── ARCHITECTURE.md     # This documentation
└── .git/               # Git repository
```

### Recommended Structure for Multi-Model

```
/ModelDecoder
├── index.html              # Main application shell
├── css/
│   └── styles.css          # Extracted styles
├── js/
│   ├── app.js              # Core application logic
│   ├── parser.js           # Parsing functions
│   ├── search.js           # Search functions
│   ├── validation.js       # Validation logic
│   └── visualization.js    # Position highlighting
├── data/
│   ├── rn-series.js        # RN Series definitions
│   ├── xy-series.js        # XY Series definitions
│   └── shared.js           # Shared utilities
├── ARCHITECTURE.md         # This documentation
└── README.md               # User documentation
```

---

## Summary

To implement a new model code set:

1. **Define the data**: `codeDefinitions`, `navStructure`, `positionMap`
2. **Create position map**: Map each category to string indices
3. **Write parsing logic**: Handle the specific string format
4. **Add dependency rules**: Define valid code combinations
5. **Add enhancement data**: Any supplemental lookup tables
6. **Create UI selector**: Dropdown to switch between models
7. **Wire up switching**: Function to swap active configuration

The core rendering, search, and visualization code can remain unchanged - only the data and parsing logic need to be customized per model type.
