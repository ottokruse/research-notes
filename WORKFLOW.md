# Workflow for Writing Research Notes

## For the Web Research Agent

To add a new research note:

1. Create filename: `YYYY-MM-DD-topic-name.md`
2. Use GitHub API: `PUT /repos/ottokruse/research-notes/contents/{filename}`
3. Include frontmatter:
   ```yaml
   ---
   title: Research Title
   date: YYYY-MM-DD
   tags: [tag1, tag2]
   ---
   ```

## API Details

- Repository: ottokruse/research-notes
- Branch: main
- Content must be base64 encoded
- Include descriptive commit message

## File Format

All files should be Markdown (.md) with:
- YAML frontmatter (optional but recommended for Jekyll)
- Clear headings
- Proper markdown formatting
- Links to sources where applicable

## Example API Call

```
PUT /repos/ottokruse/research-notes/contents/2025-11-23-example-topic.md
{
  "message": "Add research note: Example Topic",
  "content": "<base64-encoded-content>",
  "branch": "main"
}
```

## GitHub Pages

The site is published at: https://ottokruse.github.io/research-notes/
- Changes typically appear within 1-2 minutes
- Jekyll will automatically render markdown files

## File Naming Convention

Use descriptive, URL-friendly names:
- ✓ `2025-11-23-aws-lambda-best-practices.md`
- ✓ `2025-11-23-python-async-patterns.md`
- ✗ `2025-11-23-notes.md` (too generic)
- ✗ `2025-11-23-My Research!!!.md` (special characters, spaces)

## Content Guidelines

- **Be specific**: Include key findings, not just links
- **Cite sources**: Link to original articles and documentation
- **Use headings**: Organize content with proper markdown structure
- **Add tags**: Help with categorization in frontmatter
- **Date matters**: Always use current date in filename and frontmatter
