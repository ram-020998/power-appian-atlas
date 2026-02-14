# Appian Atlas Powers

> **Note**: This repository has been split into separate persona-specific repositories. Please use the links below to install the power that matches your role.

## Available Powers

### 🔧 Developer
**Repository**: [power-appian-atlas-developer](https://github.com/ram-020998/power-appian-atlas-developer)  
**Focus**: Technical implementation details

For developers who need:
- Full technical object names and UUIDs
- SAIL code and implementation details
- Dependency graphs and architecture
- Technical debt identification
- Performance optimization insights

**Install URL**:
```
https://github.com/ram-020998/power-appian-atlas-developer
```

---

### 📋 Product Owner
**Repository**: [power-appian-atlas-product-owner](https://github.com/ram-020998/power-appian-atlas-product-owner)  
**Focus**: Business capabilities and workflows

For product owners who need:
- Business-friendly language (no technical jargon)
- Feature descriptions and user capabilities
- Workflow explanations
- Business value and impact analysis
- Stakeholder-ready documentation

**Install URL**:
```
https://github.com/ram-020998/power-appian-atlas-product-owner
```

---

### 🎨 UX Designer
**Repository**: [power-appian-atlas-ux-designer](https://github.com/ram-020998/power-appian-atlas-ux-designer)  
**Focus**: User interfaces and interaction patterns

For UX designers who need:
- Interface layouts and SAIL components
- User flows and journeys
- UI patterns and consistency analysis
- Accessibility evaluation
- Design improvement suggestions

**Install URL**:
```
https://github.com/ram-020998/power-appian-atlas-ux-designer
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
   pip install "appian-atlas @ git+https://github.com/ram-020998/gam-knowledge-base.git#subdirectory=mcp_server"
   ```

2. **Set GitHub token** (for accessing parsed application data) - See [gam-knowledge-base](https://github.com/ram-020998/gam-knowledge-base) for details

3. **Choose and install a power** (see URLs above)

## Installation

1. Open the **Powers** tab in Kiro IDE
2. Click **Add Power**
3. Paste the URL for your role (see above)
4. Configure your GitHub token in the Power Config

## Related Repositories

- **appian-parser**: Core parsing engine that converts Appian packages to structured JSON
  - https://github.com/ram-020998/appian-parser

- **gam-knowledge-base**: Data repository with parsed applications and MCP server
  - https://github.com/ram-020998/gam-knowledge-base

## License

MIT License - See individual repositories for details.
