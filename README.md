# Week 2: Policy-as-Code for Terraform Compliance

## Project Overview

In Week 1, I built a compliant AWS S3 bucket using Terraform. In Week 2, I shifted from building controls to validating them automatically using Policy-as-Code.

The objective of this project was to use Open Policy Agent (OPA) and Rego to evaluate Terraform plans against NIST 800-53 compliance requirements. Instead of manually reviewing infrastructure and claiming it is compliant, these policies provide an automated pass/fail decision based on defined controls.

## Controls Validated

| Control | Requirement | Policy File |
|----------|-------------|-------------|
| SC-28 | Deny S3 buckets without encryption at rest | `sc28_encryption_aws.rego` |
| AC-3 | Deny S3 buckets missing required public access restrictions | `ac3_no_public_aws.rego` |
| CM-6 | Deny resources missing required compliance tags | `cm6_required_tags_aws.rego` |

---

## Tools Used

- Terraform
- AWS S3
- Open Policy Agent (OPA)
- Rego
- PowerShell
- GitHub

---

# Environment Setup & Troubleshooting

## Step 1: Installing Open Policy Agent (OPA)

OPA is the engine responsible for evaluating Rego policies and executing compliance tests.

Initially, OPA was not installed on my system. I went to the OPA website to download the correct executables

