# Contributing to BITCOIN4Traders

Thank you for your interest in contributing to BITCOIN4Traders! This document provides guidelines and instructions for contributing.

## 🚀 Getting Started

### Prerequisites

- Python 3.11+
- Git
- pip

### Setting Up Development Environment

```bash
# 1. Fork the repository on GitHub
# 2. Clone your fork
git clone https://github.com/YOUR_USERNAME/BITCOIN4Traders.git
cd BITCOIN4Traders

# 3. Create a virtual environment
python -m venv venv
source venv/bin/activate  # On Windows: venv\Scripts\activate

# 4. Install in development mode
pip install -e .

# 5. Install development dependencies
pip install pytest pytest-cov black flake8 mypy
```

## 📋 GitHub Flow

Keep `main` releasable. Make changes on a short-lived branch and merge them through a Pull Request; do not push directly to `main`.

### 1. Start from the Latest Main

```bash
git switch main
git pull --ff-only origin main
git switch -c fix/short-description
```

Use a descriptive prefix such as `feat/`, `fix/`, `docs/`, `test/`, or `ci/`.

### 2. Make and Verify the Change

- Keep each branch focused on one issue or change.
- Add or update tests for behavior changes.
- Run the relevant tests and checks locally. See [the local Python workflow skill](.github/skills/local-python-workflow/SKILL.md) for environment setup.
- Never include API keys, passwords, account data, or other secrets in code, logs, screenshots, or test fixtures.
- For trading or risk changes, state the potential financial impact and use simulated or isolated testnet evidence only. Do not use live credentials for development tests.

### 3. Commit and Push the Branch

```bash
git status
git add <files>
git commit -m "fix: describe the change"
git push -u origin fix/short-description
```

Use Conventional Commit prefixes such as `feat:`, `fix:`, `docs:`, `test:`, `refactor:`, `perf:`, and `ci:`. Stage only files related to the change.

### 4. Open a Pull Request

Open a PR against `main` and complete the repository PR template. Explain the problem, the approach, tests run, and any risks or limitations. Link the related issue when one exists.

### 5. Review and Merge

- Wait for all required CI checks and resolve review comments before merging.
- Do not merge with failing or skipped required checks.
- Use the repository's selected merge strategy and delete the short-lived branch after merge.
- For changes that affect trading execution, risk limits, or secrets, request an independent review where available.

Repository administrators should protect `main` in GitHub settings by requiring Pull Requests, the `python-tests` and `frontend-checks` status checks, resolved review conversations, and up-to-date branches. Disable force pushes and branch deletion. These protections must be enabled in GitHub; they cannot be enforced by this document alone.

## 📁 Project Structure Guidelines

When adding new code, follow the existing structure:

```
src/
├── data/           # Data loading and management
├── features/       # Feature engineering
├── environment/    # Trading environment
├── math_tools/     # Mathematical models
├── risk/           # Risk management
├── agents/         # RL agents
├── training/       # Training infrastructure
├── backtesting/    # Backtesting & validation
├── validation/     # Anti-bias validation
├── costs/          # Transaction costs
├── reward/         # Reward functions
├── evaluation/     # Statistical evaluation
├── orders/         # Order management
├── execution/      # Live execution
├── connectors/     # Exchange connectors
└── monitoring/     # System monitoring
```

## 🧪 Testing Guidelines

### Writing Tests

All new code should have tests. Example:

```python
# tests/test_your_feature.py
import pytest
from src.your_module import YourClass

class TestYourFeature:
    def test_basic_functionality(self):
        obj = YourClass()
        result = obj.do_something()
        assert result == expected_value
    
    def test_edge_cases(self):
        # Test edge cases
        pass
    
    def test_error_handling(self):
        # Test error conditions
        with pytest.raises(ValueError):
            YourClass(invalid_param)
```

### Test Coverage

Aim for at least 80% code coverage for new features.

## 📝 Documentation

### Docstring Format

Use Google-style docstrings:

```python
def function_name(param1: int, param2: str) -> bool:
    """Short description of the function.
    
    Longer description if needed. Can span multiple lines
    and explain the function in detail.
    
    Args:
        param1: Description of param1
        param2: Description of param2
        
    Returns:
        Description of return value
        
    Raises:
        ValueError: When param1 is negative
        
    Example:
        >>> function_name(10, "test")
        True
    """
    pass
```

### README Updates

If you add new features, update the README.md with:
- Feature description
- Usage example
- Any new dependencies

## 🐛 Bug Reports

When reporting bugs, please include:

1. **Description**: Clear description of the bug
2. **Steps to Reproduce**: Step-by-step instructions
3. **Expected Behavior**: What should happen
4. **Actual Behavior**: What actually happens
5. **Environment**: Python version, OS, etc.
6. **Code Example**: Minimal code to reproduce the issue

Example:

```markdown
**Bug Description**
The backtesting engine crashes when processing empty dataframes.

**Steps to Reproduce**
1. Load empty dataframe
2. Run backtest
3. System crashes

**Expected Behavior**
Should return empty results or raise informative error.

**Actual Behavior**
System crashes with IndexError.

**Environment**
- Python: 3.11.4
- OS: Ubuntu 22.04
- BITCOIN4Traders: v1.0.0

**Code Example**
```python
import pandas as pd
from src.backtesting import WalkForwardEngine

df = pd.DataFrame()
engine = WalkForwardEngine()
engine.run(df)  # Crashes here
```
```

## 💡 Feature Requests

For feature requests, please:

1. Check if the feature already exists
2. Describe the use case
3. Explain why it would be useful
4. Provide example usage if possible

## 🎨 Code Style

### Python Style Guide

- Follow PEP 8
- Use type hints
- Maximum line length: 100 characters
- Use descriptive variable names
- Add comments for complex logic

### Example

```python
from typing import Optional, List
import numpy as np

class TradingStrategy:
    """Base class for trading strategies."""
    
    def __init__(self, name: str, capital: float = 10000.0):
        """Initialize strategy.
        
        Args:
            name: Strategy name
            capital: Initial capital
        """
        self.name = name
        self.capital = capital
        self.positions: List[float] = []
    
    def calculate_position_size(
        self, 
        signal: float, 
        volatility: Optional[float] = None
    ) -> float:
        """Calculate position size based on signal.
        
        Args:
            signal: Trading signal (-1 to 1)
            volatility: Optional volatility estimate
            
        Returns:
            Position size as fraction of capital
        """
        if volatility is None:
            volatility = 0.02  # Default 2%
        
        # Kelly criterion sizing
        kelly_fraction = signal / (volatility ** 2)
        
        # Limit to 25% max position
        return np.clip(kelly_fraction, -0.25, 0.25)
```

## 🔒 Security

- Never commit API keys or passwords
- Use environment variables for sensitive data
- Sanitize user inputs
- Follow security best practices

## 📜 License

By contributing, you agree that your contributions will be licensed under the MIT License.

## 🙏 Recognition

Contributors will be recognized in the README.md file.

## 📞 Questions?

If you have questions:
- Open an issue on GitHub
- Check existing documentation
- Review closed issues

Thank you for contributing to BITCOIN4Traders! 🚀
