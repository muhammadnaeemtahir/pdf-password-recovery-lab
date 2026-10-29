# PDF Password Recovery and Security Analysis Using John the Ripper

**Project category:** Cybersecurity · Password Auditing · Linux Security  
**Project type:** Hands-on security lab  
**Status:** Completed  
**Environment:** Ubuntu Linux  
**Primary tools:** John the Ripper Jumbo, `pdf2john.py`, `qpdf`, Python, pyHanko  
**Authorization:** Personal lab exercise using a PDF created by the author

---

## 1. Executive Summary

This project demonstrates a practical password-recovery workflow for a password-protected PDF document in a controlled environment.

The objective was to understand how password-protected PDF files can be inspected, how a password-verification hash can be extracted, and how a password-auditing tool can test candidate passwords against that hash.

The workflow used Ubuntu Linux as the lab platform. PDF inspection utilities were used to examine the document's encryption characteristics. The `pdf2john.py` utility from John the Ripper Jumbo extracted a hash in a format supported by John. A bundled wordlist was tested first, followed by a password-mutation rule set. The second attempt successfully recovered the password.

Finally, `qpdf` independently validated the recovered password against the original PDF.

This exercise provided practical experience with Linux command-line tools, password auditing, PDF security, Python dependencies, troubleshooting, and technical documentation.

### Final outcome

- Extracted a password hash from the test PDF.
- Confirmed that John the Ripper Jumbo recognized the PDF hash format.
- Tested a bundled wordlist.
- Applied password-mutation rules after the initial attempt failed.
- Successfully recovered the password.
- Validated the password against the original PDF using `qpdf`.

---

## 2. Project Objectives

The project was designed around the following learning objectives:

1. **Understand PDF encryption metadata.** Inspect a password-protected PDF and identify its reported encryption characteristics.
2. **Configure a password-auditing environment.** Install the necessary utilities and build a version of John the Ripper with PDF support.
3. **Extract a password hash.** Convert the PDF's password-verification information into a format accepted by John.
4. **Perform wordlist-based testing.** Test candidate passwords using a wordlist supplied with John Jumbo.
5. **Apply password-mutation rules.** Generate additional candidates based on transformations of existing words.
6. **Validate the result.** Confirm that the recovered password is accepted by the original PDF.
7. **Document the findings responsibly.** Record the process and results without publishing sensitive lab artifacts.

---

## 3. Scope and Authorization

The test was performed on a PDF created by the author. The exercise was intended for learning and personal password recovery, not for accessing another person's protected documents.

The project covers:

- Local analysis of a PDF file.
- Inspection of PDF encryption settings.
- Password-hash extraction.
- Offline candidate-password testing.
- Verification of the recovered password.
- Secure handling of generated artifacts.

The project does not attempt to exploit a remote system, bypass access controls on a third-party service, or obtain data from an unauthorized source.

The original PDF, extracted hash, recovered password, and password-results file are intentionally excluded from the public repository.

---

## 4. Lab Environment

### Operating environment

| Component | Configuration |
|---|---|
| Operating system | Ubuntu Linux |
| Shell | Bash-compatible terminal |
| Python | Python environment with pyHanko available |
| Password-auditing tool | John the Ripper Jumbo |
| PDF inspection utility | qpdf |
| PDF hash extractor | `pdf2john.py` |
| Test document | Locally stored, password-protected PDF |

### Why John the Ripper Jumbo?

John the Ripper is a password-auditing tool that can test candidate passwords against supported password hashes. The Jumbo edition provides additional formats and helper utilities, including support for PDF password auditing.

The standard Ubuntu `john` package installed during the initial setup did not include the required PDF helper script. To obtain the necessary functionality, I built John the Ripper Jumbo from its source repository.

### Why qpdf?

`qpdf` provides utilities for inspecting PDF encryption and checking whether a supplied password is accepted. It was used both during initial inspection and during final validation.

### Why pdf2john.py?

John cannot directly treat every encrypted file as an ordinary text password hash. The helper extracts the relevant password-verification data from a supported PDF and formats it for John.

---

## 5. Technical Background

### 5.1 PDF password protection

PDF files can use encryption to restrict access to their contents and operations. Depending on the document's configuration, a PDF may require a password to open it or may use permissions that restrict actions such as printing, copying, or modification.

PDF encryption metadata can include:

