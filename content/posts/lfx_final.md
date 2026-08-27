---
title: "LFX Mentorship 2026: Final Report on Magma Core"
date: 2026-08-25
author: "Md Hammaduddin"
tags: ["LFX", "Linux Foundation", "Magma Core", "5G", "Open Source", "TypeScript", "C++"]
description: "A summary of my journey, technical contributions, bug fixes, and learnings as a Linux Foundation LFX Mentee on the Magma Core project."
type: "post"
---

# Reliability and Maintainability in Magma Core

LFX Mentorship Report

Md Hammaduddin  
Mentee, Linux Foundation Mentorship Program 2026  
GitHub: [Hammaduddin561](https://github.com/Hammaduddin561) | Email: hammaduddin57@gmail.com  
Project: [Magma Core](https://github.com/magma/magma) | Mentors: [Micky Kumar](https://github.com/mickymkumar) and [Lucas Amaral](https://github.com/lucaaamaral)

---

## 1. Introduction and Background

Magma is an open source software platform, hosted by the Linux Foundation, for building and operating mobile networks. It supports LTE, 5G Standalone and WiFi access, and it interoperates with existing operator infrastructure through standard 3GPP interfaces.

My contributions during the mentorship fall in two components:
1. **The 5G Standalone signaling path in the Access Gateway (AGW):** C/C++ backend handling NGAP protocol decoding and AMF application logic.
2. **The state layer of the Network Management System (NMS):** TypeScript and React frontend handling subscriber and gateway context caches.

---

## 2. Goals and Objectives

* Investigate, reproduce, and fix protocol failure reporting bugs in the 5G AMF and NGAP stack.
* Consolidate duplicated asynchronous gRPC response handlers in the 5G mobility client.
* Audit and resolve state corruption defects in the NMS React context providers.
* Follow a red-to-green testing protocol: accompany every change with automated Bazel and Jest regression tests.

---

## 3. Overview of Contributions

Across the mentorship period, I submitted 5 pull requests to magma/magma, adding 468 lines and removing 247 across 14 files with zero CI failures across 93 check runs.

| Pull Request | Stack | Title | Status |
| :--- | :--- | :--- | :--- |
| **[PR #15979](https://github.com/magma/magma/pull/15979)** | C and C++ | `fix(amf): return slice-not-supported on S-NSSAI mismatch` | In Review |
| **[PR #16009](https://github.com/magma/magma/pull/16009)** | C++ | `refactor(amf): consolidate IP allocation response logic` | In Review |
| **[PR #16035](https://github.com/magma/magma/pull/16035)** | TypeScript | `fix(nms): spread bulk-added subscribers into subscriber map` | In Review |
| **[PR #16038](https://github.com/magma/magma/pull/16038)** | TypeScript | `fix(nms): keep read-only gateway fields when caching a gateway edit` | In Review |
| **[PR #16039](https://github.com/magma/magma/pull/16039)** | TypeScript | `fix(nms): keep read-only federation gateway fields when caching edit` | In Review |

---

## 4. Technical Details

### 4.1 Reporting an Unsupported Network Slice Correctly (PR #15979)

* **Problem:** When a gNodeB requested a network slice unsupported by the AMF configuration, Magma returned an `unknown-PLMN` error instead of `slice-not-supported`. This misdirected operators to check PLMN and tracking area configuration rather than slice configuration.
* **Solution:**
  - Introduced distinct return code `TA_LIST_UNKNOWN_SLICE` in `ngap_amf_ta.h`.
  - Threaded the slice mismatch state through `ngap_amf_ta.c` comparison levels.
  - Mapped the condition to `NGAP_CAUSE_RADIO_NETWORK_SLICE_NOT_SUPPORTED` in `ngap_amf_handlers.c`.
* **Verification:** Added test case `test_ngap_setup_request_slice_mismatch` in `test_ngap_flows.cpp`. All 27 tests passed in Bazel:
  ```text
  [==========] 27 tests from 1 test suite ran.
  [  PASSED  ] 27 tests.
  ```

---

### 4.2 Consolidating Three Copies of the Same Handler (PR #16009)

* **Problem:** `M5GMobilityServiceClient.cpp` contained three address allocation status handlers (IPv4, IPv6, and dual-stack) that repeated identical APN validation and ITTI message allocation.
* **Solution:**
  - Consolidated common logic into a single `send_ip_allocation_response` helper.
  - Corrected memory release using `itti_free_msg_content` followed by `free`.
  - Removed undefined symbol references.
* **Verification:** Executed NGAP flow suite and AMF procedures suite locally: 62 test cases passed across two test binaries.

---

### 4.3 Bulk Subscriber Add State Corruption in NMS (PR #16035)

* **Problem:** In `SubscriberContext.tsx`, missing the spread operator caused bulk subscriber additions to store the new subscriber batch under a literal object key `"newSubscriberMap"`, corrupting subscriber tables and gateway associations.
* **Solution:**
  - Fixed cache update to `setSubscriberMap({...subscriberMap, ...newSubscriberMap})`.
  - Removed `@ts-ignore` and completed the legacy migration comment.
* **Verification:** Added Jest regression test in `SubscriberAddEditTest.tsx`. Verified the test failed before the fix and passed after:
  ```text
  PASS app/views/subscriber/__tests__/SubscriberAddEditTest.tsx
  ✓ given bulk-added subscribers when added then map is keyed by IMSI
  ```

---

### 4.4 Preserving Read-Only Gateway Fields on Save (PR #16038 and PR #16039)

* **Problem:** Saving an edited gateway stripped read-only fields such as `status` and `checked_in_recently` due to an unsound type cast to `MutableLteGateway`, causing healthy gateways to display as unhealthy after editing until a full reload.
* **Solution:** Replaced type cast with explicit property merging (`{...lteGateways[key], ...value}`) across `GatewayContext.tsx` and `FEGGatewayContext.tsx`.
* **Verification:** Added regression tests in `GatewayConfigTest.tsx` and `FEGGatewayDetailConfigTest.tsx`. All tests passed.

---

## 5. Verification and Continuous Integration

Across all five pull requests, the upstream CI system executed 93 check runs with 64 successes, 28 skipped (non-applicable file types), 1 cancelled, and 0 failures.

| Pull Request | Checks | Success | Skipped | Cancelled | Failed |
| :--- | :--- | :--- | :--- | :--- | :--- |
| #15979 | 16 | 11 | 5 | 0 | 0 |
| #16009 | 18 | 12 | 6 | 0 | 0 |
| #16035 | 21 | 15 | 5 | 1 | 0 |
| #16038 | 19 | 13 | 6 | 0 | 0 |
| #16039 | 19 | 13 | 6 | 0 | 0 |
| **Total** | **93** | **64** | **28** | **1** | **0** |

All PRs passed SonarCloud quality gates, ESLint, cpplint, and GitGuardian secret scans.

---

## 6. Reflections and Learnings

1. **Suppressed errors are leads:** Treating `@ts-ignore` and migration comments as indicators of latent bugs helped locate subtle state corruption issues.
2. **Symptoms before causes:** Structuring pull request descriptions around operator-visible symptoms made code reviews clearer and more actionable.
3. **Scope discipline:** Keeping independent concerns in separate pull requests improved review velocity and maintainability.
4. **Evidence-based verification:** Adopting a red-to-green testing protocol provided clear proof of correctness for reviewers.

---

## 7. Acknowledgments

I am grateful to my mentors Micky Kumar and Lucas Amaral, as well as Jordan Vrtanoski, for their technical guidance, code reviews, and feedback throughout the mentorship.

Thank you to the Linux Foundation and the LFX Mentorship program for the opportunity to contribute to Magma Core.

---

## 8. Links and References

* GitHub Profile: [@Hammaduddin561](https://github.com/Hammaduddin561)
* Magma Repository: [magma/magma](https://github.com/magma/magma)
* Pull Requests:
  * [#15979 - Fix AMF S-NSSAI Slice Mismatch](https://github.com/magma/magma/pull/15979)
  * [#16009 - Consolidate IP Allocation Response Logic](https://github.com/magma/magma/pull/16009)
  * [#16035 - Fix NMS Subscriber Map Spread](https://github.com/magma/magma/pull/16035)
  * [#16038 - Preserve NMS Gateway Read-Only Fields](https://github.com/magma/magma/pull/16038)
  * [#16039 - Preserve NMS FeG Gateway Read-Only Fields](https://github.com/magma/magma/pull/16039)
