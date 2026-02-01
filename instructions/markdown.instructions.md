---
name: 'Markdown Standards'
description: 'This file describes additional markdown standards.'
applyTo: '**/*.md'
---

# Markdown Standards

## CRITICAL RULES - MANDATORY COMPLIANCE

These rules override ALL other markdown style guides. Your linter WILL flag violations. Follow them exactly.

### Rule 1: Headings MUST Have Blank Lines After

**REQUIRED:** Every heading (# ## ### etc.) MUST be immediately followed by a blank line.

❌ **WRONG:**
```markdown
## Section Title
This is content immediately after.
```

✅ **CORRECT:**
```markdown
## Section Title

This is content with a blank line after the heading.
```

### Rule 2: Lists MUST Have Blank Lines After

**REQUIRED:** Every list (ordered or unordered) MUST be followed by a blank line before any subsequent content.

❌ **WRONG:**
```markdown
- Item 1
- Item 2
The next paragraph starts here.
```

✅ **CORRECT:**
```markdown
- Item 1
- Item 2

The next paragraph starts here with a blank line before it.
```

### Rule 3: Code Blocks MUST Be Preceded by Blank Lines

**REQUIRED:** All fenced code blocks (```) MUST have a blank line before them.

❌ **WRONG:**
```markdown
Here is some code:
```javascript
const x = 1;
```

✅ **CORRECT:**
```markdown
Here is some code:

```javascript
const x = 1;
```

### Rule 4: Code Blocks MUST Specify Language

**REQUIRED:** Every fenced code block MUST have a language identifier immediately after the opening ```.

❌ **WRONG:**
```markdown
```
const x = 1;
```

✅ **CORRECT:**
```markdown
```javascript
const x = 1;
```

**Common language identifiers:** javascript, typescript, python, bash, json, markdown, html, css, sql, yaml

### Rule 5: Code Blocks in Lists MUST Be Double Indented

**REQUIRED:** When placing code blocks inside list items, indent the code block with 8 spaces (double indent).

❌ **WRONG:**
```markdown
- Step 1: Do something
  ```javascript
  const x = 1;
  ```
```

✅ **CORRECT:**
```markdown
- Step 1: Do something

        ```javascript
        const x = 1;
        ```
```

### Rule 6: NEVER Use Bare URLs

**REQUIRED:** Never write URLs as plain text. Always use markdown link syntax `[text](url)`.

❌ **WRONG:**
```markdown
Visit https://example.com for more info.
Check out www.github.com.
```

✅ **CORRECT:**
```markdown
Visit [example.com](https://example.com) for more info.
Check out [GitHub](https://www.github.com).
```

## Compliance Checklist

Before submitting any markdown, verify:

- [ ] All headings have blank lines after them
- [ ] All lists have blank lines after them
- [ ] All code blocks have blank lines before them
- [ ] All code blocks specify a language
- [ ] All code blocks in lists are double-indented (8 spaces)
- [ ] All URLs use link syntax `[text](url)`, no bare URLs

## Why These Rules Matter

These rules ensure:

1. Consistent rendering across different markdown parsers
2. Proper linting without errors
3. Better readability and maintainability
4. Compliance with strict markdown standards
