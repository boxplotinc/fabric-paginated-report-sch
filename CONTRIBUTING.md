# Contributing to Fabric Paginated Report Batch Executor

Thank you for your interest in contributing to this project! We welcome contributions from the community.

## Table of Contents

- [Code of Conduct](#code-of-conduct)
- [How Can I Contribute?](#how-can-i-contribute)
- [Getting Started](#getting-started)
- [Development Setup](#development-setup)
- [Submitting Changes](#submitting-changes)
- [Coding Guidelines](#coding-guidelines)
- [Commit Message Guidelines](#commit-message-guidelines)

## Code of Conduct

This project and everyone participating in it is governed by our [Code of Conduct](CODE_OF_CONDUCT.md). By participating, you are expected to uphold this code. Please report unacceptable behavior via GitHub issues.

## How Can I Contribute?

### Reporting Bugs

Before creating bug reports, please check the existing issues to avoid duplicates. When you create a bug report, include as many details as possible using our bug report template.

### Suggesting Enhancements

Enhancement suggestions are tracked as GitHub issues. Use the feature request template and provide:

- A clear and descriptive title
- Detailed description of the proposed feature
- Explain why this enhancement would be useful
- Include examples if applicable

### Contributing Code

1. Fork the repository
2. Create a new branch from `main`
3. Make your changes
4. Test your changes thoroughly
5. Submit a pull request

## Getting Started

### Prerequisites

- Microsoft Fabric workspace with appropriate permissions
- Python 3.x environment
- Access to Power BI REST API
- Basic understanding of Jupyter notebooks

### Understanding the Project Structure

```
fabric-paginated-report-sch/
├── paginated_report_batch_executor.ipynb  # Main notebook
├── config/                                 # Configuration examples
│   ├── config_lakehouse.json
│   ├── config_semantic_model.json
│   ├── config_static_json.json
│   └── config_warehouse.json
├── setup/                                  # Setup scripts
│   ├── create_lakehouse_table.sql
│   ├── create_semantic_model_table.dax
│   └── create_warehouse_table.sql
└── pipeline/                               # Pipeline definition
    └── pipeline_definition.json
```

## Development Setup

1. **Clone your fork:**
   ```bash
   git clone https://github.com/YOUR-USERNAME/fabric-paginated-report-sch.git
   cd fabric-paginated-report-sch
   ```

2. **Set up Microsoft Fabric:**
   - Create a workspace in Microsoft Fabric
   - Upload the notebook to your workspace
   - Configure your data source (Lakehouse, Warehouse, or Semantic Model)

3. **Configure the environment:**
   - Copy one of the config files from `/config/` directory
   - Modify the configuration for your environment
   - Update workspace IDs, report IDs, and connection strings

4. **Test your setup:**
   - Run the notebook cells sequentially
   - Verify the configuration loads correctly
   - Test with a small parameter set first

## Submitting Changes

### Pull Request Process

1. Ensure your code follows the project's coding guidelines
2. Update the README.md if you're adding new features or changing functionality
3. Add examples in the `/config/` directory if introducing new configuration options
4. Test your changes with all four parameter source modes if applicable
5. Fill out the pull request template completely
6. Link any related issues in your PR description

### What to Include in Your PR

- Clear description of what changes were made and why
- Test results or screenshots demonstrating the changes work
- Documentation updates if applicable
- Examples of new functionality

## Coding Guidelines

### Python/Notebook Style

- Follow PEP 8 style guidelines for Python code
- Use meaningful variable and function names
- Add docstrings to new classes and functions
- Keep cell outputs clear of sensitive information (tokens, IDs, etc.)

### Configuration Files

- Use valid JSON format
- Include comments (in README) explaining new configuration options
- Provide example values that are clearly placeholders
- Never commit actual workspace IDs, tokens, or credentials

### Documentation

- Update README.md for user-facing changes
- Add inline comments for complex logic
- Keep examples up-to-date with code changes
- Include parameter descriptions for new config options

## Commit Message Guidelines

Write clear, concise commit messages:

- Use the present tense ("Add feature" not "Added feature")
- Use the imperative mood ("Move cursor to..." not "Moves cursor to...")
- Limit the first line to 72 characters or less
- Reference issues and pull requests after the first line

Examples:
```
Add retry logic for API timeout errors

- Implement exponential backoff strategy
- Add max retry configuration parameter
- Update error handling in execute_report function

Fixes #123
```

## Testing Your Changes

Before submitting a pull request:

1. **Test all parameter source modes:**
   - Lakehouse (Spark SQL)
   - Semantic Model (DAX)
   - Static JSON
   - Warehouse (T-SQL)

2. **Verify error handling:**
   - Test with invalid configuration
   - Test with network interruptions
   - Test with invalid report IDs

3. **Check documentation:**
   - Ensure README is accurate
   - Verify examples work as documented
   - Update configuration examples if needed

## Questions?

Feel free to open an issue with the "Question" template if you have questions about contributing.

## Recognition

Contributors will be acknowledged in the project. Thank you for helping improve this tool!
