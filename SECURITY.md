# Security policy

## Supported versions

| Version | Supported |
| --- | --- |
| 1.x | Yes |
| 0.x | No. bytecrypt 1.x still decrypts the files that 0.x wrote. Upgrade with `pip install -U bytecrypt`, then run `bytecrypt --reencrypt`. |

## How to report a vulnerability

Do not open a public issue for a security problem.

Use the private report form:
[Report a vulnerability](https://github.com/smilardic/bytecrypt/security/advisories/new).
Only you and the maintainer can read the report.

Include this information:

- the bytecrypt version, the Python version and the operating system
- the steps that reproduce the problem
- what an attacker gains
- a proof of concept, if you have one

The maintainer tries to answer within 7 days. After a fix exists, the advisory
becomes public, and it credits you if you want that. There is no bug bounty.

## In scope

- A weakness in the key derivation, in the encryption or in the on-disk format.
  [FORMAT.md](FORMAT.md) specifies that format.
- A path that writes the plaintext to disk during an encrypt operation, a
  decrypt operation or a `--reencrypt` migration.
- A way to make bytecrypt write outside the target directory, for example
  through a path or a symbolic link.
- A crash or a large memory allocation that a crafted input causes.
- A data loss path, such as a file that a failed operation truncates or
  destroys.
- A leak of the password, for example into a log, an error message or a
  temporary file.

## Out of scope

- A weak password that the user chooses. bytecrypt has no password policy.
- A lost password. There is no recovery, and this is intentional.
- The visible password in the shell history when the user passes `-p`. Omit
  `-p`, and bytecrypt asks for the password with `getpass`.
- The legacy 0.x format itself. It uses PBKDF2 with 1,000 iterations and a
  weak salt. bytecrypt reads it only for data continuity, and it never writes
  it. Migrate with `bytecrypt --reencrypt`.
- An attacker with read access to the machine memory while bytecrypt runs.
- A vulnerability in a dependency. Report it to that project. Dependabot
  watches the dependencies of this repository.
