+++
title = "Python Progress Bar"
date = 2025-08-25
lastmod = 2025-08-25T10:09:17+08:00
tags = ["python", "schema", "dataclass"]
categories = ["python", "schema", "dataclass"]
draft = false
author = "B40yd"
+++

# Schemas DataClass

A lightweight, Python 2/3 compatible data validation library that brings powerful schema validation to Python classes with a clean decorator-based API. Schemas DataClass combines the simplicity of Python dataclasses with robust validation capabilities, making it ideal for data processing, API validation, and configuration management.

## 🚀 Why Choose Schemas DataClass?

- **Seamless Python 2/3 Compatibility**: Works flawlessly across Python 2.7+ and 3.4+
- **Type-Safe Data Validation**: Comprehensive validation for strings, numbers, lists, and nested objects
- **Clean, Pythonic API**: Use standard class syntax with decorator-based validation
- **Customizable Error Messages**: Internationalization-ready with template formatting
- **Zero Dependencies**: Pure Python implementation using only standard libraries
- **Efficient Performance**: Lightweight design with minimal overhead
- **Recursive Validation Protection**: Safely handles nested and circular references

## 📦 Installation

### From PyPI (Recommended)

```bash
pip install schemas-dataclass
```

### From Source

```bash
git clone https://github.com/b40yd/schema_dataclass.git
cd dataclass
python setup.py install
```

### For Development

```bash
git clone https://github.com/b40yd/schema_dataclass.git
cd dataclass
pip install -e .
pip install -r requirements-dev.txt
```

## 🚀 Quick Start

### Basic Usage

```python
from schema_dataclass import StringField, NumberField, dataclass, ValidationError

@dataclass
class User(object):
    name = StringField(min_length=2, max_length=50)
    age = NumberField(minvalue=0, maxvalue=120)
    email = StringField(
        regex=r'^[a-zA-Z0-9._%+-]+@[a-zA-Z0-9.-]+\.[a-zA-Z]{2,}$'
    )

# Create a user
user = User(name="Alice", age=25, email="alice@example.com")
print(user.to_dict())  # {'name': 'Alice', 'age': 25, 'email': 'alice@example.com'}
```

### Custom Error Messages

```python
@dataclass
class User(object):
    name = StringField(
        min_length=2,
        error_messages={
            'required': 'Name is required',
            'min_length': 'Name must be at least {min_length} characters long'
        }
    )

try:
    user = User(name="A")  # Too short
except ValidationError as e:
    print(e.message)  # Output: Name must be at least 2 characters long
```

## 📚 Documentation Index

