# Patch notes — cherry-pick from upstream 5c5e2b72 (partial)

Worktree: `/root/v2a-work-patch-dsh` (detached at `166680f` = master HEAD).
Scope: bring over **2** genuine bugfixes from upstream mack-a/v2ray-agent commit
`5c5e2b72` into the mcogh/v2a fork, **without** merging upstream and **without**
removing the 7 transaction / rollback helper functions that upstream dropped
but production still depends on.

Reference diff: `upstream-fix-5c5e2b7.diff` (at worktree root, untracked).

Protected (NOT touched):
- `beginAccountTransaction`
- `commitAccountTransaction`
- `rollbackAccountTransaction`
- `migrateSingBoxLegacyOutboundConfig`
- `removeXrayClientByUUID`
- `rollbackXrayXHTTPTLSDeployment`
- `restartXrayWithXHTTPTLSRollback`

These are wired into the mcogh-specific P0/P1/P2 repair / nftables / public-IP
subscription / mcogh attribution flows deployed across the 8 VPS fleet. Upstream
removed them in 5c5e2b72; we deliberately keep them.

---

## Fix 1 — `initRealityClientServersName`: only run `checkRealityDest` for sing-box installs

Files / locations:
- `install.sh`    ~L10565 (inside `initRealityClientServersName`, tail of function)
- `shell/install_en.sh` ~L10430 (same function)

Upstream hunk:
```
@@ -10204,8 +10018,10 @@ initRealityClientServersName() {
     realityDestDomain="${realityServerName}:${realityDomainPort}"
-    checkRealityDest
     echoContent yellow ...
+    if [[ "${coreInstallType}" == "2" || "${selectCoreType}" == "2" ]]; then
+        checkRealityDest
+    fi
```

Before:
```bash
realityDestDomain="${realityServerName}:${realityDomainPort}"
checkRealityDest
echoContent yellow "\n ---> 客户端可用域名: ${realityServerName}:${realityDomainPort}\n"
```

After:
```bash
if [[ -z "${realityDestDomain}" ]]; then
    realityDestDomain="${realityServerName}:${realityDomainPort}"
fi
# checkRealityDest only when sing-box core is selected (coreInstallType=2).
# Running it unconditionally causes unnecessary outbound HTTPS requests when installing xray-only.
if [[ "${coreInstallType}" == "2" || "${selectCoreType}" == "2" ]]; then
    checkRealityDest
fi
echoContent yellow "\n ---> 客户端可用域名: ${realityServerName}:${realityDomainPort}\n"
```

Rationale:
- `checkRealityDest` performs an outbound HTTPS probe against the candidate
  reality dest domain. That probe is only meaningful when the user is
  installing the **sing-box** core (`coreInstallType=2` / `selectCoreType=2`),
  because reality dest is selected for the sing-box inbound in that path. For
  xray-only installs the call is wasted (slows install, can hang behind
  firewalls, leaks egress traffic).
- The extra `[[ -z "${realityDestDomain}" ]]` guard on top of upstream is
  defensive: if a previous code path already populated
  `realityDestDomain` (e.g. recovery flow), we keep that value instead of
  unconditionally overwriting with `${realityServerName}:${realityDomainPort}`.

## Fix 2 — `xraySelectionNeedsCustomPort`: empty selection = full install = custom ports needed

Files / locations:
- `install.sh`    ~L404
- `shell/install_en.sh` ~L414

Before:
```bash
xraySelectionNeedsCustomPort() {
    [[ "${1:-}" =~ ,(0|1|2|3|4|5), ]]
}
```

After:
```bash
xraySelectionNeedsCustomPort() {
    # Empty selection = full install sentinel — those protocols all need custom ports.
    [[ -z "${1:-}" ]] && return 0
    [[ "${1:-}" =~ ,(0|1|2|3|4|5), ]]
}
```

Rationale:
- Callers pass `"${selectCustomInstallType}"`. When the user chose the
  **full install** path (no per-protocol selection), this variable is the
  empty string. An empty selection semantically means "install everything",
  and "everything" includes VLESS TCP / Trojan (ids 0–5), which need custom
  ports.
- The regex `,(0|1|2|3|4|5),` never matches an empty string, so the function
  returned false on the full-install path and the installer skipped
  prompting for the custom port. Result: collision with the default port or
  an unconfigured inbound.
- Adding the `[[ -z ... ]] && return 0` short-circuit restores the intended
  "full install => needs custom port" behavior without changing any of the
  selective-install paths.

---

Validation:
- `bash -n install.sh` / `bash -n shell/install_en.sh` → syntax OK
- `git diff | grep` for the 7 protected function names → no hits
- No merge with upstream, no push, no changes outside this worktree.
