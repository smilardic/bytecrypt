# bytecrypt

[![PyPI version](https://img.shields.io/pypi/v/bytecrypt.svg)](https://pypi.org/project/bytecrypt/)
[![Python versions](https://img.shields.io/pypi/pyversions/bytecrypt.svg)](https://pypi.org/project/bytecrypt/)
[![CI](https://github.com/smilardic/bytecrypt/actions/workflows/ci.yml/badge.svg)](https://github.com/smilardic/bytecrypt/actions/workflows/ci.yml)
[![License: MIT](https://img.shields.io/badge/License-MIT-blue.svg)](LICENSE)
[![PyPI Downloads](https://static.pepy.tech/personalized-badge/bytecrypt?period=total&units=INTERNATIONAL_SYSTEM&left_color=BLACK&right_color=GREEN&left_text=downloads)](https://pepy.tech/projects/bytecrypt)

bytecrypt encrypts and decrypts bytes, strings, files and full directories with
a password. It is a Python library and a command line tool.

bytecrypt derives the key with scrypt. It then seals the data with Fernet, which
is AES-128-CBC with an HMAC-SHA256 tag. The package has one dependency and runs
on any system with Python 3.11 or later. A standalone executable is also
available for users who do not have Python.

```sh
pip install bytecrypt
bytecrypt -e -f secrets.txt          # encrypt, bytecrypt asks for the password
bytecrypt -d -f secrets.txt          # decrypt
```

## Contents

- [Features](#features)
- [Installation](#installation)
- [Quick start](#quick-start)
- [Python API](#python-api)
- [Command line](#command-line)
- [How it works](#how-it-works)
- [Important notes](#important-notes)
- [Compatibility and migration](#compatibility-and-migration)
- [Development](#development)
- [License](#license)

## Features

- **Password only.** bytecrypt needs no key file and no configuration. The
  password can be a `str` or `bytes`.
- **Many targets.** bytecrypt encrypts bytes, strings, single files and full
  directory trees. It can also encrypt the file names.
- **Modern cryptography.** bytecrypt derives the key with scrypt
  (n = 2^17, r = 8, p = 1) and seals the data with authenticated encryption.
  Each salt comes from a cryptographically secure random number generator.
- **Versioned format.** Each encrypted file starts with a header that identifies
  its format. New releases continue to read the files that old releases wrote.
  The [format specification](FORMAT.md) gives the details.
- **Safe writes.** bytecrypt writes each file atomically. An incorrect password
  or an interrupted run does not corrupt or truncate the file.
- **Three ways to use it.** Import the package, run the `bytecrypt` command, or
  use the standalone executable.

## Installation

### From PyPI

```sh
pip install bytecrypt
```

bytecrypt requires Python 3.11 or later. The only dependency is
[`cryptography`](https://pypi.org/project/cryptography/).

### Standalone executable

If you do not have Python, download the latest single file executable:

| Platform | Download |
| --- | --- |
| Windows | [bytecrypt.exe](https://github.com/smilardic/bytecrypt/releases/latest/download/bytecrypt.exe) |
| Linux | [bytecrypt](https://github.com/smilardic/bytecrypt/releases/latest/download/bytecrypt) |

## Quick start

If you do not give `-p`, bytecrypt asks for the password. It asks two times when
it encrypts.

```sh
bytecrypt -e -f report.pdf                 # encrypt, bytecrypt asks for the password
bytecrypt -d -f report.pdf -p mypassword   # decrypt with a password on the command line
```

The same operations in Python:

```py
from bytecrypt import encrypt_bytes, decrypt_bytes

token = encrypt_bytes(b"top secret", "my password")
assert decrypt_bytes(token, "my password") == b"top secret"
```

## Python API

Each function accepts the password as a `str` or as `bytes`.

### Bytes

```py
from bytecrypt import encrypt_bytes, decrypt_bytes

blob = encrypt_bytes(b"secret data", "password")   # -> bytes in the versioned format
data = decrypt_bytes(blob, "password")             # -> b"secret data"
```

### Strings

`encrypt_string` returns a short base64 token.

```py
from bytecrypt import encrypt_string, decrypt_string

token = encrypt_string("secret text", "password")  # -> "AEJDWQHk..." (str)
text = decrypt_string(token, "password")           # -> "secret text"
```

### Files

bytecrypt encrypts a file in place and writes it atomically. It can also encrypt
the file name.

```py
from bytecrypt import encrypt_file, decrypt_file

encrypt_file("report.docx", "password")
decrypt_file("report.docx", "password")

# encrypt and decrypt the file name too
encrypt_file("report.docx", "password", encrypt_filename=True)
decrypt_file("<encrypted-name>", "password", decrypt_filename=True)
```

### Directories

```py
from bytecrypt import encrypt_directory, decrypt_directory

# all files in the directory; recursive=True includes the subdirectories
encrypt_directory("my/directory", "password", recursive=True)
decrypt_directory("my/directory", "password", recursive=True)

# include the file names
encrypt_directory("my/directory", "password", encrypt_filename=True, recursive=True)
```

### Migration and password change

The `reencrypt_*` functions decrypt the data and encrypt it again in memory. The
plaintext does not go to disk, and the write is atomic.

```py
from bytecrypt import reencrypt_file, reencrypt_directory

reencrypt_file("old.txt", "password")                            # upgrade a legacy file
reencrypt_file("old.txt", "password", new_password="newpass")    # change the password
reencrypt_directory("my/directory", "password", recursive=True)  # the full tree
```

### Errors

The library functions raise typed exceptions. The CLI prints a message to stderr
and exits with a non-zero code. All exceptions inherit from `BytecryptError`.

| Exception | Cause |
| --- | --- |
| `InvalidPasswordError` | The password is incorrect, or the data is damaged |
| `NotBytecryptFileError` | The input is not a bytecrypt blob |
| `UnsupportedFormatError` | The format version is unknown, so the file needs a newer bytecrypt |
| `AlreadyEncryptedError` | The data is already encrypted. Use `force=True` to encrypt it again |
| `FileNameTooLongError` | The encrypted file name is longer than the file system limit |

## Command line

```sh
# files
bytecrypt -e -f secret.txt -p mypassword
bytecrypt -d -f secret.txt                       # bytecrypt asks for the password

# file name and contents
bytecrypt -e -f secret.txt -efn -p mypassword
bytecrypt -d -f <encrypted-name> -dfn -p mypassword

# strings
bytecrypt -e -str "secret text" -p mypassword
bytecrypt -d -str "<token>" -p mypassword

# directories; -r includes the subdirectories
bytecrypt -e -dir my/directory -r -p mypassword
bytecrypt -d -dir my/directory -r -p mypassword

# write the data again in the current format, or change the password
bytecrypt --reencrypt -f old.txt -p oldpw
bytecrypt --reencrypt -dir my/directory -r -p oldpw -np newpw
```

### Arguments

| Short | Long | Description |
| --- | --- | --- |
| `-e` | `--encrypt` | Encrypt the target |
| `-d` | `--decrypt` | Decrypt the target |
| | `--reencrypt` | Write the target again in the current format, or change the password |
| `-f` | `--file` | Use a file as the target |
| `-dir` | `--directory` | Use a directory as the target |
| `-str` | `--string` | Use a string as the target |
| `-efn` | `--encrypt-filename` | Encrypt the file name too |
| `-dfn` | `--decrypt-filename` | Decrypt the file name too |
| `-p` | `--password` | The password. bytecrypt asks for it if you omit this option |
| `-np` | `--new-password` | The new password, with `--reencrypt` |
| `-r` | `--recursive` | Include the subdirectories |
| `-F` | `--force` | Encrypt the data again, also when it is already encrypted |

## How it works

- **Key derivation.** bytecrypt uses
  [scrypt](https://en.wikipedia.org/wiki/Scrypt) with n = 2^17, r = 8 and p = 1.
  These are the OWASP parameters for interactive use. The inputs are your
  password and a new 16-byte salt from a secure random number generator.
- **Encryption.** [Fernet](https://cryptography.io/en/latest/fernet/) gives
  AES-128-CBC with an HMAC-SHA256 authentication tag. bytecrypt therefore
  detects a change to the encrypted data.
- **Format.** Each encrypted blob starts with a small header:
  `magic || version || salt || token`. The version byte selects the key
  derivation parameters from a fixed table. bytecrypt does not read the cost
  parameters from the file, so a crafted file cannot cause a large memory
  allocation. [FORMAT.md](FORMAT.md) gives the full byte layout.

## Important notes

- **There is no password recovery.** If you forget the password, the data stays
  encrypted. bytecrypt has no backdoor and no reset function.
- **bytecrypt encrypts a file in place.** The write is atomic, so a half-written
  file does not occur. bytecrypt does not keep a backup copy. Keep your own
  backup of important data.
- **scrypt is slow, and this is intentional.** One operation takes approximately
  0.5 s and 128 MiB of memory. This cost makes an attack with many password
  guesses expensive. A large batch of files therefore takes time.

## Compatibility and migration

bytecrypt reads the files that the 0.x releases wrote. That legacy format uses
PBKDF2 and is weaker, but bytecrypt decrypts it without an extra step. Your
existing data continues to work. bytecrypt writes new data only in the current
scrypt format.

To move old files to the stronger format, encrypt them again:

```sh
bytecrypt --reencrypt -f old.txt -p mypassword
bytecrypt --reencrypt -dir my/directory -r -p mypassword
```

## Development

```sh
git clone https://github.com/smilardic/bytecrypt
cd bytecrypt
python -m pip install -e ".[test]"
python -m pytest                   # run the test suite
python -m yapf -i -r src tests     # format the code
```

The test suite contains regression vectors in `tests/vectors/`. They prove that
bytecrypt still decrypts the files of the old releases. [FORMAT.md](FORMAT.md)
is the format specification.

## License

bytecrypt uses the [MIT license](LICENSE). Copyright 2024-2026 Stjepan Milardić.
