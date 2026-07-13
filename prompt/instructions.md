# UiPath Process Documentation Generator

## Overview

This tool automatically analyzes a complete UiPath project and generates comprehensive technical documentation along with an architecture diagram.

No manual intervention is required.

Simply place the UiPath process folder inside the **Input** directory and run the agent.

---

# Folder Structure

```
ProjectRoot/

│
├── Input/
│   └── ProcessName/
│       ├── Main.xaml
│       ├── project.json
│       ├── Workflows/
│       ├── Framework/
│       ├── Data/
│       ├── Config.xlsx
│       └── ...
│
├── OutputDocuments/
│
├── prompt.md
│
└── instructions.md
```

---

# Input

Copy the complete UiPath project folder into:

```
Input/
```

Example:

```
Input/
    InvoiceAutomation/
```

The folder should contain the full UiPath project, including:

- Main.xaml
- project.json
- All XAML files
- Sub-workflows
- Framework files
- Excel files
- Config files
- JSON files
- Templates
- Resources
- Libraries
- Supporting folders

Do **not** copy only the Main.xaml file.

The complete project is required for accurate analysis.

---

# Output

After execution, the generated documentation will be available inside:

```
OutputDocuments/
```

The following files will be generated.

---

## understanding_document.md

A complete technical documentation containing:

- Executive Summary
- Business Objective
- Process Overview
- Framework Used
- Folder Structure
- Dependencies
- Applications Used
- Configuration Files
- Input Files
- Output Files
- Assets
- Queues
- APIs
- Variables
- Arguments
- Workflow Hierarchy
- Detailed Workflow Explanations
- Complete Process Flow
- Business Logic
- Technical Logic
- Data Flow
- Exception Handling
- Integrations
- Risks
- Assumptions
- Best Practices
- Improvement Suggestions
- Conclusion

This document is intended for:

- Developers
- Solution Architects
- Support Teams
- Business Analysts
- Project Managers
- New Team Members

---

## architecture.md

A visual representation of the automation using Mermaid diagrams.

It includes:

- End-to-end process flow
- Workflow hierarchy
- Main workflow
- Invoked workflows
- Decision points
- Exception handling
- Retry logic
- External applications
- Excel interactions
- API calls
- Database interactions
- Queue interactions
- Email interactions
- Start-to-end execution flow

The document can be rendered directly by Markdown viewers that support Mermaid.

---

# Best Practices

- Place only one UiPath project inside the `Input` folder for each execution.
- Ensure the project is complete and not missing referenced workflows.
- Include all configuration files, templates, and supporting resources.
- Avoid renaming project files before analysis.

---

# Notes

- The agent performs recursive analysis of all invoked workflows.
- Nested workflows are automatically analyzed.
- Business logic and technical logic are documented separately.
- Exception handling paths are identified and documented.
- External integrations such as Excel, databases, APIs, email, Orchestrator assets, and queues are included whenever detected.
- Mermaid diagrams are generated automatically based on the discovered workflow structure.

The generated documentation is designed to provide a complete understanding of the UiPath automation, enabling maintenance, enhancement, troubleshooting, and onboarding without requiring users to inspect the XAML files directly.
