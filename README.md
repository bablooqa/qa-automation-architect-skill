# 🤖 QA Automation Architect AI Skill

[![npm version](https://badge.fury.io/js/qa-automation-architect-skill.svg)](https://badge.fury.io/js/qa-automation-architect-skill)
[![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg)](https://opensource.org/licenses/MIT)

**QA Automation Architect AI Skill** is a production-grade, highly optimized prompt and knowledge base engineered specifically for modern AI coding assistants (like Cursor, Windsurf, GitHub Copilot, Antigravity, Qoder, and Trae). 

This skill empowers your AI coding agent to become a **Senior QA Automation Architect** capable of designing, generating, optimizing, and debugging robust QA automation testing systems, AI LLM testing frameworks, and continuous integration (CI/CD) pipelines.

## 🌟 What This Skill Does

When you add this skill to your AI coding assistant, it gains deep expertise in:
- **Test Framework Architecture**: Structuring scalable Python, Pytest, Playwright, and Selenium projects using Page Object Model (POM) and modular locators.
- **AI & LLM Testing**: Evaluating chatbots, RAG pipelines, and conversational AI agents with assertions based on exact match, semantic similarity, and threshold algorithms.
- **API Test Automation**: Building robust REST/GraphQL testing suites with FastAPI and Pytest-HTTP.
- **CI/CD Integration**: Seamlessly integrating tests with GitHub Actions, Docker, and Allure reporting.
- **Smart Debugging**: Diagnosing test failures, locating root causes, and providing actionable fixes instantly.

---

## 📦 Installation via NPM

You can easily install the QA Automation Architect Skill globally or locally via NPM to pull the latest instructions directly into your workspace.

```bash
# Global installation (recommended for use across multiple projects)
npm install -g qa-automation-architect-skill

# Local installation (Project level)
npm install qa-automation-architect-skill --save-dev
```

*Note: Installing via NPM will download the `SKILL.md` and reference files to your `node_modules` folder, which you can directly reference in your AI coding assistant's rules or prompt context.*

---

## 🛠️ How to Add This Skill to Your AI Coding Agent

Integrating this skill into your favorite AI IDEs and coding agents is simple. The core of this skill relies on the `SKILL.md` file and its references.

### 1. Cursor IDE
1. Open your project in Cursor.
2. Create a `.cursorrules` file in the root directory.
3. Copy the contents of the `SKILL.md` file into your `.cursorrules` file.
4. Alternatively, you can tag `@SKILL.md` directly in the Cursor Chat to instruct the AI to follow the QA Architect guidelines on demand.

### 2. Windsurf IDE
1. Open your project in Windsurf.
2. Create a `.windsurfrules` file in the root of your repository.
3. Paste the `SKILL.md` contents into this file to enforce the QA Architect behavior globally for the workspace.

### 3. Antigravity (Google DeepMind)
1. Create a `skills/qa-automation-architect/` directory inside your `.gemini/antigravity/skills/` folder.
2. Copy `SKILL.md` and the `references/` folder into it.
3. The Antigravity agent will automatically index this as an available skill and use it when prompted for QA/Testing tasks.

### 4. Qoder
1. Open the Qoder settings and navigate to Custom Prompts or Agent Roles.
2. Paste the `SKILL.md` contents as a custom system prompt.
3. Name it **QA Automation Architect** and select this persona when generating testing frameworks.

### 5. Trae
1. Add the `SKILL.md` and `references/` folder to your workspace.
2. When chatting with Trae, simply start your prompt with: *"Read SKILL.md. Acting as the QA Automation Architect..."*

### 6. GitHub Copilot & ChatGPT
1. Copy the content from `SKILL.md`.
2. Paste it into your Custom Instructions (ChatGPT) or provide it as the leading context prompt when starting a new session to set the persona.

---

## 📖 Usage Examples

Once installed and configured, simply ask your AI agent:

- *"Set up a Playwright UI testing framework for a Next.js app using POM."*
- *"Write a Pytest API test for my FastAPI login endpoint."*
- *"How do I test my new RAG chatbot for semantic accuracy?"*
- *"Debug this failing Selenium test and fix the stale element reference."*
- *"Create a GitHub Actions CI pipeline to run my tests in Docker."*

The AI will now reply with structured, scalable, and production-ready QA code following the architect guidelines!
