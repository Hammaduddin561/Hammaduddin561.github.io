---
title: "LFX Mentorship 2026: Final Report — Reliability & Maintainability in Magma Core"
date: 2026-08-25
author: "Md Hammaduddin"
tags: ["LFX", "Linux Foundation", "Magma Core", "5G", "Open Source", "TypeScript", "C++"]
description: "A summary of my journey, technical contributions, bug fixes, and learnings as a Linux Foundation LFX Mentee on the Magma Core project."
---

# LFX Mentorship 2026: Final Report — Magma Core

* **Mentee:** [Md Hammaduddin](https://github.com/Hammaduddin561) (`Hammaduddin561`)
* **Project:** [Magma Core](https://github.com/magma/magma) (The Linux Foundation)
* **Program:** [LFX Mentorship — Summer 2026](https://lfx.linuxfoundation.org/)
* **Mentors:** [Micky Kumar](https://github.com/mickymkumar), [Lucas Amaral](https://github.com/lucaaamaral)

---

## 📖 Introduction & Background

[Magma](https://magmacore.org/) is an open-source software platform hosted by **The Linux Foundation** that provides an open, flexible, and extendable mobile packet core solution for LTE, 5G Standalone (SA), and WiFi access networks.

My mentorship focused on improving the **reliability, diagnostic reporting, and maintainability** of the core network platform across two critical layers:
1. **The 5G Standalone (SA) Access Gateway (AGW):** Written in C/C++, handling 3GPP NGAP signaling, AMF registration, and gRPC mobility clients.
2. **The Network Management System (NMS):** Built with TypeScript and React, managing subscriber contexts, gateway topologies, and configuration state.

---

## 🎯 Goals & Objectives

* Investigate, reproduce, and fix critical protocol failure reporting bugs in the 5G AMF/NGAP stack.
* Consolidate duplicated asynchronous gRPC response handlers in the mobility client.
* Audit and resolve state corruption bugs hidden behind `@ts-ignore` and legacy migration comments in the NMS React frontend.
* Follow rigorous **red-to-green** testing protocols: accompany every bug fix with automated Bazel/Jest regression unit tests.

---

## 🚀 Overview of Contributions

Across the mentorship period, I submitted **5 targeted Pull Requests** to [`magma/magma`](https://github.com/magma/magma), adding 468 lines and removing 247 lines across 14 files with **zero CI failures across 93 continuous integration check runs**.

| PR # | Stack | Description | Status |
| :--- | :--- | :--- | :--- |
| **[#15979](https://github.com/magma/magma/pull/15979)** | C / C++ | `fix(amf): return slice-not-supported on S-NSSAI mismatch` | In Review |
| **[#16009](https://github.com/magma/magma/pull/16009)** | C++ | `refactor(amf): consolidate IP allocation response logic` | In Review |
| **[#16035](https://github.com/magma/magma/pull/16035)** | TypeScript / React | `fix(nms): spread bulk-added subscribers into subscriber map` | In Review |
| **[#16038](https://github.com/magma/magma/pull/16038)** | TypeScript / React | `fix(nms): keep read-only gateway fields when caching a gateway edit` | In Review |
| **[#16039](https://github.com/magma/magma/pull/16039)** | TypeScript / React | `fix(nms): keep read-only federation gateway fields when caching edit` | In Review |

---

## 🛠️ Technical Deep-Dives

### 1. Fix 5G S-NSSAI Slice Mismatch Diagnostic ([PR #15979](https://github.com/magma/magma/pull/15979))

* **The Problem:** When a gNodeB attempted to connect to the AMF with an unsupported S-NSSAI network slice, the AMF collapsed all non-matching conditions and reported a generic `unknown-PLMN` error instead of returning `slice-not-supported`. This misdirected network operators to examine PLMN configs instead of slice parameters.
* **The Solution:** 
  - Introduced `TA_LIST_UNKNOWN_SLICE` return code in `ngap_amf_ta.h`.
  - Threaded the comparison logic in `ngap_amf_ta.c` to distinguish between PLMN mismatch and S-NSSAI mismatch.
  - Mapped the condition to standard 3GPP cause `NGAP_CAUSE_RADIO_NETWORK_SLICE_NOT_SUPPORTED` in `ngap_amf_handlers.c`.
* **Testing:** Added unit test `test_ngap_setup_request_slice_mismatch` in `test_ngap_flows.cpp`. Verified all 27 unit tests passed locally via Bazel:
  ```bash
  bazel test //lte/gateway/c/core/oai/test/ngap:ngap_flows_test
  # Result: 27/27 PASSED
  ```

---

### 2. Consolidating 5G AMF IP Allocation Handlers ([PR #16009](https://github.com/magma/magma/pull/16009))

* **The Problem:** In `M5GMobilityServiceClient.cpp`, three separate address allocation status handlers (IPv4, IPv6, Dual-Stack) duplicated ~50 lines of APN validation, memory management, and ITTI message dispatch logic.
* **The Solution:**
  - Consolidated common logic into a unified `send_ip_allocation_response` helper function.
  - Fixed memory deallocation safety on rejection paths with proper `itti_free_msg_content`.
  - Replaced undefined macro calls with standard includes.
* **Testing:** Ran the full NGAP flow and AMF procedures test suites; 62/62 test cases passed with zero regressions.

---

### 3. NMS Bulk Subscriber State Corruption ([PR #16035](https://github.com/magma/magma/pull/16035))

* **The Problem:** In `nms/app/context/SubscriberContext.tsx`, bulk adding subscribers executed:
  ```ts
  // TODO[TS-migration] Should newSubscriberMap be spread here?
  // @ts-ignore
  setSubscriberMap({...subscriberMap, newSubscriberMap});
  ```
  Missing the spread operator (`...`) caused JavaScript to store the entire batch under a literal key `"newSubscriberMap"`, corrupting subscriber tables and gateway mappings.
* **The Solution:**
  - Changed to `setSubscriberMap({...subscriberMap, ...newSubscriberMap})`.
  - Deleted the `@ts-ignore` and completed the unresolved maintainer TODO.
* **Testing:** Added Jest regression test in `SubscriberAddEditTest.tsx`. Reverting the fix demonstrably failed the test, while applying the fix passed all 3 test suites:
  ```bash
  PASS app/views/subscriber/__tests__/SubscriberAddEditTest.tsx
  ✓ given bulk-added subscribers when added then map is keyed by IMSI
  ```

---

### 4. Preserving Read-Only Gateway Fields in NMS ([PR #16038](https://github.com/magma/magma/pull/16038) & [#16039](https://github.com/magma/magma/pull/16039))

* **The Problem:** When editing a gateway in NMS, saving stripped read-only fields (`status`, `checked_in_recently`) because of an unsound type cast to `MutableLteGateway`, causing healthy gateways to immediately display as "unhealthy" until a full page reload.
* **The Solution:** Implemented explicit property merging (`{...lteGateways[key], ...value}`) in both `GatewayContext.tsx` and `FEGGatewayContext.tsx`.
* **Testing:** Added regression tests in `GatewayConfigTest.tsx` and `FEGGatewayDetailConfigTest.tsx`, ensuring gateway health state survives configuration saves.

---

## 📊 Continuous Integration Summary

Across all 5 pull requests, the automated CI pipelines executed **93 check runs**:
* **Succeeded:** 64
* **Skipped (irrelevant file types):** 28
* **Cancelled (superseded title check):** 1
* **Failed:** **0**

Passed all static analyzers and quality gates: SonarCloud, ESLint, cpplint, GitGuardian, shellcheck, markdownlint, and yamllint.

---

## 💡 Key Learnings & Takeaways

1. **Suppressed Errors are Leads:** `// @ts-ignore` and `// TODO` comments are often pointers to real, latent bugs left behind during automated migrations.
2. **Prioritize the Symptom Over the Mechanism:** Opening a PR with an operator-visible bug report makes code reviews significantly clearer than arguing about style.
3. **Scope Discipline:** Keep independent concerns in separate PRs. Splitting PR #15979 into #15979 and #16009 on mentor request resulted in much cleaner, easily reviewable commits.
4. **Red-to-Green Testing:** Always verify that a test fails on unfixed code and passes cleanly after the fix.

---

## 🤝 Acknowledgments

I would like to express my deepest gratitude to my mentors [Micky Kumar](https://github.com/mickymkumar) and [Lucas Amaral](https://github.com/lucaaamaral), as well as [Jordan Vrtanoski](https://github.com/jordanvrtanoski), for their invaluable guidance, patient code reviews, and technical feedback throughout this journey.

Thank you to **The Linux Foundation** and the **LFX Mentorship Program** for providing this incredible opportunity to contribute to open-source telecommunication infrastructure!

---

## 🔗 Useful Links & References

* **GitHub Profile:** [@Hammaduddin561](https://github.com/Hammaduddin561)
* **Magma Repository:** [magma/magma](https://github.com/magma/magma)
* **All PRs:**
  * [#15979 — S-NSSAI Slice Mismatch](https://github.com/magma/magma/pull/15979)
  * [#16009 — IP Allocation Consolidation](https://github.com/magma/magma/pull/16009)
  * [#16035 — NMS Subscriber Map Spread](https://github.com/magma/magma/pull/16035)
  * [#16038 — NMS Gateway Read-Only Fields](https://github.com/magma/magma/pull/16038)
  * [#16039 — NMS FeG Gateway Read-Only Fields](https://github.com/magma/magma/pull/16039)