- The security-handler revision.
- The encryption key length.
- Document permissions.
- The encryption algorithms used for streams and strings.
- Whether a supplied password is accepted as a user password or owner password.

These values help describe the document's security configuration. They do not, by themselves, reveal the original password.

### 5.2 RC4 and revision 4

The test document's inspection output reported:

- PDF version: 1.4
- Security-handler revision: 4
- Key length: 128 bits
- Encryption method: RC4

RC4 is a legacy stream cipher and is no longer considered suitable for modern cryptographic protection. Its presence is a useful observation about the age and configuration of the PDF's encryption scheme.

However, the presence of RC4 does not mean the password can automatically be recovered. Recovery still depends on the document's password-verification mechanism, the candidate passwords tested, and the time and computing resources available.

### 5.3 Password hashes and offline auditing

A password hash or password-verification value can allow a password-auditing tool to test guesses without repeatedly opening the document through a graphical application.

At a high level, the workflow is:

1. Extract the PDF's password-verification data.
2. Generate a candidate password.
3. Apply the PDF's verification procedure to that candidate.
4. Compare the result with the stored verification data.
5. Repeat until a candidate matches or the search ends.

The extracted value is sensitive because it can support offline password guessing. It must therefore be treated as a security-sensitive artifact.

### 5.4 Wordlists and mutation rules

A wordlist contains candidate passwords. A password-mutation rule transforms a candidate into variants—for example, by changing capitalization, appending digits, or applying other common modifications.

This can improve the coverage of a wordlist-based attempt, but it does not guarantee recovery. A password that is absent from the generated candidates will not be found by that attempt.

---

## 6. Implementation and Procedure

### Phase 1 — Install and inspect the basic utilities

The initial environment was prepared with the standard Ubuntu package manager:

```bash
sudo apt update
sudo apt install john poppler-utils qpdf
```

The tools were then checked from the terminal.

During setup, the standard Ubuntu `john` package did not provide the required PDF extraction helper. This established the need for the Jumbo build.

**Learning outcome:** The availability of a feature depends on the installed build and its supported formats, not just on the tool's name.

### Phase 2 — Build John the Ripper Jumbo

The Jumbo source repository was cloned and compiled:

```bash
git clone --depth 1 \
  https://github.com/openwall/john \
  -b bleeding-jumbo john-jumbo

cd john-jumbo/src
./configure
make -s clean
make -sj2
```

After compilation, I moved to the runtime directory:

```bash
cd ~/john-jumbo/run
```

I checked whether the build supported the PDF format:

```bash
./john --list=formats | grep -i PDF
```

The output included `PDF`, confirming that the format was available in the compiled build.

The `pdf2john.py` helper was also present in the runtime directory.

**Learning outcome:** Source builds can provide functionality missing from distribution packages, but compiling software introduces additional dependency and maintenance considerations.

### Phase 3 — Resolve the Python dependency

The initial hash-extraction attempt reported that pyHanko was missing. The output file was consequently empty.

After installing the dependency, I verified that Python could import it:

```bash
python -c "import pyhanko; print('pyHanko import successful')"
```

The command completed successfully.

**Troubleshooting note:** Redirecting command output to a file can leave an empty file when a program fails. Always check the program's error output and verify the generated file before continuing.

For a reproducible lab, Python dependencies should ideally be installed in a dedicated virtual environment with their versions recorded.

### Phase 4 — Inspect the protected PDF

I used PDF utilities to inspect the document:

```bash
pdfinfo "$HOME/Downloads/target.pdf"
```

The document required a password for normal inspection. I then examined its encryption information:

```bash
qpdf --show-encryption "$HOME/Downloads/target.pdf"
```

The output reported revision 4, a 128-bit key length, RC4 encryption, and several restricted document operations.

This established the encryption characteristics needed to describe the test case.

**Observation:** Encryption metadata can often be inspected without knowing the correct password, although some operations and document details may remain unavailable.

### Phase 5 — Extract the PDF password hash

From the John Jumbo runtime directory, I ran the PDF hash-extraction helper:

```bash
python pdf2john.py \
  "$HOME/Downloads/target.pdf" \
  > "$HOME/target.hash"
```

I verified that the output file was not empty:

```bash
wc -c "$HOME/target.hash"
```

The file contained 193 bytes.

