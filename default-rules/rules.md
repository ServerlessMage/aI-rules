---
description: Use ALWAYS when asked to CREATE A RULE or UPDATE A RULE or taught a lesson from the user that should be retained as a new rule for AI
globs: ""
---
# Cursor Rules Format
## Core Structure

```mdc
---
description: ACTION when TRIGGER to OUTCOME
globs: *.mdc
---

# Rule Title

## Context
- When to apply this rule
- Prerequisites or conditions

## Requirements
- Concise, actionable items
- Each requirement must be testable

## Examples
<example>
Good concise example with explanation
</example>

<example type="invalid">
Invalid concise example with explanation
</example>
```

## File Organization

### Location
- Path: `.cursor/rules/`
- Extension: `.mdc`

### Naming Convention
PREFIX-name.mdc where PREFIX is:
- 0XX: Core standards
- 1XX: Tool configs
- 3XX: Testing standards
- 1XXX: Language rules
- 2XXX: Framework rules
- 8XX: Workflows
- 9XX: Templates
- _name.mdc: Private rules

### Glob Pattern Examples
Common glob patterns for different rule types:
- Core standards: .cursor/rules/*.mdc
- Language rules: src/**/*.{js,ts}
- Testing standards: **/*.test.{js,ts}
- React components: src/components/**/*.tsx
- Documentation: docs/**/*.md
- Configuration files: *.config.{js,json}
- Build artifacts: dist/**/*
- Multiple extensions: src/**/*.{js,jsx,ts,tsx}
- Multiple files: dist/**/*, docs/**/*.md

## Required Fields

### Frontmatter
- description: ACTION TRIGGER OUTCOME format
- globs: `glob pattern for files and folders`

### Body
- <version>X.Y.Z</version>
- context: Usage conditions
- requirements: Actionable items
- examples: Both valid and invalid

## Formatting Guidelines

- Use Concise Markdown primarily
- XML tags limited to:
  - <example>
  - <danger>
  - <required>
  - <rules>
  - <rule>
  - <critical>
  - <version>
- Always indent content within XML or nested XML tags by 2 spaces
- Keep rules as short as possbile
- Use Mermaid syntax if it will be shorter or clearer than describing a complex rule
- Use Emojis where appropriate to convey meaning that will improve rule understanding by the AI Agent
- Keep examples as short as possible to clearly convey the positive or negative example

## AI Optimization Tips

1. Use precise, deterministic ACTION TRIGGER OUTCOME format in descriptions
2. Provide concise positive and negative example of rule application in practice
3. Optimize for AI context window efficiency
4. Remove any non-essential or redundant information
5. Use standard glob patterns without quotes (e.g., *.js, src/**/*.ts)

## AI Context Efficiency

1. Keep frontmatter description under 120 characters (or less) while maintaining clear intent for rule selection by AI AGent
2. Limit examples to essential patterns only
3. Use hierarchical structure for quick parsing
4. Remove redundant information across sections
5. Maintain high information density with minimal tokens
6. Focus on machine-actionable instructions over human explanations

<critical>
  - NEVER include verbose explanations or redundant context that increases AI token overhead
  - Keep file as short and to the point as possible BUT NEVER at the expense of sacrificing rule impact and usefulness for the AI Agent.
  - the front matter can ONLY have the fields description and globs.
</critical>

## Example

