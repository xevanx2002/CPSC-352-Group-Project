# Signed Software Releases

**CPSC 352 — Cryptography — Spring 2026 — Final Project**

---

## Project Summary

This project creates a secure signed software release workflow using GitHub.

A developer pushes a signed Git tag like `v1.0`. That tag triggers GitHub Actions, which automatically:

1. Builds `Hello_World.cpp` into `Hello_World.exe`
2. Creates a SHA-256 checksum file named `SHA256SUMS`
3. Digitally signs the checksum file as `SHA256SUMS.asc`
4. Publishes all three files to a GitHub Release

A user can then verify:

- **Integrity:** `Hello_World.exe` was not changed
- **Authenticity:** the checksum file was signed by the trusted project signer

---

## Group Members & Contributions

| Member | Role | Contributions |
| --- | --- | --- |
| Evan Wenzel | Implementation | Wrote `Hello_World.cpp`, authored `.github/workflows/release.yml`, generated the project GPG key pair, configured `GPG_PRIVATE_KEY` and `GPG_PASSPHRASE` secrets, tested end-to-end release builds |
| Mánu Uribe | README & Repository Setup | Authored the README, set up the GitHub repository structure, wrote the run/verify instructions, documented the GitHub Secrets configuration |
| Tiffany Buu | Design Document — Abstract, Introduction, Design | Wrote the Abstract and Introduction sections, produced the system/component diagrams, described the high-level workflow and use cases |
| William Dam | Design Document — Security Protocols, Implementation, Conclusion | Wrote the Security Protocols section (SHA-256 + OpenPGP detached signature flow), the Implementation section (tools, languages, libraries, configuration choices), and the Conclusion |
| Jonathon Do | Video Presentation | Recorded and edited the 5–10 minute demo video, scripted the walkthrough, demonstrated tag signing, workflow run, release publication, and end-user verification |

---

## Repository Contents

```
.
├── .github/
│   └── workflows/
│       └── release.yml      # GitHub Actions workflow: builds + signs the release
├── Hello_World.cpp          # Sample C++ source
├── public-key.asc           # Public GPG key (for verifying releases)
└── README.md                # This file
```

---

## Step-by-Step Instructions to Test / Recreate Results

### 1. Clone the Repository

```bash
git clone https://github.com/xevanx2002/CPSC-352-Group-Project.git
cd CPSC-352-Group-Project
```

### 2. Confirm the Main Files Exist

You should see:

```
Hello_World.cpp
README.md
.github/workflows/release.yml
public-key.asc
```

### 3. Confirm GitHub Secrets Are Set

In GitHub, go to:

```
Repo → Settings → Secrets and variables → Actions
```

Confirm these secrets exist:

```
GPG_PRIVATE_KEY
GPG_PASSPHRASE
```

These allow GitHub Actions to digitally sign the release checksum.

### 4. Create a New Signed Release Tag

From the project folder, run:

```bash
git tag -s v1.1 -m "Version 1.1 release"
```

Verify the tag locally:

```bash
git tag -v v1.1
```

Push the tag:

```bash
git push origin v1.1
```

> The tag must start with `v` (e.g. `v1.0`, `v1.1`). The workflow trigger in `release.yml` is `tags: v*`.

### 5. Watch GitHub Actions Run

Go to:

```
GitHub Repo → Actions
```

You should see the workflow start automatically.

Expected result:

```
Build Signed Release workflow completes successfully.
```

### 6. Check the GitHub Release

Go to:

```
GitHub Repo → Releases
```

Open release `v1.1`. You should see three assets attached:

```
Hello_World.exe
SHA256SUMS
SHA256SUMS.asc
```

---

## How to Verify the Release

Download all three release files into the same directory:

```
Hello_World.exe
SHA256SUMS
SHA256SUMS.asc
```

You will also need the project's public GPG key — `public-key.asc` from this repository.

### 7. Import the Public Key (one-time setup)

```bash
gpg --import public-key.asc
```

Expected output:

