# SAST (Static Application Security Testing) SOP for Microsoft MFC / Visual C++ Projects

## NuvoISP – ISP Programming Tool

| Field            | Value                                   |
|------------------|-----------------------------------------|
| Document ID      | SOP-CRA-SAST-001                        |
| Version          | 2.0                                     |
| Date             | 2026-05-20                              |
| Product          | NuvoISP – ISP Programming Tool v4.18    |
| Product Type     | Windows MFC Desktop Application (C/C++) |
| Applicable Regulation | EU Cyber Resilience Act (CRA) – Regulation (EU) 2024/2847 |
| Author           | Engineering Team                        |
| Approved By      | *(To be filled)*                        |

---

## 1. Purpose

This SOP defines the standard procedure for running Static Application Security Testing (SAST) on Windows MFC / Visual C++ projects using **MSVC Code Analysis (`/analyze`)** as part of CRA DevSecOps requirements.

CRA Article 10 and Annex I require manufacturers to:
- Apply secure development practices, including static analysis
- Identify and remediate vulnerabilities before release
- Document the security testing performed

---

## 2. Tool: MSVC Code Analysis (`/analyze`)

MSVC `/analyze` is the recommended SAST tool for MFC projects. It is built into Visual Studio, understands MFC natively, and provides security-focused rulesets.

| Feature | Detail |
|---------|--------|
| Cost | Free (included with Visual Studio) |
| Language | C/C++ (full MFC support) |
| Integration | Visual Studio IDE + MSBuild CLI |
| Rulesets | Microsoft Native Recommended, Security Rules, All Rules |
| SAL Annotations | Supports `_In_`, `_Out_`, `_Ret_maybenull_` for deeper analysis |
| CRA relevance | **High** — detects buffer overflows, uninitialized memory, null dereference, resource leaks |

---

## 3. Procedure

### Step 1: Run MSVC Code Analysis

#### Option A: Visual Studio IDE

1. Open `NuvoISP_VS2019.sln` in Visual Studio 2022
2. Go to **Project → Properties → Code Analysis**
3. Set **Configuration**: `Release` | `Win32`
4. Set **Rule Set**: `Microsoft Native Recommended Rules` or `Microsoft All Rules`
5. Enable **Run Code Analysis on Build**: `Yes`
6. Build the project: **Build → Rebuild Solution**
7. Review warnings in the **Error List** window (filter by "Code Analysis")

#### Option B: MSBuild Command Line (CI/CD)

```powershell
# Navigate to solution directory
cd "D:\repo\NuvoISP"

# Run MSBuild with Code Analysis enabled
msbuild NuvoISP_VS2019.sln `
  /p:Configuration=Release `
  /p:Platform=Win32 `
  /p:RunCodeAnalysis=true `
  /p:CodeAnalysisRuleSet="NativeRecommendedRules.ruleset" `
  /p:EnablePREfast=true `
  /t:Rebuild 2>&1 | Tee-Object -FilePath "sast-msvc-results.log"

# Filter warnings
Select-String -Path "sast-msvc-results.log" -Pattern "warning C\d{4,5}:" |
  Select-Object -ExpandProperty Line |
  Set-Content "sast-msvc-warnings.txt"

Write-Host "MSVC Code Analysis warnings:"
(Get-Content "sast-msvc-warnings.txt").Count
```

### Key MSVC Warning Categories for CRA

| Warning Range | Category | CRA Relevance |
|--------------|----------|---------------|
| C6001–C6099 | Uninitialized memory | High |
| C6200–C6299 | Buffer overrun | **Critical** |
| C6300–C6399 | Format string | High |
| C6500–C6599 | Invalid annotation | Medium |
| C26400–C26499 | C++ Core Guidelines | Medium |
| C28xxx | SAL annotation warnings | High |

### Step 2: Review and Triage Findings

| Severity | Action Required | SLA |
|----------|----------------|-----|
| **Error / Critical** | Must fix before release | Immediate |
| **Warning** | Should fix; document if accept risk | Before release |
| **Style / Performance** | Fix if feasible; OK to defer | Next release |
| **Information** | Review only | No action required |

Create a findings summary:

```
SAST Report - NuvoISP v4.18
Date: YYYY-MM-DD

MSVC /analyze:
  Critical: X
  Warning:  Y
  Info:     Z

Unresolved (with justification):
  - [warning ID]: [justification why accepted]
```

### Step 3: Suppress with Justification

For accepted findings, suppress with documented reason:

```cpp
#pragma warning(suppress: 6001) // Reason: variable is initialized by MFC framework
```

### Step 4: Archive Results

Store SAST results alongside the SBOM deliverables:

```
release/
├── _manifest/spdx_2.2/manifest.spdx.json   ← SBOM
├── sbom-validation.json                      ← SBOM validation
├── third_party_notice.txt                    ← Third-party components
├── sast/
│   ├── sast-msvc-results.log                ← MSVC /analyze full log
│   ├── sast-msvc-warnings.txt               ← MSVC warnings summary
│   └── sast-summary.md                      ← Triage summary
```

---

## 4. When to Run SAST

| Trigger | Scope |
|---------|-------|
| Every release build | Full project scan |
| PR/code review (if CI available) | Changed files only |
| New third-party component added | Scan the new component |
| After security-related code changes | Affected modules |
| CRA compliance audit | Full project + archive results |

---

## 5. CRA SAST Checklist (Pre-Release Gate)

- [ ] MSVC `/analyze` run with `NativeRecommendedRules` ruleset
- [ ] All Critical/Error findings resolved
- [ ] All Warning findings resolved or documented with justification
- [ ] SAST results archived with release package
- [ ] Suppressed warnings have documented justification

---

## 6. Roles and Responsibilities

| Role | Responsibility |
|------|---------------|
| **Developer** | Run SAST during development, fix findings before PR |
| **Build Engineer** | Integrate MSVC `/analyze` into build pipeline |
| **Security/PSIRT** | Review Critical/High findings, approve suppressions |
| **Tech Lead** | Final approval of suppressed warnings with justification |
| **QA** | Verify SAST reports are archived with each release |

---

## 7. Reference Documents

| Document | Description |
|----------|-------------|
| [MSVC Code Analysis](https://learn.microsoft.com/en-us/cpp/code-quality/code-analysis-for-c-cpp-overview) | Microsoft C++ static analysis documentation |
| [MSVC /analyze Warning Reference](https://learn.microsoft.com/en-us/cpp/code-quality/code-analysis-for-c-cpp-warnings) | All Code Analysis warning codes |
| [C++ Core Guidelines Checkers](https://learn.microsoft.com/en-us/cpp/code-quality/using-the-cpp-core-guidelines-checkers) | Modern C++ rule checkers |
| [EU Cyber Resilience Act](https://eur-lex.europa.eu/eli/reg/2024/2847) | CRA regulation |