<rule>
  <meta>
    <title>Cursor Rule File Format (.mdc)</title>
    <description>Standard format and requirements for writing .mdc files (Cursor rules)</description>
    <created-at utc-timestamp="1744157700">April 9, 2025, 10:15 AM AEST</created-at>
    <last-updated-at utc-timestamp="1744240920">April 10, 2025, 09:22 AM AEST</last-updated-at>
    <applies-to>
      <file-matcher glob="*.mdc">All Cursor rule files with .mdc extension</file-matcher>
      <action-matcher action="create-rule">Triggered when creating a new Cursor rule</action-matcher>
    </applies-to>
  </meta>
  <requirements>
    <requirement priority="critical">
      <description>Follow the specified structure for .mdc files, starting with YAML frontmatter followed by XML-like rule representation.</description>
      <examples>
        <example title="Basic Rule Structure">
          <correct-example title="Properly structured rule file" conditions="Creating a new rule file" expected-result="Valid .mdc file structure" correctness-criteria="Follows the required format with frontmatter and XML structure"><![CDATA[---
description: Example rule
globs: "*.js"
alwaysApply: false
---

<rule>
  <meta>
    <title>JavaScript Formatting Rule</title>
    <description>Ensures consistent formatting in JavaScript files</description>
    <!-- Additional meta elements -->
  </meta>
  <!-- Additional rule elements -->
</rule>]]></correct-example>
          <incorrect-example title="Improperly structured rule file" conditions="Creating a new rule file" expected-result="Valid .mdc file structure" incorrectness-criteria="Missing frontmatter and incorrect XML structure"><![CDATA[<rule>
  <meta>
    <title>JavaScript Formatting Rule</title>
    <description>Ensures consistent formatting in JavaScript files</description>
  </meta>
</rule>]]></incorrect-example>
        </example>
      </examples>
    </requirement>
    <requirement priority="critical">
      <description>Use concise, clear, and unambiguous language throughout the rule.</description>
      <examples>
        <example title="Language Clarity">
          <correct-example title="Clear requirement description" conditions="Writing a requirement" expected-result="Unambiguous instruction" correctness-criteria="Uses precise, clear language"><![CDATA[<requirement priority="high">
  <description>Limit function length to maximum 30 lines of code.</description>
</requirement>]]></correct-example>
          <incorrect-example title="Ambiguous requirement description" conditions="Writing a requirement" expected-result="Clear instruction" incorrectness-criteria="Uses vague, imprecise language"><![CDATA[<requirement priority="high">
  <description>Functions should not be too long.</description>
</requirement>]]></incorrect-example>
        </example>
      </examples>
    </requirement>
    <requirement priority="critical">
      <description>Include concrete examples that demonstrate correct and incorrect applications of the rule.</description>
      <examples>
        <example title="Providing Related Examples">
          <correct-example title="Related correct/incorrect examples" conditions="Demonstrating a naming convention rule" expected-result="Examples that clearly show the contrast" correctness-criteria="Examples are directly related and show the same concept correctly and incorrectly"><![CDATA[<example title="Variable Naming">
  <correct-example title="Proper camelCase" conditions="Naming a variable" expected-result="Valid variable name" correctness-criteria="Uses camelCase format">const userName = 'John';</correct-example>
  <incorrect-example title="Improper snake_case" conditions="Naming a variable" expected-result="Valid variable name" incorrectness-criteria="Uses snake_case instead of required camelCase">const user_name = 'John';</incorrect-example>
</example>]]></correct-example>
          <incorrect-example title="Unrelated examples" conditions="Demonstrating a naming convention rule" expected-result="Clear contrast between correct and incorrect" incorrectness-criteria="Examples show different concepts and aren't directly comparable"><![CDATA[<example title="Variable Naming">
  <correct-example title="Proper camelCase" conditions="Naming a variable" expected-result="Valid variable name" correctness-criteria="Uses camelCase format">const userName = 'John';</correct-example>
  <incorrect-example title="Missing semicolon" conditions="Ending a statement" expected-result="Statement with semicolon" incorrectness-criteria="Lacks required semicolon">const age = 30</incorrect-example>
</example>]]></incorrect-example>
        </example>
      </examples>
    </requirement>
    <requirement priority="critical">
      <description>All other rule files (.mdc) must reference this file as a dependency in their references section.</description>
      <examples>
        <example title="Rule Dependency Reference">
          <correct-example title="Proper dependency reference" conditions="Creating a new rule file" expected-result="Valid reference to rules.mdc" correctness-criteria="Includes proper reference to the main rules file"><![CDATA[<references>
  <reference as="dependency" href=".cursor/rules/rules.mdc" reason="Follows standard rule format">Base rule format definition</reference>
  <!-- Other references as needed -->
</references>]]></correct-example>
          <incorrect-example title="Missing dependency reference" conditions="Creating a new rule file" expected-result="Valid reference to rules.mdc" incorrectness-criteria="Missing required reference to the main rules file"><![CDATA[<references>
  <reference as="context" href=".cursor/rules/other-file.mdc" reason="Related context">Additional context</reference>
  <!-- No reference to the main rules.mdc file -->
</references>]]></incorrect-example>
        </example>
      </examples>
    </requirement>
    <requirement priority="high">
      <description>Use CDATA tags for multiline content within XML elements.</description>
      <examples>
        <example title="CDATA Usage">
          <correct-example title="Proper CDATA wrapping" conditions="Including multiline content" expected-result="Properly escaped content" correctness-criteria="Uses CDATA tags correctly"><![CDATA[<example title="Function Structure">
  <correct-example title="Properly structured function" conditions="Writing a function" expected-result="Valid function definition" correctness-criteria="Follows the style guide"><![CDATA[function calculateTotal(items) {
  let sum = 0;
  for (const item of items) {
    sum += item.price;
  }
  return sum;
}]]]]><![CDATA[></correct-example>
</example>]]></correct-example>
          <incorrect-example title="Missing CDATA tags" conditions="Including multiline content" expected-result="Properly escaped content" incorrectness-criteria="Lacks required CDATA tags"><![CDATA[<example title="Function Structure">
  <correct-example title="Properly structured function" conditions="Writing a function" expected-result="Valid function definition" correctness-criteria="Follows the style guide">function calculateTotal(items) {
  let sum = 0;
  for (const item of items) {
    sum += item.price;
  }
  return sum;
}</correct-example>
</example>]]></incorrect-example>
        </example>
      </examples>
    </requirement>
    <non-negotiable priority="critical">
      <description>Always maintain precise structure and avoid ambiguity in rule definitions.</description>
      <examples>
        <example title="Precision and Clarity">
          <correct-example title="Precise rule specification" conditions="Defining a coding standard" expected-result="Clear, actionable rule" correctness-criteria="Provides specific, measurable criteria"><![CDATA[<requirement priority="high">
  <description>Maximum line length must be 80 characters. Lines exceeding this limit must be broken with line continuation.</description>
</requirement>]]></correct-example>
          <incorrect-example title="Ambiguous rule specification" conditions="Defining a coding standard" expected-result="Clear, actionable rule" incorrectness-criteria="Uses subjective language without measurable criteria"><![CDATA[<requirement priority="high">
  <description>Keep lines to a reasonable length and break them when they get too long.</description>
</requirement>]]></incorrect-example>
        </example>
      </examples>
    </non-negotiable>
    <non-negotiable priority="critical">
      <description>Create rules that address only the specified problem without introducing unrelated constraints or assumptions.</description>
      <examples>
        <example title="Problem-Specific Rules">
          <correct-example title="Focused rule" conditions="Creating a rule for import ordering" expected-result="Rule addressing only import ordering" correctness-criteria="Stays focused on the specific problem"><![CDATA[<requirement priority="medium">
  <description>Group imports in the following order: built-in modules, external dependencies, internal modules.</description>
</requirement>]]></correct-example>
          <incorrect-example title="Overreaching rule" conditions="Creating a rule for import ordering" expected-result="Rule addressing only import ordering" incorrectness-criteria="Introduces unrelated constraints"><![CDATA[<requirement priority="medium">
  <description>Group imports in the following order: built-in modules, external dependencies, internal modules. Also ensure all code is properly commented and functions have error handling.</description>
</requirement>]]></incorrect-example>
        </example>
      </examples>
    </non-negotiable>
    <non-negotiable priority="critical">
      <description>Use single line content when possible, reserving multiline format only for examples or when absolutely necessary.</description>
      <examples>
        <example title="Line Format Efficiency">
          <correct-example title="Concise single-line description" conditions="Writing a requirement description" expected-result="Brief, clear description" correctness-criteria="Uses single line for concise content"><![CDATA[<description>Use camelCase for variable names and PascalCase for class names.</description>]]></correct-example>
          <incorrect-example title="Unnecessarily multi-line description" conditions="Writing a requirement description" expected-result="Brief, clear description" incorrectness-criteria="Splits simple content across multiple lines"><![CDATA[<description>
  Use camelCase for variable names
  and PascalCase for class names.
</description>]]></incorrect-example>
        </example>
      </examples>
    </non-negotiable>
    <non-negotiable priority="critical">
      <description>Mark critical requirements that are most likely to be misinterpreted as non-negotiable with high priority.</description>
      <examples>
        <example title="Non-negotiable Designation">
          <correct-example title="Properly marked critical requirement" conditions="Defining a security standard" expected-result="Clearly marked non-negotiable rule" correctness-criteria="Uses non-negotiable tag with critical priority"><![CDATA[<non-negotiable priority="critical">
  <description>Never store passwords or API keys in source code.</description>
</non-negotiable>]]></correct-example>
          <incorrect-example title="Improperly categorized critical requirement" conditions="Defining a security standard" expected-result="Clearly marked non-negotiable rule" incorrectness-criteria="Uses regular requirement tag for critical security rule"><![CDATA[<requirement priority="high">
  <description>Never store passwords or API keys in source code.</description>
</requirement>]]></incorrect-example>
        </example>
      </examples>
    </non-negotiable>
  </requirements>
  <grammar>
    <grammar-entry title="Frontmatter Structure">
      <pattern description="YAML frontmatter format">^---\ndescription: .+\nglobs: .+\nalwaysApply: (true|false)\n---$</pattern>
      <example description="Valid frontmatter">---