```
gpg: key ABCD1234EF567890: public key "Evan Wenzel <xevanx2002@csu.fullerton.edu>" imported
gpg: Total number processed: 1
gpg:               imported: 1
```

### 8. Verify File Integrity

Run:

```bash
sha256sum -c SHA256SUMS
```

Expected output:

```
Hello_World.exe: OK
```

This proves the executable was not modified.

### 9. Verify Digital Signature

Run:

```bash
gpg --verify SHA256SUMS.asc SHA256SUMS
```

Expected output:

```
Good signature from "Evan Wenzel <xevanx2002@csu.fullerton.edu>"
```

This proves the checksum file was signed by the trusted release signer.

> A `WARNING: This key is not certified with a trusted signature!` message is normal — it just means you haven't personally signed the project's key with your own. The `Good signature` line is what matters.

### 10. Run the Executable

Once both checks pass, you can safely run the binary:

```bash
./Hello_World.exe         # On Windows
wine Hello_World.exe      # On Linux/macOS with Wine
```

### 11. Optional Tampering Test

Edit or change `Hello_World.exe`, then re-run:

```bash
sha256sum -c SHA256SUMS
```

Expected output:

```
Hello_World.exe: FAILED
```

This proves the verification process detects modified files.

### Verification Failure Reference

| Symptom | What it means |
| --- | --- |
| `gpg: BAD signature` | `SHA256SUMS` was modified. Do not trust the release. |
| `gpg: Can't check signature: No public key` | You haven't imported `public-key.asc` yet. |
| `Hello_World.exe: FAILED` | The `.exe` was modified or corrupted. Do not run it. |

---

## How the Workflow Works (Under the Hood)

When the signed tag is pushed, `.github/workflows/release.yml` runs on an Ubuntu runner and:

1. Checks out the repository (`actions/checkout@v4`).
2. Installs `mingw-w64` (Windows cross-compiler) and `gnupg`.
3. Compiles `Hello_World.cpp` → `Hello_World.exe` using `x86_64-w64-mingw32-g++`.
4. Computes `sha256sum Hello_World.exe > SHA256SUMS`.
5. Imports the private signing key from the `GPG_PRIVATE_KEY` secret.
6. Produces a detached ASCII-armored signature → `SHA256SUMS.asc`.
7. Publishes the release with `softprops/action-gh-release@v2`, attaching all three files.

The private signing key never leaves GitHub's encrypted secrets store. Only the public key (`public-key.asc`) is distributed to verifiers.

---

## Cryptographic Design

| Primitive | Algorithm | Service Provided |
| --- | --- | --- |
| Hash function | SHA-256 | **Integrity** of `Hello_World.exe` |
| Digital signature | OpenPGP detached signature | **Authenticity** + **non-repudiation** of the release |
| Signed Git tags | OpenPGP | **Authenticity** of the source revision being released |

A plain checksum file alone is not enough — an attacker who tampers with the binary could simply recompute the checksum. The detached signature on `SHA256SUMS` closes this gap: only someone holding the project's private GPG key can produce a `SHA256SUMS.asc` that verifies against `public-key.asc`. Combined with SHA-256's collision resistance, an attacker cannot substitute a malicious `Hello_World.exe` without invalidating either the hash check or the signature check.

The signed Git tag adds a second layer of authenticity: the release commit itself is tagged by a developer holding a trusted GPG key, so the build starts from a verified source state.

---

## In Simple Terms

The signed tag starts the release. GitHub Actions builds the program. SHA-256 proves the file was not changed. GPG proves the release came from the trusted signer.

---

## References

- [GitHub Actions documentation](https://docs.github.com/en/actions)
- [GnuPG manual — detached signatures](https://www.gnupg.org/gph/en/manual.html)
- [Git documentation — signed tags](https://git-scm.com/book/en/v2/Git-Tools-Signing-Your-Work)
- [`softprops/action-gh-release`](https://github.com/softprops/action-gh-release)
- Paar & Pelzl, *Understanding Cryptography*, Chapters 10 (Digital Signatures) and 11 (Hash Functions).