I also checked the beginning of the extracted value and confirmed that it had the expected PDF-hash prefix. I then verified that the John Jumbo build recognized the PDF format.

**Result:** Hash extraction succeeded, and the generated file was ready for password auditing.

**Security note:** The extracted hash was retained locally and is not included in this repository.

### Phase 6 — Attempt recovery with a bundled wordlist

The first candidate-generation attempt used the wordlist supplied with John Jumbo:

```bash
./john \
  --format=PDF \
  --wordlist=./password.lst \
  "$HOME/target.hash"
```

John loaded one PDF password hash and processed candidates. The session finished without recovering the password.

This was a useful result rather than a tool failure. It showed that the candidates generated by the initial wordlist did not produce a successful match.

**Learning outcome:** A completed wordlist run is not proof that a password is secure or unrecoverable. It only describes the candidate set tested in that run.

### Phase 7 — Apply password-mutation rules

After the initial wordlist attempt, I used the bundled `rockyou-30000` rule set:

```bash
./john \
  --format=PDF \
  --wordlist=./password.lst \
  --rules=rockyou-30000 \
  "$HOME/target.hash"
```

This attempt reported a successful password recovery.

The result demonstrated that candidate generation matters: the rule-based run tested variants that were not successfully matched during the initial wordlist-only attempt.

**Result:** One PDF password hash was recovered.

The actual password is intentionally omitted from this report.

### Phase 8 — Confirm the recovered result

I checked John's saved recovery results:

```bash
./john --show --format=PDF "$HOME/target.hash"
```

John reported that one hash had been cracked.

To independently verify the result, I used `qpdf` to check the original document. In the interactive terminal, I supplied the recovered password and ran the PDF check:

```bash
qpdf --password='[RECOVERED_PASSWORD]' \
  --check "$HOME/Downloads/target.pdf"
```

The tool confirmed that the supplied password was accepted as the PDF's user password. It also reported that no syntax or stream-encoding errors were found.

**Important distinction:** This validation confirms that the recovered password was accepted and that qpdf found no such errors. It does not prove the document is free of every possible defect.

---

## 7. Results and Evidence

### Summary of observed results

| Test | Result |
|---|---|
| PDF encryption inspection | Completed |
| John Jumbo PDF-format check | Passed |
| Python dependency check | Passed |
| Hash extraction | Successful |
| Initial wordlist attempt | No password recovered |
| Wordlist plus mutation rules | Password recovered |
| John recovery-result check | One hash cracked |
| qpdf password validation | Successful |
| qpdf syntax and stream-encoding check | No errors reported |

### Evidence suitable for publication

For a public repository, include sanitized screenshots or terminal excerpts demonstrating:

1. The PDF encryption metadata.
2. The successful PDF-format check.
3. The non-empty hash-file size, without displaying the hash.
4. The initial wordlist attempt finishing without recovery.
5. The rule-based attempt reporting a successful recovery, with the password redacted.
6. The final qpdf validation output, with the password redacted.

Never publish the original protected PDF, the recovered password, the complete extracted hash, or `john.pot`.

---

## 8. Troubleshooting and Lessons Learned

### Problem 1 — PDF inspection required a password

**Symptom:** `pdfinfo` reported that the password was incorrect or required.

**Resolution:** I used qpdf's encryption-inspection functionality to examine the document's encryption settings.

**Lesson:** Different utilities expose different kinds of PDF metadata, and a failed metadata operation does not necessarily indicate a corrupted file.

### Problem 2 — The distribution package lacked the required helper

**Symptom:** The installed John package did not include the PDF extraction script.

**Resolution:** I compiled John the Ripper Jumbo from source.

**Lesson:** Verify format support and helper availability before designing a workflow around a particular software package.

### Problem 3 — The extraction script reported a missing Python dependency

**Symptom:** The first extraction attempt reported that pyHanko was missing and generated an empty hash file.

**Resolution:** I installed the required Python dependency and confirmed that it could be imported.

**Lesson:** Check dependencies and output files before assuming a command succeeded. For a more reproducible environment, isolate dependencies and record versions.

### Problem 4 — The expected wordlist was not installed system-wide

**Symptom:** The expected `rockyou.txt` path did not exist, and the package manager could not locate the attempted package.

**Resolution:** I used the `password.lst` file included with John Jumbo instead.

**Lesson:** A tool's own runtime directory may contain useful resources even when common system paths do not.