- **[Installation and Usage](#installation-and-usage)** - Getting started guide
- **[Core Features](#core-features)** - Key capabilities and design principles
- **[Complete Examples](#complete-examples)** - Practical usage scenarios
- **[API Reference](#api-reference)** - Detailed API documentation
- **[Validation Features](#validation-features)** - Comprehensive validation capabilities
- **[Testing](#testing)** - How to run tests
- **[Compatibility](#compatibility)** - Supported Python versions
- **[Performance](#performance)** - Efficiency characteristics
- **[Contributing](#contributing)** - How to contribute to the project
- **[License](#license)** - Licensing information
- **[Changelog](#changelog)** - Version history and updates

## Installation and Usage

### Basic Usage Guide

```python
from schema_dataclass import StringField, NumberField, ListField, dataclass

@dataclass
class User(object):
    name = StringField(min_length=2, max_length=50)
    age = NumberField(minvalue=0, maxvalue=120)
    email = StringField(
        regex=r'^[a-zA-Z0-9._%+-]+@[a-zA-Z0-9.-]+\.[a-zA-Z]{2,}$'
    )
    tags = ListField(item_type=str, required=False)

# Create and use
user = User(
    name="Alice",
    age=25,
    email="alice@example.com",
    tags=["developer", "python"]
)

print(user.name)        # Alice
print(user['age'])      # 25
print(user.get('email')) # alice@example.com
print(user.to_dict())   # Convert to dictionary
```

## Core Features

### 🔧 Field Types Support

- **StringField**: String validation
  - Length constraints (`min_length`, `max_length`)
  - Regular expression validation (`regex`)
  - Enumeration validation (`choices`)
  - Custom error messages with template formatting
  
- **NumberField**: Numeric validation (int, float, long)
  - Range validation (`minvalue`, `maxvalue`)
  - Enumeration validation (`choices`)
  - Type validation with automatic coercion
  - Custom error messages with template formatting
  
- **ListField**: List validation
  - Length constraints (`min_length`, `max_length`)
  - Item type validation (`item_type`)
  - Supports nested types including strings, numbers, and dataclass models
  - Custom error messages with template formatting

### 🌍 Custom Error Messages

- **Multi-language Ready**: Supports internationalization with locale-aware messages
- **Template Formatting**: Use `{parameter}` style formatting for dynamic messages
- **Complete Coverage**: Customize error messages for all validation types
- **Backward Compatible**: Optional feature that doesn't affect existing code

```python
# Custom error messages example
@dataclass
class User(object):
    name = StringField(
        min_length=3,
        max_length=20,
        error_messages={
            'required': 'Username is required',
            'min_length': 'Username must be at least {min_length} characters long',
            'max_length': 'Username cannot exceed {max_length} characters'
        }
    )
```

### 🎯 Decorator Syntax

```python
@dataclass
class User(object):
    name = StringField(min_length=1, max_length=100)
    age = NumberField(minvalue=0, maxvalue=150)
```

### 🔍 Custom Validation Decorator

```python
@dataclass
class Product(object):
    name = StringField()
    price = NumberField()
    
    @validate("name")
    def validate_name_custom(self, name):
        if not name.isalnum():
            raise ValidationError("Name must be alphanumeric")
    
    @validate("price")
    def validate_price_custom(self, price):
        if price <= 0:
            raise ValidationError("Price must be positive")
```

### 🔧 Custom Get Methods

```python
@dataclass
class BlogPost(object):
    title = StringField()
    status = StringField(default='draft')
    
    def get_title(self):
        """Custom method to get formatted title"""
        title = self.__dict__.get('title', '')
        status = self.__dict__.get('status', 'draft')
        return "[{0}] {1}".format(status.upper(), title)
```

### 🌐 Nested DataClass Support

```python
@dataclass
class Address(object):
    street = StringField()
    city = StringField()
    zip_code = StringField()

@dataclass
class User(object):
    name = StringField()
    address = Address  # Class reference (auto-instantiated)
    addresses = ListField(item_type=Address)  # List of nested objects
```

## Complete Examples

### 📁 Example Files

The project provides rich examples in the `examples/` directory:

- **[Basic Usage Example](examples/basic_usage.py)** - Field types, basic dataclass functionality
- **[Custom Error Messages Example](examples/custom_error_messages.py)** - Multi-language messages, template formatting
- **[Advanced Features Example](examples/advanced_features.py)** - Custom validation, nested dataclasses, conditional validation
- **[Real World Examples](examples/real_world_examples.py)** - User management, e-commerce products, blog systems

### 🚀 Running Examples

```bash
# Basic usage example
python examples/basic_usage.py

# Custom error messages example
python examples/custom_error_messages.py

# Advanced features example
python examples/advanced_features.py

# Real world examples
python examples/real_world_examples.py
```

### 💡 Quick Example

#### User Management System

```python
from schema_dataclass import StringField, NumberField, ListField, dataclass, validate

@dataclass
class User(object):
    username = StringField(
        min_length=3,
        max_length=20,
        regex=r'^[a-zA-Z][a-zA-Z0-9_]*$',
        error_messages={
            'required': 'Username is required',
            'min_length': 'Username must be at least {min_length} characters long',
            'regex': 'Username must start with a letter and contain only letters, numbers, and underscores'
        }
    )
    
    email = StringField(
        regex=r'^[a-zA-Z0-9._%+-]+@[a-zA-Z0-9.-]+\.[a-zA-Z]{2,}$',
        error_messages={
            'required': 'Email address is required',
            'regex': 'Please enter a valid email address'
        }
    )
    
    age = NumberField(
        minvalue=13,
        maxvalue=120,
        error_messages={
            'minvalue': 'Age cannot be less than {minvalue}',
            'maxvalue': 'Age cannot be greater than {maxvalue}'
        }
    )
    
    tags = ListField(
        item_type=str,
        required=False,
        max_length=10,
        error_messages={
            'max_length': 'Cannot have more than {max_length} tags'
        }
    )
    
    @validate("username")
    def validate_username_not_reserved(self, username):
        """Check if username is a reserved word"""
        reserved = ['admin', 'root', 'system']
        if username.lower() in reserved:
            raise ValidationError(f"Username '{username}' is a reserved word")

# Usage example
user = User(
    username="alice_dev",
    email="alice@example.com",
    age=28,
    tags=["developer", "python"]
)

print("User: {}".format(user.username))
print("Email: {}".format(user.email))
print("Age: {}".format(user.age))
print("Tags: {}".format(user.tags))
```

## API Reference

> **Important Change Notice**: Starting from version 2.0, all fields are optional by default (`required=False`). For required fields, explicitly set `required=True`.

### Field Types

#### StringField

```python
StringField(
    default=None,           # Default value
    alias=None,            # Field alias
    required=False,        # Whether the field is required (default: False)
    min_length=None,       # Minimum length
    max_length=None,       # Maximum length
    regex=None,            # Regular expression pattern
    choices=None,          # Enumeration options
    error_messages=None    # Custom error messages
)
```

#### NumberField

```python
NumberField(
    default=None,           # Default value
    alias=None,            # Field alias
    required=False,        # Whether the field is required (default: False)
    minvalue=None,         # Minimum value
    maxvalue=None,         # Maximum value
    choices=None,          # Enumeration options
    error_messages=None    # Custom error messages
)
```

#### ListField

```python
ListField(
    default=None,           # Default value
    alias=None,            # Field alias
    required=False,        # Whether the field is required (default: False)
    min_length=None,       # Minimum length
    max_length=None,       # Maximum length
    item_type=None,        # Type of list items
    error_messages=None    # Custom error messages
)
```

### Decorators

#### @dataclass

```python
@dataclass
class MyClass(object):
    field1 = StringField()
    field2 = NumberField()
```

#### @validate

```python
@dataclass
class MyClass(object):
    field1 = StringField()

    @validate("field1")
    def validate_field1(self, value):
        # Custom validation logic
        if not condition:
            raise ValidationError("Custom validation failed")
```

### Error Message Keys

#### Common Error Message Keys

- `required`: Required field is empty
- `invalid_type`: Type mismatch

#### StringField Error Message Keys

- `min_length`: Length below minimum
- `max_length`: Length above maximum
- `regex`: Regular expression mismatch
- `choices`: Value not in enumeration options

#### NumberField Error Message Keys

- `minvalue`: Value below minimum
- `maxvalue`: Value above maximum
- `choices`: Value not in enumeration options

#### ListField Error Message Keys

- `min_length`: List length below minimum
- `max_length`: List length above maximum
- `invalid_list_item`: List item type mismatch

## Validation Features

### String Validation

- Length validation: `min_length`, `max_length`
- Regular expression validation: `regex`
- Enumeration validation: `choices`
- Custom error messages for all validation types

### Number Validation

- Range validation: `minvalue`, `maxvalue`
- Enumeration validation: `choices`
- Type validation: automatic support for int, float, long (Python 2)
- Custom error messages for all validation types

### List Validation

- Length validation: `min_length`, `max_length`
- Item type validation: `item_type`
- Supports nesting: strings, numbers, dataclass models
- Custom error messages for list item type errors

### DataClass Field Support

- Support dataclass as field types
- Automatic instantiation and validation
- Re-creation of objects on reassignment
- Support nested `to_dict()` conversion
- Validation on reassignment

### Custom Validation

- Use `@validate("field_name")` decorator
- Executed after basic validation
- Support multiple custom validation functions

### Custom Error Messages Features

- **Multi-language Support**: Full support for Chinese, English, and other languages
- **Template Formatting**: Support `{parameter}` style parameter replacement
- **Complete Coverage**: Support custom error messages for all validation types
- **Backward Compatibility**: Doesn't affect existing code, optional usage
- **Robustness**: Graceful degradation when formatting fails, returns original template
- **Zero Performance Impact**: Same performance as original version when not using custom messages

#### Supported Error Message Types

- **Common**: `required`, `invalid_type`
- **StringField**: `min_length`, `max_length`, `regex`, `choices`
- **NumberField**: `minvalue`, `maxvalue`, `choices`
- **ListField**: `min_length`, `max_length`, `invalid_list_item`

## Testing

### Running Tests

```bash
# Run all tests
pytest

# Run specific test file
pytest tests/test_fields.py

# Run tests with coverage
pytest --cov=schema_dataclass

# Run tests with specific markers
pytest -m "unit"
pytest -m "integration"
pytest -m "error_messages"
```

### Test Structure

```
tests/
├── conftest.py                    # pytest configuration and fixtures
├── test_fields.py                 # Field type tests
├── test_custom_error_messages.py  # Custom error messages tests
├── test_dataclass.py             # dataclass functionality tests
└── test_integration.py           # Integration tests
```

### Test Coverage

- **25+ test cases** covering all functionality
- **100% test pass rate**
- **Backward compatibility verification**
- **Multi-language error message tests**
- **Complex scenario boundary testing**

## Compatibility

- **Python 2.7+**: Fully supported
- **Python 3.4+**: Fully supported
- **PyPy**: Supported
- **Jython**: Theoretically supported (untested)

## Performance

- **Zero Dependencies**: Uses only Python standard library
- **Lightweight**: Core code under 1000 lines
- **High Performance**: Fast validation with low memory usage
- **Extensible**: Easy to add new field types and validation rules

## Contributing

Contributions are welcome! Please follow these steps:

1. Fork the project
2. Create a feature branch (`git checkout -b feature/amazing-feature`)
3. Add test cases
4. Ensure all tests pass (`pytest`)
5. Update relevant documentation
6. Commit your changes (`git commit -m 'Add amazing feature'`)
7. Push to the branch (`git push origin feature/amazing-feature`)
8. Create a Pull Request

### Development Environment Setup

```bash
git clone https://github.com/b40yd/schema_dataclass.git
cd dataclass
pip install -e .
pip install -r requirements-dev.txt
```

### Code Guidelines

- Follow PEP 8 coding style
- Add appropriate docstrings
- Add test cases for new features
- Maintain Python 2/3 compatibility

## License

This project is licensed under the GNU General Public License v3.0. See the [LICENSE](LICENSE) file for details.
