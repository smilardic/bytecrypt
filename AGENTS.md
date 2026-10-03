# bytecrypt - Project Rules

This is a Python encryption package that encrypts/decrypts bytes, strings, files, and directories with a password.

## Quick Links

- **Format spec:** [FORMAT.md](FORMAT.md) - normative on-disk format specification
- **Changelog:** [CHANGELOG.md](CHANGELOG.md) - release notes
- **PyPI:** https://pypi.org/project/bytecrypt/

---

## Build, Lint, Test

```sh
# Install (editable, with test deps)
python -m pip install -e ".[test]"

# Test suite
python -m pytest

# Format (CI runs --diff and fails on drift)
python -m yapf -i -r src tests
```

## Commands

```sh
# Encrypt a file
bytecrypt -e -f test.txt -p mypassword

# Decrypt a file
bytecrypt -d -f test.txt -p mypassword

# Encrypt with encrypted filename
bytecrypt -e -f test.txt -efn -p mypassword

# Decrypt with encrypted filename
bytecrypt -d -f <encrypted-name> -dfn -p mypassword

# Encrypt directory (recursive)
bytecrypt -e -dir mydir -r -p mypassword

# Re-encrypt/migrate legacy files or change password
bytecrypt --reencrypt -f old.txt -p mypassword
bytecrypt --reencrypt -f old.txt -p mypassword -np newpassword
```

---

## Architecture

### Source Layout

- **`src/bytecrypt/`** - package source
  - **`bytecrypt.py`** - core crypto/file system logic + typed error classes
  - **`__init__.py`** - public API re-exports, version resolution
  - **`__main__.py`** - argparse CLI
  - **`err_messages.py`** - error messages + ANSI print helpers

### Package Entry Points

```python
# Library API
from bytecrypt import encrypt_bytes, decrypt_bytes
from bytecrypt import encrypt_file, decrypt_file
from bytecrypt import encrypt_directory, decrypt_directory
from bytecrypt import reencrypt_file, reencrypt_directory

# CLI: python -m bytecrypt or bytecrypt command
```

### Errors (all inherit from `BytecryptError`)

- **`InvalidPasswordError`** - wrong password or damaged data
- **`NotBytecryptFileError`** - input is not a bytecrypt blob
- **`UnsupportedFormatError`** - needs a newer bytecrypt
- **`AlreadyEncryptedError`** - data already encrypted (use `force=True`)
- **`FileNameTooLongError`** - encrypted filename exceeds 255 bytes

---

## On-Disk Format (Critical!)

**[FORMAT.md](FORMAT.md) is normative.** Follow these rules:

### Format Versions

- **Version 1** (current default): `magic || version || salt || token`
  - Magic: `b"\x00BCY"` (4 bytes)
  - Version: `0x01` (1 byte)
  - Salt: 16 bytes from `secrets.token_bytes`
  - Token: Fernet token over plaintext
  
- **Legacy** (0.x, read-only): `salt || token` (no header)
  - PBKDF2-HMAC-SHA512, 1000 iterations (weak)

### Key Rules

1. **Never change a format version silently.** A change needs:
   - New version byte in `_KDF_REGISTRY`
   - Update to [FORMAT.md](FORMAT.md) sections 1.2 and 9
   - New regression vector in `tests/vectors/`

2. **Never edit legacy vectors.** `tests/vectors/legacy-v0.3.1.bin` is from the real `v0.3.1` tag. Never edit it or regenerate it to make tests pass.

3. **Salt MUST be 16 bytes from `secrets`**, never from `random`.

4. **Do not read cost parameters from the blob.** Use the registry lookup.

5. **Atomic writes always:** temp file → `flush` + `fsync` → `os.replace`.

---

## Testing Rules

### Test Files

- **`tests/test_roundtrip.py`** - encrypt/decrypt round trips (bytes, strings, files, directories, recursive)
- **`tests/test_format.py`** - format detection, decrypt dispatch, typed errors
- **`tests/test_migration.py`** - `--reencrypt` migration, idempotency, password change
- **`tests/test_cli.py`** - CLI end-to-end, args, stdout, exit codes
- **`tests/test_vectors.py`** - decrypts real blobs from old releases

### Critical Warning

**Never modify `tests/vectors/*.bin` files.** They are real encrypted blobs from released code:
- `legacy-v0.3.1.bin` - from `v0.3.1` tag (PBKDF2 legacy format)
- `v1-scrypt.bin` - from `v1.0.0` release (current scrypt format)

A failure in `tests/test_vectors.py` is a **real backward compatibility regression**. Fix the code, don't edit the vector.

Add a new vector only when the default *write* format changes.

---

## Project Conventions

### Path Handling

- Use `os.path` (`split`, `join`, `dirname`, `abspath`) everywhere
- Use `os.walk` for directory traversal
- **Never** add manual `'/'` parsing (correct on Windows via `os.path`)
- **Never follow symbolic links**

### Password Handling

- Accept `str` or `bytes`
- Encode `str` as UTF-8 before key derivation
- No Unicode normalization (NFC is a deliberate non-goal)

### String Encoding

- File contents: binary, transported as base64 in Fernet token
- Strings: base64-encoded for transport
- Version 1 blobs start with NUL byte → don't use as text

### File Name Encryption

- Base64-encodes the encrypted name
- Raises `FileNameTooLongError` if > 255 bytes
- Uses `os.replace` to rename

### CLI Behavior

- `-p` is optional; uses `getpass` when omitted
- Encrypt: asks password twice
- Errors: prints to stderr, exits with 1 (argument errors exit with 2)
- ANSI colors enabled on Windows via `os.system("")`

---

## Security Notes

- **No password recovery** - if you forget the password, data stays encrypted
- **No backup copies** - encrypted file replaces original (atomic write)
- **scrypt is slow by design** - ~0.5s per operation, ~128 MiB memory
- **Magic detection** - warns before encrypting already-encrypted blobs (v1+)

---

## Development Workflow

1. Edit code in `src/bytecrypt/`
2. Run `python -m pytest` to verify tests
3. Run `python -m yapf -i -r src tests` to format
4. If changing format: update [FORMAT.md](FORMAT.md) + add vector
5. Commit to main branch

---

## References

- **PyPI:** https://pypi.org/project/bytecrypt/
- **GitHub:** https://github.com/smilardic/bytecrypt
- **Format spec:** [FORMAT.md](FORMAT.md)
- **Changelog:** [CHANGELOG.md](CHANGELOG.md)
- **Security:** [SECURITY.md](SECURITY.md)