### Problem 5 — The first wordlist attempt did not recover the password

**Symptom:** John completed its run but reported no recovered hash.

**Resolution:** I applied the bundled `rockyou-30000` mutation rules to the wordlist.

**Lesson:** Password auditing depends on the candidates generated, not merely on the number of times a tool is run.

---

## 9. Security Analysis

### Why the exercise matters

Password protection is only as resistant to guessing as the effective password-verification scheme, password choice, and attacker's candidate-generation strategy allow.

A password that follows common patterns may appear in a wordlist or be generated by mutation rules. This is why passwords based on common words, predictable substitutions, or short numeric suffixes are often poor choices.

### Defensive recommendations

- Use long, unique passwords generated or stored by a reputable password manager.
- Avoid predictable word-and-number combinations.
- Use modern PDF encryption settings supported by the software and required by the security policy.
- Protect sensitive PDFs with appropriate access controls in addition to encryption.
- Store password-recovery artifacts securely and delete temporary files when they are no longer needed.
- Do not distribute hashes or password files with a public project.
- Treat a password recovery success as a reason to review password policy, not as proof that all PDF files are equally vulnerable.

### Important technical limitation

The project demonstrated recovery for one specific document. It does not establish that all PDFs using revision-4 security or RC4 encryption can be recovered quickly. Recovery difficulty varies with the password, document configuration, candidate-generation strategy, and available computing resources.

---

## 10. Ethical Considerations

All testing was performed on a document created by the author.

Password auditing should be limited to files and systems for which explicit authorization exists. Extracted hashes can enable offline guessing, so they should be treated as sensitive even when they are not the original document.

This public report is intentionally sanitized. It preserves the methodology and results while withholding the recovered password, complete hash, original PDF, and John the Ripper's results file.

---

## 11. Skills Demonstrated

### Technical skills

- Linux command-line usage
- Package installation and troubleshooting
- Building security tools from source
- Python dependency troubleshooting
- PDF encryption inspection
- Password-hash extraction
- Wordlist-based password auditing
- Password-mutation rules
- Validation using independent utilities
- Secure handling of sensitive test artifacts

### Professional skills

- Structured problem-solving
- Evidence-based reporting
- Technical documentation
- Reproducible workflow design
- Security-conscious handling of artifacts
- Clear communication of technical limitations

---

## 12. Future Improvements

Possible extensions for future lab iterations include:

1. Record exact tool versions and system configuration.
2. Create a dependency file or setup script for a reproducible environment.
3. Add sanitized screenshots with captions.
4. Compare wordlist-only and rule-based candidate-generation approaches.
5. Measure elapsed time and candidate rates while documenting the limitations of those measurements.
6. Repeat the experiment using self-generated test PDFs with different password lengths and controlled password patterns.
7. Compare supported PDF encryption configurations and document their security implications.
8. Add a repository checklist to prevent accidental publication of sensitive artifacts.

Any future tests should use self-created files or other explicitly authorized test data.

---

## 13. Conclusion

This project provided hands-on experience with a complete PDF password-recovery workflow, from inspecting encryption metadata through extracting a password hash, testing candidate passwords, applying mutation rules, and independently validating the result.

The most important practical lesson was that a failed wordlist attempt does not necessarily end a password-recovery exercise. Changing the candidate-generation strategy led to success in this lab.

The project also reinforced the importance of validating tool output, resolving dependencies, understanding the limits of test results, and protecting sensitive artifacts.

It serves as an introductory cybersecurity portfolio project demonstrating practical Linux security skills, password-auditing concepts, troubleshooting, and responsible technical reporting.

---

## Repository Safety Checklist

Before publishing, verify that the repository does **not** contain:

- The original protected PDF.
- The recovered password.
- The complete extracted hash file.
- John's `john.pot` or session files.
- Personal paths, usernames, or other identifying terminal details in screenshots.

Suggested repository structure:

```text
pdf-password-recovery-lab/
├── README.md
├── screenshots/
│   ├── 01-encryption-metadata.png
│   ├── 02-format-check.png
│   ├── 03-wordlist-attempt.png
│   ├── 04-rule-based-recovery-redacted.png
│   └── 05-password-validation-redacted.png
├── docs/
│   └── methodology.md
└── .gitignore
```

**Disclaimer:** This project is intended for educational purposes and authorized password recovery only.
