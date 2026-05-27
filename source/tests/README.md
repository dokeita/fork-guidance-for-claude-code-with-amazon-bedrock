# Test Suite Documentation

This directory contains all tests for the Claude Code with Bedrock project. Tests are organized by category and use pytest as the test framework.

## Quick Start

```bash
# Navigate to source directory
cd source

# Install dependencies
uv sync

# Run all tests (Note: Some tests may fail when run together - see Troubleshooting)
uv run pytest ../tests

# Run with verbose output
uv run pytest ../tests -v

# Recommended: Run tests by category for best results
uv run pytest ../tests/lambda/test_quota_monitor.py -v      # Quota monitor tests
uv run pytest ../tests/lambda/test_metrics_aggregator.py -v  # Metrics aggregator tests
uv run pytest ../tests/cli/ -v                              # CLI command tests
uv run pytest ../tests/integration/ -v                      # Integration tests
```

## Prerequisites

Before running tests, ensure you have:

1. **Python 3.12+** installed
2. **uv** for dependency management
3. **Working directory**: Navigate to the `source` directory before running tests

```bash
cd source
uv sync  # Install all dependencies including test requirements
```

## Test Structure

```
tests/
├── cli/
│   └── commands/      # CLI command tests
│       ├── test_init.py            # Init command validation tests
│       ├── test_init_e2e.py        # Init command E2E tests
│       ├── test_init_models.py     # Init model selection tests
│       ├── test_init_quota.py      # Init quota configuration tests
│       ├── test_init_source_regions.py  # Init source region tests
│       ├── test_deploy_quota.py    # Deploy quota monitoring tests
│       ├── test_package.py         # Package command tests
│       ├── test_package_async.py   # Package async build tests
│       └── test_package_models.py  # Package model tests
├── test_cloudformation.py  # CloudFormation template validation
├── test_config.py          # Profile and config management tests
├── test_config_models.py   # Model configuration persistence tests
├── test_models.py          # Model configuration tests
├── test_smoke.py           # Comprehensive smoke tests
└── test_source_regions.py  # Source region tests
```

### Future Test Categories (Not Yet Implemented)

The following test categories are planned but not yet implemented:

- **Lambda function tests** - Tests for quota monitoring and metrics aggregation lambdas
  - Requires fixing boto3 module-level import isolation issues
- **Integration tests** - End-to-end tests with AWS services
  - Planned for LocalStack or dedicated test AWS account
- **Additional CLI commands** - Tests for deploy, destroy, distribute, quota, status commands
  - See guidance-docs/TESTING_TODO.md for full list

## Running Tests

### All Tests

```bash
# Run all tests
uv run pytest ../tests

# Run all tests with verbose output
uv run pytest ../tests -v

# Run all tests with coverage report
uv run pytest ../tests --cov=claude_code_with_bedrock --cov-report=term-missing
```

### By Category

#### CLI Command Tests
```bash
# All CLI tests
uv run pytest ../tests/cli/commands/

# Specific command tests
uv run pytest ../tests/cli/commands/test_init*.py  # Init command tests
uv run pytest ../tests/cli/commands/test_deploy*.py  # Deploy command tests
uv run pytest ../tests/cli/commands/test_package*.py  # Package command tests
```

#### Core Functionality Tests
```bash
# Model configuration tests
uv run pytest ../tests/test_models.py

# Source regions tests
uv run pytest ../tests/test_source_regions.py

# CloudFormation template validation
uv run pytest ../tests/test_cloudformation.py

# Configuration and profile tests
uv run pytest ../tests/test_config*.py

# Smoke tests (imports, instantiation)
uv run pytest ../tests/test_smoke.py
```

### Specific Test Files or Functions

```bash
# Run a specific test file
uv run pytest ../tests/lambda/test_quota_monitor.py

# Run a specific test class
uv run pytest ../tests/lambda/test_quota_monitor.py::TestQuotaMonitorLambda

# Run a specific test function
uv run pytest ../tests/lambda/test_quota_monitor.py::TestQuotaMonitorLambda::test_lambda_handler_no_usage
```

## Test Options

### Output Control

```bash
# Quiet mode (minimal output)
uv run pytest ../tests -q

# Verbose mode (detailed output)
uv run pytest ../tests -v

# Extra verbose (show test output)
uv run pytest ../tests -vv

# Show print statements during tests
uv run pytest ../tests -s

# Combined verbose with prints
uv run pytest ../tests -xvs
```

### Failure Handling

```bash
# Stop on first failure
uv run pytest ../tests -x

# Stop after N failures
uv run pytest ../tests --maxfail=3

# Show last failed tests
uv run pytest ../tests --lf

# Show failed tests first, then pass
uv run pytest ../tests --ff
```

### Performance

```bash
# Run tests in parallel (requires pytest-xdist)
uv run pytest ../tests -n auto

# Show slowest tests
uv run pytest ../tests --durations=10

# Timeout for tests (requires pytest-timeout)
uv run pytest ../tests --timeout=60
```