[https://github.com/open-policy-agent/opa/releases?utm_source=chatgpt.com](https://github.com/open-policy-agent/opa/releases/latest)

Downloaded binary:

```text
opa_windows_amd64.exe
```

To enable OPA to be executed globally from PowerShell, I added the directory containing opa.exe (C:\Users\XXX\tools) to my Windows User PATH environment variable. This allows Windows to locate and execute OPA from any working directory without requiring the full file path. Verifying and troubleshooting PATH configuration is a common task when installing developer and security tooling.


<img width="875" height="824" alt="image" src="https://github.com/user-attachments/assets/9c480952-e448-48f1-a51f-dcc3c45e2375" />

---

## Step 2: Downloading and Configuring OPA

Renamed:

```text
opa.exe
```

Stored in:

```text
C:\Users\XXXX\tools
```

---

## Step 3: Verifying Installation

After updating the PATH environment variable and restarting PowerShell:

### Command

```powershell
opa
```

### Result

```text
Usage:
opa [command]
```

### Screenshot to Validate OPA Installation

<img width="478" height="358" alt="image" src="https://github.com/user-attachments/assets/00a867b4-8f24-4116-aeb3-fdcce07d0117" />


This confirmed:

- OPA installed successfully
- PATH configured correctly
- Rego policies could be executed locally

---

# Step 4: Reviewing the Starter Files

The challenge included three policy files and three corresponding test files.

```text
ac3_no_public_aws.rego
ac3_no_public_aws_test.rego

cm6_required_tags_aws.rego
cm6_required_tags_aws_test.rego

sc28_encryption_aws.rego
sc28_encryption_aws_test.rego
```

The test files acted as the specification.

Rather than guessing how Terraform plan data was structured, I used the test fixtures to understand:

- Resource types
- Resource references
- Planned values
- Expected pass/fail behavior

---

# Step 5: Building the SC-28 Encryption Policy

## Objective

SC-28 requires encryption at rest.

The policy must deny any S3 bucket that does not have a matching server-side encryption configuration.

### Key Clue From the Test File

The test fixture showed that bucket names may not be available during planning.

Instead of comparing bucket names, Terraform stores references such as:

```text
aws_s3_bucket.primary.id
```

This meant resources needed to be matched by reference rather than value.

### Policy Logic

```rego
deny contains msg if {
    bucket := input.configuration.root_module.resources[_]

    bucket.type == "aws_s3_bucket"

    not encrypted_bucket(bucket)

    msg := sprintf(
        "SC-28 violation: bucket '%s' has no server-side encryption configuration.",
        [bucket.name]
    )
}
```

### Why This Change Was Made

The starter code contained:

```rego
false
msg := "todo"
```

Replacing:

```rego
false
```

with:

```rego
not encrypted_bucket(bucket)
```

converted the policy from a placeholder into a real compliance control.

### Debugging

OPA initially returned:

```text
rego_compile_error: var ref declared above
```

Root Cause:

```rego
some ref

ref := ...
```

Resolution:

Removed the duplicate declaration and allowed Rego to infer the variable assignment.

### Screenshot

![SC28 Debugging](./screenshots/04-sc28-debugging.png)

---

# Step 6: Building the AC-3 Public Access Policy

## Objective

AC-3 requires access enforcement.

The policy must deny any bucket missing a valid public access block.

Required settings:

```text
block_public_acls
block_public_policy
ignore_public_acls
restrict_public_buckets
```

### Key Clue From the Test File

The comments inside the test fixture indicated:

```text
configuration.root_module.resources
```

contains resource relationships.

However:

```text
planned_values.root_module.resources
```

contains the actual values.

### Policy Logic

The policy:

1. Matches the bucket to its public access block.
2. Reads the actual boolean values from planned values.
3. Verifies all four settings equal `true`.

### Debugging

The first implementation incorrectly attempted to validate:

```rego
pab.expressions.block_public_acls.constant_value
```

The tests revealed those values were stored under:

```rego
planned.values.block_public_acls
```

After updating the policy, the AC-3 tests passed.

### Screenshot

![AC3 Debugging](./screenshots/05-ac3-debugging.png)

---

# Step 7: Building the CM-6 Required Tags Policy

## Objective

CM-6 requires consistent configuration settings.

The policy validates that resources contain required compliance tags.

Required tags:

```text
Project
Environment
ManagedBy
ComplianceScope
```

### Key Clue From the Test File

The test fixture showed tags were stored under:

```rego
values.tags_all
```

### Policy Logic

The policy:

1. Retrieves resource tags.
2. Compares them against the required tag set.
3. Calculates missing tags.
4. Generates a denial message when required tags are absent.

### Example Denial Message

```text
CM-6 violation: resource 'aws_s3_bucket.primary' is missing required tags.
```

### Screenshot

![CM6 Debugging](./screenshots/06-cm6-debugging.png)

---

# Step 8: Running the Test Suite

Once all three policies were implemented:

### Command

```powershell
opa test .
```

### Initial Results

```text
PASS: 4/6
FAIL: 2/6
```

### Screenshot

![Initial Test Results](./screenshots/07-pass-4-of-6.png)

After correcting the AC-3 and CM-6 policies:

### Final Result

```text
PASS: 6/6
```

### Screenshot

![All Tests Passing](./screenshots/08-pass-6-of-6.png)

---

# Final Architecture

```text
Terraform Plan JSON
         │
         ▼
      OPA Engine
         │
         ▼
   Rego Policies
         │
 ┌───────┼────────┐
 ▼       ▼        ▼

SC-28   AC-3    CM-6

 ▼       ▼        ▼

 PASS / FAIL
         │
         ▼
Compliance Verdict
```

---

# Key Lessons Learned

## Match by Reference, Not Name

Terraform plans may contain unknown values during planning.

Instead of comparing resource names directly, policies matched resources using Terraform references:

```text
aws_s3_bucket.primary.id
```

This creates more reliable compliance validation because references remain available even when actual resource names are unknown.

---

## Why This Matters for GRC Engineering

Traditional GRC often relies on:

- Manual reviews
- Screenshots
- Spreadsheets
- Human interpretation

Policy-as-Code transforms compliance into executable logic:

```text
Terraform Plan
        ↓
OPA/Rego Policy
        ↓
Automated Verdict
        ↓
Repeatable Evidence
```

This project demonstrates how compliance requirements can be expressed, tested, and enforced programmatically using infrastructure-as-code and policy-as-code principles.

---

## Final Validation

```powershell
opa test .
```

Result:

```text
PASS: 6/6
```

All compliance policies successfully validated both compliant and non-compliant Terraform plans.
