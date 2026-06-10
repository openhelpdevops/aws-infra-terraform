# Terraform Installation Guide for Windows 10

## Overview

This guide explains how to install Terraform on Windows 10 using the official HashiCorp binary.

---

## Step 1: Download Terraform

Official download page:

https://developer.hashicorp.com/terraform/downloads

Download:

- Windows AMD64 (64-bit) ZIP

Example file:

terraform_1.x.x_windows_amd64.zip

---

## Step 2: Extract the ZIP File

1. Right-click the downloaded ZIP file.
2. Click **Extract All**.
3. Extract to:

```text
C:\Terraform
```

Result:

```text
C:\Terraform
└── terraform.exe
```


## Step 3: Add Terraform to PATH

1. Press Windows Key.
2. Search:

```text
Environment Variables
```

3. Open:

```text
Edit the system environment variables
```

4. Click:

```text
Environment Variables
```

5. Under **System Variables**, select:

```text
Path
```

6. Click:

```text
Edit
```

7. Click:

```text
New
```

8. Add:

```text
C:\Terraform
```

9. Click OK on all windows.

---

## Screenshot Placeholder

```text
[Screenshot 4 - Environment Variables]

[Screenshot 5 - PATH Variable]

[Screenshot 6 - C:\Terraform Added to PATH]
```

---

## Step 4: Verify Installation

Open PowerShell:

```powershell
terraform -version
```

Expected Output:

```text
Terraform v1.x.x
on windows_amd64
```

Check executable location:

```powershell
where terraform
```

Example:

```text
C:\Terraform\terraform.exe
```

---

## Troubleshooting

### terraform is not recognized

Close and reopen PowerShell and run:

```powershell
where terraform
```

Verify PATH contains:

```text
C:\Terraform
```

---

## Alternative Installation Using Winget

```powershell
winget install Hashicorp.Terraform
```

Verify:

```powershell
terraform -version
```

---

## Alternative Installation Using Chocolatey

```powershell
choco install terraform -y
```

Verify:

```powershell
terraform -version
```
