# Reference

**Information-oriented documentation**

## Purpose

Reference guides provide technical descriptions and specifications. They are **information-oriented**, containing accurate and complete information that users need to correctly use the system. Reference is consulted, not followed.

## What Belongs Here

### ✅ Include:
- **API documentation** - All endpoints, parameters, responses
- **Configuration options** - Every setting with type and default
- **Data models** - Complete schema definitions
- **Error codes** - All possible errors and meanings
- **Type definitions** - Complete TypeScript/type information
- **Command-line interfaces** - All flags and arguments
- **Technical specifications** - Exact behavior descriptions
- **Neutral, factual information** - No opinions

### ❌ Don't Include:
- Step-by-step instructions (save for [Tutorials](../tutorials/) or [How-to Guides](../how-to-guides/))
- Opinions or recommendations (save for [Explanation](../explanation/))
- Learning paths or lessons (save for [Tutorials](../tutorials/))
- Problem-solving scenarios (save for [How-to Guides](../how-to-guides/))

## Writing Guidelines

1. **Be comprehensive** - Document everything, leave nothing out
2. **Be consistent** - Use the same format throughout
3. **Be accurate** - Technical precision is critical
4. **Structure matches code** - Mirror the actual architecture
5. **Be neutral** - Just the facts, no interpretation
6. **Be concise** - Dense information, minimal prose
7. **Use examples sparingly** - Only to clarify meaning

## Example Reference Topics

- API Endpoints Reference
- Configuration Options
- Database Schema
- Environment Variables
- OAuth Client API
- Error Codes and Messages
- CLI Command Reference
- TypeScript Types and Interfaces

## Reference Structure Template

```markdown
# [Component/API Name]

Brief one-line description.

## Properties/Parameters

### `propertyName`
- **Type:** `string | number`
- **Required:** Yes/No
- **Default:** `value`
- **Description:** What it does

## Methods/Endpoints

### `methodName(param1, param2)`

**Description:** What it does

**Parameters:**
- `param1` (type) - Description
- `param2` (type) - Description

**Returns:** Return type and description

**Example:**
\`\`\`typescript
// Minimal example showing usage
\`\`\`

**Errors:**
- `ERROR_CODE` - When this occurs

## See Also
Links to related reference pages
```

---

[← Back to Main Documentation](../../README.md)
