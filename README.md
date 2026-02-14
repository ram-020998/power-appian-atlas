# Appian Atlas Powers

Persona-specific Kiro Powers for exploring Appian applications. Each power provides the same MCP tools but with tailored steering instructions for different roles.

## Available Powers

### 🔧 Developer
**Path**: `developer/`  
**Focus**: Technical implementation details

For developers who need:
- Full technical object names and UUIDs
- SAIL code and implementation details
- Dependency graphs and architecture
- Technical debt identification
- Performance optimization insights

**Install**:
```bash
kiro power add https://github.com/ram-020998/power-appian-atlas/developer
```

---

### 📋 Product Owner
**Path**: `product-owner/`  
**Focus**: Business capabilities and workflows

For product owners who need:
- Business-friendly language (no technical jargon)
- Feature descriptions and user capabilities
- Workflow explanations
- Business value and impact analysis
- Stakeholder-ready documentation

**Install**:
```bash
kiro power add https://github.com/ram-020998/power-appian-atlas/product-owner
```

---

### 🎨 UX Designer
**Path**: `ux-designer/`  
**Focus**: User interfaces and interaction patterns

For UX designers who need:
- Interface layouts and SAIL components
- User flows and journeys
- UI patterns and consistency analysis
- Accessibility evaluation
- Design improvement suggestions

**Install**:
```bash
kiro power add https://github.com/ram-020998/power-appian-atlas/ux-designer
```

---

## How It Works

All three powers use the same **appian-atlas** MCP server, which provides tools to explore parsed Appian applications. The difference is in how information is presented:

| Aspect | Developer | Product Owner | UX Designer |
|--------|-----------|---------------|-------------|
| **Language** | Technical terms, object names | Business language | UI/UX terminology |
| **Detail Level** | Full implementation details | High-level capabilities | Interface-focused |
| **Code Display** | Always show SAIL code | Hide unless asked | Show UI-relevant SAIL |
| **Focus** | How it works | What it does | How users interact |
| **Identifiers** | Show UUIDs and technical names | Abstract technical details | Show interface names |

## Prerequisites

1. **Install the MCP server**:
   ```bash
   pip install "appian-atlas @ git+https://github.com/ram-020998/gam-knowledge-base.git"
   ```

2. **Set GitHub token** (for accessing parsed application data):
   ```bash
   export GITHUB_TOKEN="your_github_token"
   ```

3. **Choose and install a power** (see above)

## Example Usage

### Developer
```
User: "How does the Add Vendors action work?"
Response: "The Add Vendors action (Record Action) triggers process model AS_GSS_PM_addVendors.
          Entry form: AS_GSS_FM_addVendors (UUID: _a-0000e30a...)
          Calls: AS_GSS_BL_validateVendors for validation
          [Shows full SAIL code and dependency graph]"
```

### Product Owner
```
User: "How does the Add Vendors action work?"
Response: "The Add Vendors feature allows procurement teams to add new vendors.
          Users click 'Add Vendors' → fill form → submit → vendor is added.
          System validates vendor name is unique and sends notifications.
          This streamlines vendor management and maintains data quality."
```

### UX Designer
```
User: "How does the Add Vendors action work?"
Response: "The Add Vendors form uses a standard form layout:
          - Text field for vendor name (real-time validation)
          - Email field with format validation
          - Dropdown for category selection
          - Text area for notes
          - Primary 'Add Vendor' button and secondary 'Cancel' button
          [Shows SAIL UI components and user flow diagram]"
```

## Repository Structure

```
power-appian-atlas/
├── README.md                    # This file
├── developer/
│   ├── POWER.md                 # Developer steering instructions
│   └── mcp.json                 # MCP server configuration
├── product-owner/
│   ├── POWER.md                 # Product Owner steering instructions
│   └── mcp.json                 # MCP server configuration
├── ux-designer/
│   ├── POWER.md                 # UX Designer steering instructions
│   └── mcp.json                 # MCP server configuration
└── POWER.md                     # Legacy (kept for backward compatibility)
```

## Contributing

To add a new persona or improve existing ones:

1. Fork this repository
2. Create or modify the persona directory
3. Update `POWER.md` with persona-specific steering instructions
4. Test with various queries to ensure appropriate responses
5. Submit a pull request

## Related Repositories

- **appian-parser**: Core parsing engine that converts Appian packages to structured JSON
  - https://github.com/ram-020998/appian-parser

- **gam-knowledge-base**: Data repository with parsed applications and MCP server
  - https://github.com/ram-020998/gam-knowledge-base

## License

MIT License - See individual repositories for details.
