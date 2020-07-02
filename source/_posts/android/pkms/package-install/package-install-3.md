
---
title:Android Q 应用安装流程(二)--Package Install的核心
date:2020/07/01
category:
  - android
  - framework
---

# PackageManagerService.installPackagesTracedLI()
```java
    private void installPackagesTracedLI(List<InstallRequest> requests) {
        try {
            Trace.traceBegin(TRACE_TAG_PACKAGE_MANAGER, "installPackages");
            installPackagesLI(requests);
        } finally {
            Trace.traceEnd(TRACE_TAG_PACKAGE_MANAGER);
        }
    }
```

# PackageManagerService.installPackagesLI()
```java
    private void installPackagesLI(List<InstallRequest> requests) {
        final Map<String, ScanResult> preparedScans = new ArrayMap<>(requests.size());
        final Map<String, InstallArgs> installArgs = new ArrayMap<>(requests.size());
        final Map<String, PackageInstalledInfo> installResults = new ArrayMap<>(requests.size());
        final Map<String, PrepareResult> prepareResults = new ArrayMap<>(requests.size());
        final Map<String, VersionInfo> versionInfos = new ArrayMap<>(requests.size());
        final Map<String, PackageSetting> lastStaticSharedLibSettings =
                new ArrayMap<>(requests.size());
        final Map<String, Boolean> createdAppId = new ArrayMap<>(requests.size());
        boolean success = false;
        try {
            Trace.traceBegin(TRACE_TAG_PACKAGE_MANAGER, "installPackagesLI");
            for (InstallRequest request : requests) {
                // TODO(b/109941548): remove this once we've pulled everything from it and into
                //                    scan, reconcile or commit.
                final PrepareResult prepareResult;
                try {
                    Trace.traceBegin(TRACE_TAG_PACKAGE_MANAGER, "preparePackage");
                    //1. Prepare
                    prepareResult = preparePackageLI(request.args, request.installResult);
                } catch (PrepareFailure prepareFailure) {
                    request.installResult.setError(prepareFailure.error,
                            prepareFailure.getMessage());
                    request.installResult.origPackage = prepareFailure.conflictingPackage;
                    request.installResult.origPermission = prepareFailure.conflictingPermission;
                    return;
                } finally {
                    Trace.traceEnd(TRACE_TAG_PACKAGE_MANAGER);
                }
                request.installResult.setReturnCode(PackageManager.INSTALL_SUCCEEDED);
                request.installResult.installerPackageName = request.args.installerPackageName;

                final String packageName = prepareResult.packageToScan.packageName;
                prepareResults.put(packageName, prepareResult);
                installResults.put(packageName, request.installResult);
                installArgs.put(packageName, request.args);
                try {
                    //2. Scan
                    final List<ScanResult> scanResults = scanPackageTracedLI(
                            prepareResult.packageToScan, prepareResult.parseFlags,
                            prepareResult.scanFlags, System.currentTimeMillis(),
                            request.args.user);
                    for (ScanResult result : scanResults) {
                        if (null != preparedScans.put(result.pkgSetting.pkg.packageName, result)) {
                            request.installResult.setError(
                                    PackageManager.INSTALL_FAILED_DUPLICATE_PACKAGE,
                                    "Duplicate package " + result.pkgSetting.pkg.packageName
                                            + " in multi-package install request.");
                            return;
                        }
                        createdAppId.put(packageName, optimisticallyRegisterAppId(result));
                        versionInfos.put(result.pkgSetting.pkg.packageName,
                                getSettingsVersionForPackage(result.pkgSetting.pkg));
                        if (result.staticSharedLibraryInfo != null) {
                            final PackageSetting sharedLibLatestVersionSetting =
                                    getSharedLibLatestVersionSetting(result);
                            if (sharedLibLatestVersionSetting != null) {
                                lastStaticSharedLibSettings.put(result.pkgSetting.pkg.packageName,
                                        sharedLibLatestVersionSetting);
                            }
                        }
                    }
                } catch (PackageManagerException e) {
                    request.installResult.setError("Scanning Failed.", e);
                    return;
                }
            }
            ReconcileRequest reconcileRequest = new ReconcileRequest(preparedScans, installArgs,
                    installResults,
                    prepareResults,
                    mSharedLibraries,
                    Collections.unmodifiableMap(mPackages), versionInfos,
                    lastStaticSharedLibSettings);
            CommitRequest commitRequest = null;
            synchronized (mPackages) {
                Map<String, ReconciledPackage> reconciledPackages;
                try {
                    Trace.traceBegin(TRACE_TAG_PACKAGE_MANAGER, "reconcilePackages");
                    //3. Reconcile
                    reconciledPackages = reconcilePackagesLocked(
                            reconcileRequest, mSettings.mKeySetManagerService);
                } catch (ReconcileFailure e) {
                    for (InstallRequest request : requests) {
                        request.installResult.setError("Reconciliation failed...", e);
                    }
                    return;
                } finally {
                    Trace.traceEnd(TRACE_TAG_PACKAGE_MANAGER);
                }
                try {
                    Trace.traceBegin(TRACE_TAG_PACKAGE_MANAGER, "commitPackages");
                    commitRequest = new CommitRequest(reconciledPackages,
                            sUserManager.getUserIds());
                    //4. Commit
                    commitPackagesLocked(commitRequest);
                    success = true;
                } finally {
                    for (PrepareResult result : prepareResults.values()) {
                        if (result.freezer != null) {
                            result.freezer.close();
                        }
                    }
                    Trace.traceEnd(TRACE_TAG_PACKAGE_MANAGER);
                }
            }
            //5. dex
            executePostCommitSteps(commitRequest);
        } finally {
            if (!success) {
                for (ScanResult result : preparedScans.values()) {
                    if (createdAppId.getOrDefault(result.request.pkg.packageName, false)) {
                        cleanUpAppIdCreation(result);
                    }
                }
                // TODO(patb): create a more descriptive reason than unknown in future release
                // mark all non-failure installs as UNKNOWN so we do not treat them as success
                for (InstallRequest request : requests) {
                    if (request.installResult.returnCode == PackageManager.INSTALL_SUCCEEDED) {
                        request.installResult.returnCode = PackageManager.INSTALL_UNKNOWN;
                    }
                }
            }
            for (PrepareResult result : prepareResults.values()) {
                if (result.freezer != null) {
                    result.freezer.close();
                }
            }
            Trace.traceEnd(TRACE_TAG_PACKAGE_MANAGER);
        }
    }
```
根据注释中的文档，最终的安装分为4个步骤：
1. **Prepare**: Analyzes any current install state, parses the package and does initial validation on it.<br/>
**准备**：分析所有的安装状态，解析package并做初始的校验
2. **Scan**: Interrogates the parsed packages given the context collected in prepare.<br/>
**扫描**：分析从prepare中解析后packages的上下文
3. **Reconcile**: Validates scanned packages in the context of each other and the current system state to ensure that the install will be successful.<br/>
**一致性**：在各自的上下文中验证扫描后的pacakges和当前的系统状态来保证install成功
4. **Commit**: Commits all scanned packages and updates system state. This is the only place that system state may be modified in the install flow and all predictable errors must be determined before this phase.
**提交**:提交所有的已扫描的packages并且更新系统状态， 这步只是在安装流程中将系统状态修改并且所有可预知的错误必须在这个阶段之前就决定
下面根据这4个步骤一步步分析即可, 这里有个地方注意`if (null != preparedScans.put(result.pkgSetting.pkg.packageName, result))` 如果`preparedScan`中已经有该`package`的
`ScanResult`则返回null,如果已经有了就会返回`ScanResult`




