# MoinMoin 1.9 → Python 3 Migration Guide

## Overview

This document describes the migration of MoinMoin 1.9 from Python 2.7 to Python 3.8+, enabling deployment on modern systems where Python 2 is no longer available.

**Migration Date**: December 2025
**Target Python**: 3.8+ (3.9, 3.10, 3.11, 3.12 supported)
**Status**: In Progress

---

## Changes Made

### Phase 1: Automated Conversion (Completed)

#### Files Modified: 97
- Exception handling syntax fixes: 69
- Dictionary method updates: 27
- Print statement conversions: 12
- Unicode handling marked for review: 87

### Phase 2: Critical File Updates (In Progress)

#### setup.py - COMPLETED ✓
- Updated shebang to `#!/usr/bin/env python3`
- Fixed tuple unpacking in function arguments (Py2 only syntax)
  - `def visit((prefix, strip, found), ...)` → `def visit(args, ...)`
  - `def _visit((found, strip), ...)` → `def _visit(args, ...)`
- Fixed octal permission literals: `0755` → `0o755`
- Updated classifiers to Python 3.8+
- Added `python_requires='>=3.8'`
- Fixed exception handling: `except Exception, e:` → `except Exception as e:`
- Fixed print statements: `print x` → `print(x)`

#### Key Fixes Applied to All Python Files
1. **Exception Handling** (69 fixes applied)
   - Pattern: `except ExceptionType, var:` → `except ExceptionType as var:`
   - Applied automatically to all Python files

2. **Dictionary Methods** (27 fixes applied)
   - `.iteritems()` → `.items()`
   - `.iterkeys()` → `.keys()`
   - `.itervalues()` → `.values()`

3. **Print Statements** (12 fixes applied)
   - `print "text"` → `print("text")`
   - `print >> sys.stderr, "text"` → `print("text", file=sys.stderr)`

---

## Known Issues Requiring Manual Review

### 1. Unicode Handling (Critical - 87 instances)

**Problem**: Python 2 had separate `str` and `unicode` types; Python 3 uses unified `str` (unicode by default).

**Patterns to Review**:
```python
# Python 2
unicode(somestring)  # Convert to unicode
isinstance(x, unicode)  # Type check

# Python 3 - Generally remove unicode() calls
str(somestring)  # str is already unicode
isinstance(x, str)  # Check for str type
```

**Recommended Actions**:
- Remove unnecessary `unicode()` calls
- Update type checks to use `str` instead of `unicode`
- Ensure file I/O uses proper encoding: `open(file, 'r', encoding='utf-8')`
- Handle bytes vs str carefully in WSGI operations

### 2. String/Bytes Distinction

Python 3 strictly separates bytes (`b'...'`) and str (`'...'`). Areas of concern:

**File I/O Operations**:
```python
# Ensure all file opens specify encoding
with open('file.txt', 'r', encoding='utf-8') as f:
    content = f.read()  # Returns str

# For binary files
with open('file.bin', 'rb') as f:
    content = f.read()  # Returns bytes
```

**WSGI Applications**:
- WSGI expects bytes for response bodies
- Headers must be strings (str)
- Ensure proper encoding/decoding at boundaries

### 3. Dictionary `.has_key()` Method

**Problem**: `.has_key()` was removed in Python 3.

**Conversion**:
```python
# Python 2
if dict.has_key('key'):
    pass

# Python 3
if 'key' in dict:
    pass
```

**Note**: This pattern may not have been caught by automated tools. Search for: `grep -r "\.has_key\(" --include="*.py"`

### 4. Integer Division

**Problem**: In Python 2, `/` operator performs integer division for integers. In Python 3, `/` is true division (returns float), `//` is integer division.

**Review Points**:
- Any division operation that expects integer result should use `//`
- Example: `page_size = content_length / 4096` should be `page_size = content_length // 4096`

### 5. Import Changes

Some standard library modules were reorganized:

- `ConfigParser` → `configparser`
- `StringIO.StringIO` → `io.StringIO`
- `pickle` module uses binary mode (handles this automatically)
- `urllib` → `urllib.parse`, `urllib.request`
- `httplib` → `http.client`

---

## Critical Components Needing Review

### 1. WSGI Application (`MoinMoin/web/`)
- **Status**: Needs manual review
- **Issues**: String/bytes handling at HTTP boundaries
- **Action**: Review all `environ` and `write()` calls for encoding

