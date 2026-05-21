# SAST Scan Summary — NuvoISP v4.18

| Field | Value |
|-------|-------|
| **Date** | 2026-05-21 |
| **Tool** | MSVC `/analyze` (cl.exe 14.44.35207, MSVC v143) |
| **Ruleset** | NativeRecommendedRules.ruleset |
| **Configuration** | Release \| Win32 |
| **Build Result** | **0 Errors, 184 Warnings** |
| **Build Time** | 01:59 |
| **Files Scanned** | 62 source files (.cpp / .c / .h) |

---

## Warning Summary by Code

| Code | Count | Severity | Description |
|------|------:|----------|-------------|
| **C26495** | 292 | Medium | Variable is not initialized (type.6) |
| **C5208** | 18 | Low | Unnamed class in typedef — non-standard extension |
| **C28159** | 16 | Medium | Use `GetTickCount64` instead of `GetTickCount` (49-day overflow) |
| **C6284** | 10 | Medium | Object passed as parameter when string is required in format call |
| **C6387** | 10 | **High** | Possible NULL value passed to function that expects non-null |
| **C6011** | 4 | **High** | Dereferencing NULL pointer |
| **C6031** | 4 | Medium | Return value ignored (`freopen`) |
| **C6385** | 4 | **High** | Buffer overrun — reading past buffer bounds |
| **C28251** | 2 | Low | Inconsistent annotation for `NTSTATUS` |
| **C6001** | 2 | **High** | Using uninitialized memory |
| **C6302** | 2 | Medium | Format string mismatch |
| **Total** | **364** | | (184 unique, duplicated across PCH/analysis passes) |

> **Note:** 184 warnings reported by MSBuild; 364 matching lines in the log due to
> precompiled-header and multi-pass analysis duplication. Unique warning count is **184**.

---

## High-Severity Findings (C6xxx — Require Review)

### C6011 — NULL Pointer Dereference
| File | Line | Detail |
|------|------|--------|
| hid.c | 149 | NULL pointer `dev` dereferenced |
| hid.c | 327 | NULL pointer `device_interface_detail_data` dereferenced |

### C6001 — Uninitialized Memory Use
| File | Line | Detail |
|------|------|--------|
| DialogMain.cpp | 1049 | Using uninitialized variable `a7` |

### C6385 — Buffer Overrun (Read)
| File | Line | Detail |
|------|------|--------|
| DialogChipSetting_M251.cpp | 107 | Reading past buffer bounds |
| DialogChipSetting_M251.cpp | 117 | Reading past buffer bounds |

### C6387 — Possible NULL Passed to Non-null Parameter
| File | Line | Detail |
|------|------|--------|
| CUartIO.cpp | 94 | `o.hEvent` may be 0 → `ResetEvent` |
| CUartIO.cpp | 96 | `o.hEvent` may be 0 → `WaitForSingleObject` |
| CUartIO.cpp | 109 | `o.hEvent` may be 0 → `CloseHandle` |
| CUartIO.cpp | 138 | `overlapped.hEvent` may be 0 → `CloseHandle` |
| hid.c | 633 | `buf` may be 0 → `memcpy` |

### C6302 — Format String Mismatch
| File | Line | Detail |
|------|------|--------|
| DialogMain.cpp | 1073 | Format string type mismatch |

### C6284 — Object Passed Where String Expected
| File | Line | Detail |
|------|------|--------|
| About.cpp | 45 | CString passed to format expecting string |
| DlgNuvoISP.cpp | 298 | CString passed to format expecting string |
| DlgNuvoISP.cpp | 1058 | CString passed to format expecting string |
| DlgNuvoISP.cpp | 1148 | CString passed to format expecting string |

### C6031 — Return Value Ignored
| File | Line | Detail |
|------|------|--------|
| NuvoISPTool.cpp | 177 | `freopen` return value ignored |
| NuvoISPTool.cpp | 178 | `freopen` return value ignored |

---

## Medium-Severity Findings

### C26495 — Uninitialized Member Variables (292 occurrences)
Widespread across Dialog classes. Member variables not initialized in constructors.
Top affected files: all `DialogChipSetting_*.cpp`, `DialogConfiguration_*.cpp`,
`ISPLdCMD.cpp`, `ISPProc.cpp`, `DialogMain.cpp`, `DlgNuvoISP.cpp`.

### C28159 — GetTickCount → GetTickCount64 (16 occurrences)
| File | Lines |
|------|-------|
| CTRSP.cpp | 322, 324 |
| ISPProc.cpp | 157, 166, 201, 205 |
| ISPLdCMD.cpp | 843, 860 |

---

## Recommendations

1. **Priority 1 (High):** Fix C6011, C6001, C6385, C6387 — potential crashes / undefined behavior.
2. **Priority 2 (Medium):** Fix C6284, C6302 — format string issues could produce wrong output.
3. **Priority 3 (Medium):** Replace `GetTickCount` with `GetTickCount64` (C28159).
4. **Priority 4 (Low):** Initialize member variables in constructors (C26495) — best practice.
5. **Priority 5 (Low):** Fix `freopen` return value checks (C6031).
6. **No action required:** C5208, C28251 — third-party / SDK headers.

---

## Files

| Artifact | Path |
|----------|------|
| Full build log | `SAST/sast-msvc-results.log` |
| Filtered warnings | `SAST/sast-msvc-warnings.txt` |
| This summary | `SAST/sast-summary.md` |
| SOP document | `SAST/SOP-CRA-SAST.md` |
