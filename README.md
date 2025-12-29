# Claude Code Skills - API Development Automation

This repository contains a collection of custom skills for Claude Code, specifically designed to automate and optimize tasks related to API development.

## 📋 Table of Contents

- [What are Claude Code Skills?](#what-are-claude-code-skills)
- [Installation](#installation)
- [Available Skills](#available-skills)
- [How to Use the Skills](#how-to-use-the-skills)
- [Creating Your Own Skills](#creating-your-own-skills)
- [Contributing](#contributing)

## What are Claude Code Skills?

Skills are sets of specialized instructions and knowledge that extend Claude Code's capabilities to perform specific tasks more efficiently and consistently. In this case, our skills are focused on automating common processes in API development.

## Installation

### Method 1: Manual Installation

1. **Locate your Claude Code skills directory:**
   - **macOS/Linux**: `~/.claude/skills/`
   - **Windows**: `%USERPROFILE%\.claude\skills\`

2. **Clone this repository:**
```bash
   cd ~/.claude/skills/  # or the corresponding path on Windows
   git clone https://github.com/your-username/claude-code-api-skills.git
```

3. **Copy the individual skills:**
```bash
   cp -r claude-code-api-skills/skills/* .
```

### Method 2: Individual Skill Installation

If you only want to install specific skills:

1. **Navigate to the skills directory:**
```bash
   cd ~/.claude/skills/
```

2. **Copy only the skill folders you need:**
```bash
   cp -r /path/to/repo/skills/skill-name ./
```

### Verify Installation

After installing the skills:

1. Restart Claude Code if it's running
2. Skills should load automatically
3. You can verify by asking Claude: "What skills do you have available for API development?"

## Available Skills

| Skill | Description | Use Cases |
|-------|-------------|-----------|
| `mulesoft-documentor` | Generates documentation for a MuleSoft application | Create MuleSoft application documentation when it doesn't exist |
| `mulesoft-api-patterns` | Generates MuleSoft projects based on API design patterns | Generate sample code based on Design principles |

*(This table will be updated as more skills are added)*

## How to Use the Skills

Skills are activated automatically when Claude Code detects that your request is related to their domain. However, you can also invoke them explicitly:

### Usage Examples

**Generate an OpenAPI specification:**
```
Using the api-spec-generator skill, create an OpenAPI 3.0 specification 
for a user management API with CRUD endpoints
```

**Create a new endpoint:**
```
Create a POST /api/products endpoint that accepts name, price, and description, 
validating that the price is positive
```

**Generate tests:**
```
Generate unit tests for the GET /api/users/:id endpoint using Jest
```

## Creating Your Own Skills

### Skill Structure

Each skill must follow this structure:
```
your-skill-name/
├── SKILL.md          # Main file (required)
├── templates/        # Optional templates
├── examples/         # Usage examples
└── resources/        # Additional resources
```

### SKILL.md Format

Your `SKILL.md` file should include:
```markdown
# Skill Name

## Description
[Brief description of what the skill does]

## When to Use This Skill
- [Scenario 1]
- [Scenario 2]
- [Scenario 3]

## Capabilities
- [Capability 1]
- [Capability 2]

## Instructions for Claude
[Detailed step-by-step instructions on how to execute the skill]

## Usage Examples

### Example 1: [Title]
**User input:**
```
[Example of what the user would request]
```

**Expected action:**
[What Claude should do]

## Special Considerations
- [Important note 1]
- [Important note 2]

## Dependencies
- [If it requires specific tools]
- [If it depends on other skills]
```

### Best Practices

1. **Be specific**: Provide clear and detailed instructions
2. **Include examples**: Examples help Claude understand the context
3. **Define scope**: Clearly specify when to use (and when not to use) the skill
4. **Maintain modularity**: Each skill should have a well-defined purpose
5. **Document dependencies**: Indicate if it requires specific libraries, frameworks, or tools

### Complete Example

Review the existing skills in this repository as a reference. For example, `api-spec-generator/SKILL.md` is a good starting point.

## Contributing

Contributions are welcome! If you have a useful skill for API development:

1. **Fork** this repository
2. **Create** a new branch (`git checkout -b feature/new-skill`)
3. **Add** your skill following the described structure
4. **Commit** your changes (`git commit -m 'Add: skill for [functionality]'`)
5. **Push** to the branch (`git push origin feature/new-skill`)
6. **Open** a Pull Request

### Contribution Guidelines

- Ensure your skill is well documented
- Include clear usage examples
- Verify it doesn't duplicate existing functionality
- Follow naming conventions (kebab-case for folder names)
- Update the "Available Skills" table in this README

## Repository Structure
```
.
├── README.md
├── skills/
│   ├── api-spec-generator/
│   │   ├── SKILL.md
│   │   └── templates/
│   ├── endpoint-creator/
│   │   ├── SKILL.md
│   │   └── examples/
│   └── ... (more skills)
└── docs/
    ├── creating-skills.md
    └── best-practices.md
```

## Support

If you encounter problems or have questions:

- 📝 Open an [Issue](https://github.com/your-username/claude-code-api-skills/issues)
- 💬 Check the [official Claude Code documentation](https://docs.claude.com)
- 🤝 Join the discussions in the Discussions section

## License

TBD

## Acknowledgments

- Developed for the Claude Code community
- Inspired by API development best practices

---

**Note**: This project is not officially affiliated with Anthropic. It is a community project to improve the development experience with Claude Code.