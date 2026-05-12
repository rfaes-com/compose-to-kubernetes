# Contributing to Compose to Kubernetes Workshop

Thank you for your interest in contributing to this Kubernetes workshop! This document provides guidelines and information for contributors.

## How to Contribute

### Reporting Issues

If you find errors, typos, or have suggestions:

1. Check existing issues to avoid duplicates
2. Create a new issue with:
   - Clear description
   - Steps to reproduce (if applicable)
   - Expected vs actual behavior
   - Environment details (OS, Kubernetes version, etc.)

### Suggesting Enhancements

We welcome suggestions for:

- New workshop sections
- Additional examples
- Improved explanations
- Better lab exercises
- Tool updates

### Contributing Content

#### Workshop Content Guidelines

1. **Structure**
   - Each section should have: README.md (theory), examples/, lab/instructions.md, lab/solutions/
   - Follow existing format and style
   - Keep explanations clear and concise

2. **Code Examples**
   - Use public container images (no proprietary images)
   - Test all examples before submitting
   - Include comments explaining key concepts
   - Follow Kubernetes best practices

3. **Lab Exercises**
   - Include clear objectives
   - Provide step-by-step instructions
   - Add verification steps
   - Include solutions with explanations

4. **Style Guide**
   - No emojis in documentation (accessibility)
   - Use markdown for formatting
   - Code blocks with proper syntax highlighting
   - Consistent terminology

#### Pull Request Process

1. **Fork the repository**

2. **Create a feature branch**

   ```bash
   git checkout -b feature/your-feature-name
   ```

3. **Make your changes**
   - Follow the style guide
   - Test all examples and exercises
   - Update relevant documentation

4. **Commit with clear messages**

   ```bash
   git commit -m "Add: Section on StatefulSets advanced patterns"
   ```

5. **Push to your fork**

   ```bash
   git push origin feature/your-feature-name
   ```

6. **Create a Pull Request**
   - Describe what you changed and why
   - Reference related issues
   - Include screenshots if relevant

#### Testing Requirements

Before submitting:

- [ ] All YAML files are valid
- [ ] Examples work on a fresh cluster
- [ ] Lab solutions are accurate
- [ ] No broken links in documentation
- [ ] Spelling and grammar checked

#### Commit Message Format

Use clear, descriptive commit messages:

```text
Type: Brief description

Detailed explanation of changes (if needed)

Fixes #123
```

**Types:**

- `Add:` New content or features
- `Fix:` Bug fixes or corrections
- `Update:` Changes to existing content
- `Remove:` Deletion of content
- `Docs:` Documentation only changes
- `Style:` Formatting changes

### Documentation Contributions

Help improve documentation by:

- Fixing typos and grammar
- Clarifying confusing sections
- Adding diagrams or visual aids
- Improving examples
- Updating outdated information

## Code of Conduct

Please note that this project is released with a [Code of Conduct](CODE_OF_CONDUCT.md). By participating in this project you agree to abide by its terms.

## Recognition

Contributors will be recognized in the project:

- Added to AUTHORS file
- Mentioned in release notes
- Acknowledged in documentation

## Questions?

- Open an issue for questions about contributing
- Join discussions in issues and pull requests
- Check existing issues and PRs for answers

## License

By contributing, you agree that your contributions will be licensed under the MIT License.

## Thank You

Your contributions make this workshop better for everyone learning Kubernetes. We appreciate your time and effort!
