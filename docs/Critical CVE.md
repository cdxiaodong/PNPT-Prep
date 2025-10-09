---
title: "CVE-2025-49844 — Redis Lua Use-After-Free (RCE)"
description: "Critical vulnerability in Redis Lua scripting engine (CVSS 10.0) allowing remote code execution."
tags:
  - Redis
  - CVE
  - Security Advisory
  - RCE
  - Lua
  - Azure Redis
  - Redis Enterprise
  - Lettuce Redis
published: 2025-01-15
severity: Critical
cvss_score: 10.0
references:
  - https://nvd.nist.gov/vuln/detail/CVE-2025-49844
  - https://github.com/redis/redis/releases
  - https://github.com/redis-stack/redis-stack/releases
  - https://docs.redis.com/latest/rs/release-notes/
  - https://learn.microsoft.com/en-us/azure/azure-cache-for-redis/cache-whats-new
---

# 🧱 CVE-2025-49844 — Redis Lua Use-After-Free (RCE)

!!!danger 
    "Critical RCE Vulnerability"

    A **Use-After-Free vulnerability** was discovered in the **Redis Lua scripting engine**, allowing potential **remote code execution (RCE)**.
    This impacts all Redis releases that include Lua scripting functionality.

    **CVSS:** 10.0 (Critical)  
    **CVE:** [CVE-2025-49844](https://nvd.nist.gov/vuln/detail/CVE-2025-49844)  
    **Component:** Redis Lua scripting engine  
    **Disclosure Date:** January 2025  
    **Upstream Fix:** [redis/redis on GitHub](https://github.com/redis/redis)


## 🚨 Impacted Releases

| Edition | Impacted Versions |
|----------|------------------|
| **Redis Software (Enterprise)** | All releases prior to fixed builds |
| **Redis OSS / CE / Stack** | All versions with Lua scripting enabled |

---

## ✅ Fixed Releases

| Edition | Fixed Version(s) |
|----------|------------------|
| **Redis Enterprise / Software** | 7.22.2-12 and above<br>7.8.6-207 and above<br>7.4.6-272 and above<br>7.2.4-138 and above<br>6.4.2-131 and above |
| **Redis OSS / CE / Stack** | OSS/CE: 8.2.2+, 8.0.4+, 7.4.6+, 7.2.11+<br>Stack: 7.4.0-v7+, 7.2.0-v19+ |

**Reference:**  
- [Redis OSS Releases](https://github.com/redis/redis/releases)  
- [Redis Stack Releases](https://github.com/redis-stack/redis-stack/releases)  
- [Redis Enterprise Release Notes](https://docs.redis.com/latest/rs/release-notes/)

---

## 🧮 Versioning Explained

!!! note "Redis Versioning Scheme"
    Redis OSS uses **semantic versioning**:
    ```
    Major.Minor.Patch
    Example: 7.2.11 → Major 7, Minor 2, Patch 11
    ```
    Each minor line (e.g., `7.2.x`, `7.4.x`, `8.0.x`) receives **backported patches** independently.
    The phrase **“and above”** means *all subsequent patch versions within the same maintenance branch* —  
    not necessarily all future major releases.
    
    Each Redis branch is a maintenance line, such as 7.22.x, 7.23.x, 7.8.x, etc.

    When Redis says a fix applies to “7.22.2-12 and above,” it means:

    ✅ All newer builds and patches within the 7.22 branch (for example 7.22.2-13, 7.22.3-100, etc.) will include that fix.

    🟨 Newer Enterprise branches (like 7.23.x) will probably have it — but you must verify in the release notes, because each branch is developed and tested separately.

    ❌ Future major versions (like 8.x.x) are not automatically guaranteed to include that same fix unless Redis confirms it in their changelog.

!!! tip "Enterprise Versioning (Compound Scheme)"
    Redis Enterprise (Software) versions use:
    ```
    Major.Minor.Patch-Build
    Example: 7.22.2-12
    ```
    - `7.22.2` → Enterprise base version  
    - `-12` → internal build iteration or hotfix  
    - These internal builds reflect Redis Ltd.’s commercial patch cycles.

---

## ☁️ Integrated and Managed Versions

| Integration | Description | Affected? | Notes |
|--------------|--------------|-----------|-------|
| **Azure Redis / Azure Redis Database** | Microsoft-managed Redis service (based on OSS Redis) | 🟨 Potentially, vendor-managed | Microsoft backports critical fixes automatically. See [Azure Cache for Redis – What’s New](https://learn.microsoft.com/en-us/azure/azure-cache-for-redis/cache-whats-new). |
| **Lettuce Redis Client** | Java client library for Redis | ❌ Should Not be affected | [Client-side only](https://redis.io/docs/latest/develop/clients/lettuce/?utm_source=chatgpt.com), no embedded Lua execution. |
| **Redis Insight** | Redis management GUI | ❌ Should Not be affected  | Does not run Redis internally. |
| **Redis on Windows / Redis OSS** | Self-hosted Redis binaries | ✅ Affected if below fixed versions | Upgrade to latest patched release from [Redis GitHub](https://github.com/redis/redis/releases). |

---

## 🧾 Summary

- **Affected:** All Redis servers (OSS/Enterprise/Stack) with Lua scripting enabled  
- **Not affected:** Client libraries (e.g., Lettuce, Jedis), management tools (Redis Insight)  
- **Remediation:**  
  - Upgrade Redis OSS to ≥7.2.11, ≥7.4.6, ≥8.0.4, or ≥8.2.2  
  - Upgrade Redis Enterprise/Software to ≥7.22.2-12  
  - For **Azure Redis**, monitor Microsoft’s patch rollout (handled automatically)

---

!!! info "Key Takeaways"
    - Redis uses **Major.Minor.Patch** for OSS and **Major.Minor.Patch-Build** for Enterprise.  
    - The **dash (-)** indicates an **internal build number** for Enterprise releases.  
    - “**and above**” means *subsequent patch builds within that same maintenance line*,  
      not all future major versions automatically include the fix.

[Redis Enterprise Software product lifecycle](https://redis.io/docs/latest/operate/rs/installing-upgrading/product-lifecycle/)