### Coverage Reports

```bash
# Generate coverage report
uv run pytest ../tests --cov=claude_code_with_bedrock

# Coverage with missing lines
uv run pytest ../tests --cov=claude_code_with_bedrock --cov-report=term-missing

# Generate HTML coverage report
uv run pytest ../tests --cov=claude_code_with_bedrock --cov-report=html

# Coverage for specific modules
uv run pytest ../tests --cov=claude_code_with_bedrock.cli.commands
```

## Test Categories Explained

### Unit Tests
- **test_models.py**: Tests Claude model configurations, cross-region profiles, and model ID mappings
- **test_source_regions.py**: Tests source region configurations and region availability

### CLI Command Tests
- **test_init_*.py**: Tests for the `ccwb init` command (configuration wizard)
- **test_deploy_*.py**: Tests for the `ccwb deploy` command (infrastructure deployment)
- **test_package_*.py**: Tests for the `ccwb package` command (distribution creation)

### Lambda Function Tests
- **test_quota_monitor.py**: Tests quota monitoring Lambda that checks user usage and sends alerts
- **test_metrics_aggregator.py**: Tests metrics aggregation Lambda that processes CloudWatch logs

### Integration Tests
- **test_quota_monitoring_integration.py**: End-to-end tests for quota monitoring flow

### Fixtures
- **quota_fixtures.py**: Reusable test data and mock objects for quota monitoring tests

## Common Commands Reference

```bash
# Quick test run (no output unless failures)
uv run pytest ../tests -q

# Development testing (verbose, stop on failure, show prints)
uv run pytest ../tests -xvs

# Full test suite with coverage
uv run pytest ../tests --cov=claude_code_with_bedrock --cov-report=term-missing

# Test a specific feature area
uv run pytest ../tests -k "quota"  # Run all tests with "quota" in the name

# Run tests matching a pattern
uv run pytest ../tests -k "test_deploy or test_init"

# Exclude slow tests (if marked)
uv run pytest ../tests -m "not slow"
```

## Troubleshooting

### Module Import Errors
If you encounter import errors for Lambda functions:
- The Lambda function directories use hyphens which aren't valid Python module names
- Tests handle this by adding paths to sys.path dynamically
- Ensure you're running tests from the `source` directory

### Test Isolation Issues
Lambda tests may fail when run together but pass individually due to module state contamination:
```bash
# Lambda tests should be run separately by file for best results
uv run pytest ../tests/lambda/test_quota_monitor.py -v      # ✅ All pass
uv run pytest ../tests/lambda/test_metrics_aggregator.py -v  # ✅ All pass

# Running both together may cause failures due to shared module state
uv run pytest ../tests/lambda/ -v  # ⚠️ Some tests may fail

# CLI and other tests can be run together without issues
uv run pytest ../tests/cli/ -v       # ✅ Works fine
uv run pytest ../tests/integration/ -v  # ✅ Works fine
```

**Why this happens:** Lambda functions create boto3 clients at module import time. When multiple test files import the same Lambda module with different mock configurations, the module state gets contaminated. The tests use module-scoped fixtures to minimize this issue, but complete isolation would require more complex module reloading between test files.

### Environment Variables
The test suite automatically sets AWS region to avoid boto3 errors (via `tests/conftest.py`).

For manual testing or debugging, you may need to set:
```bash
# Set test environment variables
export AWS_DEFAULT_REGION=us-east-1
export AWS_REGION=us-east-1
export AWS_PROFILE=test-profile
```

### Test Isolation
Tests should be independent and not affect each other:
- Use fixtures for setup/teardown
- Mock external dependencies
- Clean up any created resources

## Writing New Tests

### Naming Conventions
- Test files: `test_<module_name>.py`
- Test classes: `Test<ClassName>`
- Test functions: `test_<description>`

### Basic Test Structure
```python
import pytest
from unittest.mock import Mock, patch

class TestMyFeature:
    @pytest.fixture(autouse=True)
    def setup(self):
        """Setup before each test."""
        # Setup code here
        pass

    def test_feature_behavior(self):
        """Test specific behavior."""
        # Arrange
        # Act
        # Assert
        assert result == expected
```

### Mocking AWS Services
```python
@patch("boto3.client")
def test_aws_operation(mock_client):
    """Test with mocked AWS client."""
    mock_client.return_value.operation.return_value = {"Result": "Success"}
    # Test code here
```

## CI/CD Integration

For continuous integration, use:

```bash
# CI-friendly output with coverage
uv run pytest ../tests --junitxml=test-results.xml --cov=claude_code_with_bedrock --cov-report=xml

# Strict mode for CI (fail on warnings)
uv run pytest ../tests --strict-markers -W error
```

## Support

For test-related issues:
1. Check test output for detailed error messages
2. Run with `-xvs` flags for maximum debugging information
3. Review test fixtures and mocks for correctness
4. Ensure all dependencies are installed with `uv sync`