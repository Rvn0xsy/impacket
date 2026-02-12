# AGENTS.md - Impacket Development Guide

This document provides guidelines and instructions for agents working on the Impacket codebase.

## Build, Test, and Lint Commands

### Installation
```bash
python3 -m pipx install .
python3 -m pip install tox -r requirements-test.txt
```

### Running Tests
```bash
# Run local tests only (no remote target required)
pytest -m "not remote"

# Run remote tests (requires configured AD environment)
pytest -m "remote"

# Run specific test pattern
pytest -k "ldap"

# Run with coverage
pytest --cov --cov-config=tox.ini

# Run via tox for multiple Python versions
tox -- -m "not remote"
```

### Single Test Execution
```bash
pytest tests/smb/test_smb.py::TestSMBClass::test_case
python -m unittest tests.smb.test_smb.TestSMBClass.test_case
```

### Coverage Report
```bash
coverage report
coverage html
```

## Code Style Guidelines

### Imports
- Organize imports alphabetically within groups
- Group imports: standard library, third-party, local modules
- Use explicit relative imports for internal modules
- Example:
```python
import os
import socket
from struct import pack, unpack

from impacket import nmb, ntlm, nt_errors, LOG
from impacket.structure import Structure
```

### Naming Conventions
- **Classes**: PascalCase (e.g., `SMBConnection`, `SessionError`)
- **Functions/methods**: snake_case (e.g., `negotiate_session`, `get_error_code`)
- **Constants**: UPPER_SNAKE_CASE (e.g., `SMB_DIALECT`, `STATUS_SUCCESS`)
- **Variables**: snake_case (e.g., `remote_name`, `timeout`)

### Type Hints
- Use type hints for function signatures
- Support Python 3.8+ (use `from __future__ import annotations` if needed)
- Example:
```python
def negotiateSession(self, preferredDialect: Optional[str] = None) -> bool:
```

### Error Handling
- Use `SessionError` for SMB-specific errors
- Use `UnsupportedFeature` for unimplemented protocol features
- Catch protocol-specific exceptions and re-raise as `SessionError`
- Include packet context in errors when available

### Structure Classes
- Use `Structure` base class from `impacket.structure`
- Define structure layouts as tuples in the `structure` class attribute
- Use format specifiers: `B` (uint8), `H` (uint16), `L` (uint32), `Q` (uint64), `s` (bytes), `z` (string)
- Use endianness prefixes: `<` (little-endian), `>` (big-endian)
- Example:
```python
class SMBCommand(Structure):
    structure = (
        ('WordCount', 'B=len(Parameters)//2'),
        ('Parameters',':'),
        ('ByteCount','<H-Data'),
        ('Data',':'),
    )
```

### Protocol Implementation
- Follow Microsoft protocol specifications (MS-SMB, MS-SMB2, MS-CIFS)
- Implement both SMB1 (`SMB_DIALECT = 'NT LM 0.12'`) and SMB2/3 dialects
- Use version-specific classes: `SMB` (SMB1), `SMB3` (SMB2/3)
- Handle dialect negotiation transparently in `SMBConnection`

### Security Considerations
- Never log or expose secrets, keys, or hashes
- Use cryptographic functions from `hashlib` and `pycryptodomex`
- Support signing and encryption as per protocol specs
- Validate all input data before processing

### File Organization
- Core protocols: `impacket/smb.py`, `impacket/smb3.py`, `impacket/smb3structs.py`
- Connection wrapper: `impacket/smbconnection.py`
- Protocol structures: `impacket/smb3structs.py`
- Examples: `examples/*.py`

### Common Patterns
- Use `NewSMBPacket` for constructing SMB packets
- Use `SMBCommand` for individual SMB commands
- Use `SMBAndXCommand_Parameters` for chained commands
- Implement `isValidAnswer()` to validate responses

### Documentation
- Document all public classes and methods with docstrings
- Include parameter types and return types
- Reference protocol specifications when applicable
- Use informative error messages

## SMB Negotiation Implementation

### Key Files
- `impacket/smbconnection.py:95-174` - Main negotiation entry point
- `impacket/smbconnection.py:176-218` - Wildcard negotiation logic
- `impacket/smb.py:2999-3100` - SMB1 negotiate command
- `impacket/smb3.py:560-670` - SMB2/3 negotiateSession method

### Negotiation Data Format
- Default negotiate data includes: `PC NETWORK PROGRAM 1.0`, `LANMAN1.0`, `Windows for Workgroups 3.1a`, `LM1.2X002`, `LANMAN2.1`, `NT LM 0.12`, `SMB 2.002`, `SMB 2???`
- Port 139 uses NETBIOS session with different negotiate data than port 445
- Response parsing handles both SMB1 (`0xff` prefix) and SMB2 (`0xfe` prefix) responses

### Key Configuration Points
- `flags1`: SMB FLAGS1 (pathcaseless, canonicalized paths)
- `flags2`: SMB FLAGS2 (extended security, NT status, long names, unicode)
- `extended_security`: Boolean for SPNEGO/NTLMSSP usage

## Testing Notes
- Remote tests require AD domain controller configuration
- Use `tests/dcetests.cfg.template` for test configuration
- Set `REMOTE_CONFIG` environment variable for test targets
- Some tests are not idempotent (may modify target environment)
