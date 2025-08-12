# AI Rules Collection

A comprehensive collection of AI context rules and coding standards for enhanced AI-assisted development across multiple platforms and languages.

## 🎯 Overview

This repository contains curated AI context rules designed to improve code quality, consistency, and development efficiency when working with AI assistants like Amazon Q, GitHub Copilot, Cursor, and Claude.

## 📚 Resources & References

### Essential Reading
- [Awesome Reviewers](https://awesomereviewers.com/) - Code review best practices
- [Cursor Directory](https://cursor.directory/) - Cursor AI rules and prompts
- [Awesome Cursor Rules](https://github.com/PatrickJS/awesome-cursorrules) - Community cursor rules

### Articles & Guides
- [Coding for the Future: Agentic World](https://addyo.substack.com/p/coding-for-the-future-agentic-world)
- [Maximizing Amazon Q in Command Line](https://medium.com/@jason_94622/maximizing-amazon-q-in-the-command-line-a-guide-to-contextual-ai-0db5c08968d9)

## 🚀 Quick Start: Project Analysis

When starting with a new project, use this prompt to generate a comprehensive project summary:

```
Analyze the code base and provide a summary. Your summary should read exactly the way you would want to read the document if you were using it as an introduction to the project. Evaluate which details you would want in this summary and provide them. After we agree on the project summary, can you create a project-summary.md document with the details we agreed on, and we will refer to that when we start conversations about this project.
```

This creates a `project-summary.md` file that serves as contextual foundation for all AI interactions with your project.

## ⚙️ Setting up AI Context Rules

### For Amazon Q (.amazonq/rules)
1. Create a `.amazonq/rules/` folder in your project root
2. **Always copy** `default-rules/base.md` as it contains essential coding standards
3. Copy additional rules based on your project needs:

**Core Rules (recommended for most projects):**
- `default-rules/base.md` ✅ (required)
- `default-rules/code-quality.md`
- `default-rules/clean-code.md`
- `default-rules/task-list.md`

**Language-specific Rules:**
- **TypeScript projects:** Copy relevant files from `typescript/` folder
- **Python projects:** Copy `python/python-black.md`
- **React projects:** Copy `react/react.md`
- **Docker projects:** Copy `docker/docker.md`

**Testing Rules:**
- `testing/jest-testing.md` (for Jest/JavaScript testing)
- `testing/localstack-env.md` (for AWS LocalStack testing)

### For GitHub Copilot (.github/rules)
1. Create a `.github/rules/` folder in your project root
2. Follow the same copying strategy as Amazon Q above
3. **Always include** `default-rules/base.md` as your foundation

### Example Setup Commands
```bash
# Create directories
mkdir -p .amazonq/rules
mkdir -p .github/rules

# Copy base rules (required)
cp aI-rules/default-rules/base.md .amazonq/rules/
cp aI-rules/default-rules/base.md .github/rules/

# Copy additional rules as needed
cp aI-rules/default-rules/code-quality.md .amazonq/rules/
cp aI-rules/typescript/naming-conventions.md .amazonq/rules/
cp aI-rules/testing/jest-testing.md .amazonq/rules/
```

### Rule Selection Guide
- **Start minimal:** Base + code-quality + language-specific rules
- **Add gradually:** Include testing, framework-specific rules as needed
- **Avoid overloading:** Too many rules can overwhelm the AI context

### Project Types Quick Reference
| Project Type | Recommended Rules |
|--------------|-------------------|
| **Node.js/TypeScript** | base.md + typescript/* + jest-testing.md |
| **Python** | base.md + python-black.md |
| **React App** | base.md + typescript/* + react.md + jest-testing.md |
| **AWS Infrastructure** | base.md + docker.md + localstack-env.md |

## 📂 Available Rule Categories

### 🔧 Default Rules (`default-rules/`)
- **`base.md`** ⭐ - Core coding standards (always include)
- **`code-quality.md`** - Code quality best practices
- **`clean-code.md`** - Clean code principles
- **`task-list.md`** - Task management and planning guidelines
- **`jsdoc-comments.md`** - Documentation standards
- **`rules.md`** - Comprehensive rule collection

### 📘 TypeScript Rules (`typescript/`)
- **`naming-conventions.md`** - TypeScript naming standards
- **`interface-extends.md`** - Interface design patterns
- **`discriminated-unions.md`** - Union type best practices
- **`import-type.md`** - Type import guidelines
- **`default-exports.md`** - Export/import patterns
- **`enums.md`** - Enum usage guidelines
- **`any-inside-generic-functions.md`** - Generic function patterns
- **`optional-properties.md`** - Optional property handling
- **`readonly-properties.md`** - Immutability patterns
- **`return-types.md`** - Function return type standards
- **`throwing.md`** - Error handling patterns
- **`installing-libraries.md`** - Library installation best practices
- **`no-unchecked-indexed-access.md`** - Safe array/object access

### 🐍 Python Rules (`python/`)
- **`python-black.md`** - Python Black formatter rules

### ⚛️ React Rules (`react/`)
- **`react.md`** - React component and hook best practices

### 🐳 Docker Rules (`docker/`)
- **`docker.md`** - Docker and containerization standards

### 🧪 Testing Rules (`testing/`)
- **`jest-testing.md`** - Jest testing framework guidelines
- **`localstack-env.md`** - AWS LocalStack testing setup

### 🐚 Shell Rules (`shell/`)
- **`shell.md`** - Shell scripting best practices
- **`shell-colours.md`** - Shell output formatting

### 🤖 Claude Code Rules (`claude-code/`)
- **`claude-code.md`** - Claude-specific coding guidelines

## 💡 Best Practices

### Rule Selection Strategy
1. **Start with essentials:** Always include `base.md`
2. **Add language-specific rules:** Match your primary development language
3. **Include testing rules:** If you write tests (recommended)
4. **Add framework rules:** Based on your tech stack
5. **Avoid rule overload:** Too many rules can confuse AI assistants

### Maintenance Tips
- **Regular updates:** Keep rules current with your coding standards
- **Team alignment:** Ensure all team members use the same rule set
- **Project-specific customization:** Adapt rules for specific project needs
- **Version control:** Track rule changes in your project repository

## 🤝 Contributing

Feel free to contribute improvements, new rules, or suggestions by:
1. Creating issues for rule requests or improvements
2. Submitting pull requests with new rules
3. Sharing feedback on existing rules

## 📄 License

This collection is provided as-is for educational and development purposes. Adapt and modify as needed for your projects.