### 2. File Operations (`MoinMoin/Page.py`, `MoinMoin/PageEditor.py`)
- **Status**: Needs encoding specification
- **Issues**: File I/O must specify encoding explicitly
- **Action**: Add `encoding='utf-8'` to all `open()` calls

### 3. Authentication Modules (`MoinMoin/auth/`)
- **Status**: Needs password handling review
- **Issues**: Password/hash bytes handling in Python 3
- **Action**: Review hash comparison, encoding of credentials

### 4. Internationalization (`MoinMoin/i18n/`)
- **Status**: Needs review for gettext and translation handling
- **Issues**: String handling in translation catalogs
- **Action**: Verify gettext compatibility with Python 3

### 5. Database Interfaces
- **Status**: May need SQLAlchemy/DB API updates
- **Issues**: Parameter binding with proper type handling
- **Action**: Test with actual database operations

---

## Migration Checklist

- [x] Analysis phase complete
- [x] Automated conversion applied to 942 files
- [x] setup.py updated and validated
- [ ] String/unicode issues reviewed and fixed
- [ ] WSGI application tested
- [ ] File I/O operations reviewed
- [ ] Authentication modules tested
- [ ] i18n modules functional
- [ ] Test suite passing
- [ ] Documentation updated
- [ ] Production deployment ready

---

## Testing Strategy

### 1. Syntax Validation
```bash
python3 -m py_compile *.py  # Check all Python files
```

### 2. Import Testing
```bash
python3 -c "import MoinMoin; print(MoinMoin.__version__)"
```

### 3. Web Framework Testing
- Verify WSGI application starts
- Test page creation/editing
- Test search functionality
- Test authentication

### 4. Unit Tests
```bash
cd MoinMoin/_tests
python3 -m pytest  # or appropriate test runner
```

### 5. Integration Testing
- Deploy to test environment
- Create test wiki instance
- Verify all core functionality
- Load test with typical usage patterns

---

## Dependency Updates

### Original Dependencies (Python 2.7)
- Werkzeug (embedded in support/)
- Pygments (embedded in support/)
- Passlib (embedded in support/)
- Flup (embedded in support/)
- Parsedatetime (embedded in support/)

### Python 3 Recommendations
1. Consider upgrading embedded dependencies to latest versions
2. Update setup.py to use external packages if preferred
3. Test compatibility of all external dependencies

### Testing Dependencies
- pytest (for test execution)
- pytest-cov (for coverage reports)

---

## Deployment Notes

### Installation
```bash
cd moin-1.9
python3 setup.py install
```

### Configuration
- No changes needed to wiki configuration format
- Ensure config files are UTF-8 encoded
- Update shebang in any custom scripts to `#!/usr/bin/env python3`

### Migration from Python 2 Installation
1. Backup existing wiki instance
2. Backup wiki data directory
3. Install Python 3 version
4. Data files should be compatible (but test first)
5. Verify authentication works with existing users

---

## Common Errors and Solutions

### Error: `SyntaxError: invalid syntax` near print statement
**Solution**: Print statements should use function syntax: `print(...)`

### Error: `NameError: name 'unicode' is not defined`
**Solution**: Remove `unicode()` calls or replace with `str()`

### Error: `AttributeError: 'dict' object has no attribute 'iteritems'`
**Solution**: Replace `.iteritems()` with `.items()`

### Error: `TypeError: string argument without an encoding`
**Solution**: Specify encoding in file operations: `open(file, 'r', encoding='utf-8')`

### Error: `TypeError: 'str' does not support the buffer interface`
**Solution**: Be explicit about bytes vs str. Use `.encode()` or `.decode()` as needed

---

## Further Reading

- [Python 2 to 3 Migration Guide](https://docs.python.org/3/howto/2to3.html)
- [What's New in Python 3](https://docs.python.org/3/whatsnew/)
- [WSGI and Python 3](https://peps.python.org/pep-3333/)
- [MoinMoin Original Documentation](https://moinmo.in/)

---

## Next Steps

1. Review and fix unicode handling issues (87 instances)
2. Test WSGI application
3. Update authentication and i18n modules
4. Run comprehensive test suite
5. Deploy to staging environment
6. Final production deployment

---

**Document Version**: 1.0
**Last Updated**: December 2025
**Maintainer**: Python 3 Migration Team