description: Rule for consistent comment formatting
globs: "*.{js,ts}"
alwaysApply: false
---</example>
    </grammar-entry>
    <grammar-entry title="XML Structure">
      <pattern description="Valid rule element structure">&lt;rule&gt;(\s*&lt;meta&gt;.+&lt;/meta&gt;\s*&lt;requirements&gt;.+&lt;/requirements&gt;(\s*&lt;grammar&gt;.+&lt;/grammar&gt;)?(\s*&lt;context&gt;.+&lt;/context&gt;)?(\s*&lt;references&gt;.+&lt;/references&gt;)?)&lt;/rule&gt;</pattern>
    </grammar-entry>
    <schema title="Cursor Rule Schema" description="Standard structure for .mdc cursor rule files">
<![CDATA[---
description: {Rule description}
globs: {glob patterns of files this rule applies to, or wildcard glob that matches all files if alwaysApply is true}
alwaysApply: {true or false}
---

<rule>
  <meta>
    <title>{Rule title}</title>
    <description>{Rule description}</description>
    <created-at utc-timestamp="{creation timestamp in UTC}">{Datetime of creation in human-readable form and local time}</created-at>
    <last-updated-at utc-timestamp="{last update timestamp in UTC}">{Last update date and time in human-readable form and local time}</last-updated-at>
    <applies-to>
      <file-matcher glob="{glob pattern}">{Description of files that match the glob}</file-matcher>
      <!-- Repeat for each glob or use a wildcard glob if alwaysApply is true -->
      <!-- Optional action matcher -->
      <action-matcher action="{keyword for trigger action}">{Description of action that triggers this rule}</action>
    </applies-to>
  </meta>
  <requirements>
    <requirement priority="{priority keyword: high, medium, critical, etc.}">
      <description>{Requirement description}</description>
      <examples>
        <example title="{example title}">
          <correct-example title="{scenario title}" conditions="{conditions description}" expected-result="{expected result}" correctness-criteria="{correctness criteria}">{Example content}</correct-example>
          <incorrect-example title="{scenario title}" conditions="{conditions description}" expected-result="{expected result}" incorrectness-criteria="{incorrectness criteria}">{Example content}</incorrect-example>
        </example>
        <!-- Repeat for each example -->
      </examples>
    </requirement>
    <!-- Repeat for each requirement -->
    <non-negotiable priority="{priority keyword: high, medium, critical, etc.}">
      <description>{Non-negotiable requirement description}</description>
      <examples>
        <example title="{example title}">
          <correct-example title="{scenario title}" conditions="{conditions description}" expected-result="{expected result}" correctness-criteria="{correctness criteria}">{Example content}</correct-example>
          <incorrect-example title="{scenario title}" conditions="{conditions description}" expected-result="{expected result}" incorrectness-criteria="{incorrectness criteria}">{Example content}</incorrect-example>
        </example>
        <!-- Repeat for each example -->
      </examples>
    </non-negotiable>
    <!-- Repeat for each non-negotiable requirement -->
  </requirements>
  <grammar>
    <grammar-entry title="{grammar requirement title}">
      <pattern description="{pattern description}">{grammar pattern}</pattern>
      <!-- Repeat for each pattern -->
      <example description="{example description}">{example content}</example>
      <!-- Repeat for each example -->
    </grammar-entry>
    <!-- Repeat for each grammar entry -->
    <schema title="{schema title}" description="{schema description}">
      <!-- Include schema definition here if applicable -->
    </schema>
  </grammar>
  <!-- Optional context section -->
  <context description="{context description}">
    {Additional information for rule application}
  </context>
  <!-- Optional references section -->
  <references>
    <reference as="{reference type: context, dependency, glossary, examples}" href="{path to referenced file}" reason="{reason for reference}">{Reference description}</reference>
    <!-- Repeat for each reference -->
  </references>
</rule>]]>
    </schema>
  </grammar>
  <context description="Additional considerations for creating Cursor rules">
    The primary purpose of a Cursor rule (.mdc file) is to provide clear, unambiguous guidance that can be consistently applied. The rule acts essentially as a prompt or part of a prompt for an LLM, so clarity and precision are paramount. When creating rules, focus on solving only the specific problem the rule is intended to address, avoiding scope creep or assumptions based on common knowledge unless explicitly requested.
  </context>
</rule>