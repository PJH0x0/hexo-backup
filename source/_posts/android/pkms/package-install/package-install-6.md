
---
title:Android Q 应用安装流程(五)--Package一致性与系统更新
date:2020/07/01
category:
  - android
  - framework
---

# PackageManagerService.reconcilePackagesLocked()
```java
    private static Map<String, ReconciledPackage> reconcilePackagesLocked(
            final ReconcileRequest request, KeySetManagerService ksms)
            throws ReconcileFailure {
        final Map<String, ScanResult> scannedPackages = request.scannedPackages;

        final Map<String, ReconciledPackage> result = new ArrayMap<>(scannedPackages.size());

        // make a copy of the existing set of packages so we can combine them with incoming packages
        final ArrayMap<String, PackageParser.Package> combinedPackages =
                new ArrayMap<>(request.allPackages.size() + scannedPackages.size());
        combinedPackages.putAll(request.allPackages);

        final Map<String, LongSparseArray<SharedLibraryInfo>> incomingSharedLibraries =
                new ArrayMap<>();

        for (String installPackageName : scannedPackages.keySet()) {
            final ScanResult scanResult = scannedPackages.get(installPackageName);

            // add / replace existing with incoming packages
            combinedPackages.put(scanResult.pkgSetting.name, scanResult.request.pkg);

            // in the first pass, we'll build up the set of incoming shared libraries
            final List<SharedLibraryInfo> allowedSharedLibInfos =
                    getAllowedSharedLibInfos(scanResult, request.sharedLibrarySource);
            final SharedLibraryInfo staticLib = scanResult.staticSharedLibraryInfo;
            if (allowedSharedLibInfos != null) {
                for (SharedLibraryInfo info : allowedSharedLibInfos) {
                    if (!addSharedLibraryToPackageVersionMap(incomingSharedLibraries, info)) {
                        throw new ReconcileFailure("Static Shared Library " + staticLib.getName()
                                + " is being installed twice in this set!");
                    }
                }
            }

            // the following may be null if we're just reconciling on boot (and not during install)
            final InstallArgs installArgs = request.installArgs.get(installPackageName);
            final PackageInstalledInfo res = request.installResults.get(installPackageName);
            final PrepareResult prepareResult = request.preparedPackages.get(installPackageName);
            final boolean isInstall = installArgs != null;
            if (isInstall && (res == null || prepareResult == null)) {
                throw new ReconcileFailure("Reconcile arguments are not balanced for "
                        + installPackageName + "!");
            }

            final DeletePackageAction deletePackageAction;
            // we only want to try to delete for non system apps
            if (isInstall && prepareResult.replace && !prepareResult.system) {
                final boolean killApp = (scanResult.request.scanFlags & SCAN_DONT_KILL_APP) == 0;
                final int deleteFlags = PackageManager.DELETE_KEEP_DATA
                        | (killApp ? 0 : PackageManager.DELETE_DONT_KILL_APP);
                deletePackageAction = mayDeletePackageLocked(res.removedInfo,
                        prepareResult.originalPs, prepareResult.disabledPs,
                        prepareResult.childPackageSettings, deleteFlags, null /* all users */);
                if (deletePackageAction == null) {
                    throw new ReconcileFailure(
                            PackageManager.INSTALL_FAILED_REPLACE_COULDNT_DELETE,
                            "May not delete " + installPackageName + " to replace");
                }
            } else {
                deletePackageAction = null;
            }

            final int scanFlags = scanResult.request.scanFlags;
            final int parseFlags = scanResult.request.parseFlags;
            final PackageParser.Package pkg = scanResult.request.pkg;

            final PackageSetting disabledPkgSetting = scanResult.request.disabledPkgSetting;
            final PackageSetting lastStaticSharedLibSetting =
                    request.lastStaticSharedLibSettings.get(installPackageName);
            final PackageSetting signatureCheckPs =
                    (prepareResult != null && lastStaticSharedLibSetting != null)
                            ? lastStaticSharedLibSetting
                            : scanResult.pkgSetting;
            boolean removeAppKeySetData = false;
            boolean sharedUserSignaturesChanged = false;
            SigningDetails signingDetails = null;
            if (ksms.shouldCheckUpgradeKeySetLocked(signatureCheckPs, scanFlags)) {
                if (ksms.checkUpgradeKeySetLocked(signatureCheckPs, pkg)) {
                    // We just determined the app is signed correctly, so bring
                    // over the latest parsed certs.
                } else {
                    if ((parseFlags & PackageParser.PARSE_IS_SYSTEM_DIR) == 0) {
                        throw new ReconcileFailure(INSTALL_FAILED_UPDATE_INCOMPATIBLE,
                                "Package " + pkg.packageName + " upgrade keys do not match the "
                                        + "previously installed version");
                    } else {
                        String msg = "System package " + pkg.packageName
                                + " signature changed; retaining data.";
                        reportSettingsProblem(Log.WARN, msg);
                    }
                }
                signingDetails = pkg.mSigningDetails;
            } else {
                try {
                    final VersionInfo versionInfo = request.versionInfos.get(installPackageName);
                    final boolean compareCompat = isCompatSignatureUpdateNeeded(versionInfo);
                    final boolean compareRecover = isRecoverSignatureUpdateNeeded(versionInfo);
                    final boolean compatMatch = verifySignatures(signatureCheckPs,
                            disabledPkgSetting, pkg.mSigningDetails, compareCompat, compareRecover);
                    // The new KeySets will be re-added later in the scanning process.
                    if (compatMatch) {
                        removeAppKeySetData = true;
                    }
                    // We just determined the app is signed correctly, so bring
                    // over the latest parsed certs.
                    signingDetails = pkg.mSigningDetails;


                    // if this is is a sharedUser, check to see if the new package is signed by a
                    // newer
                    // signing certificate than the existing one, and if so, copy over the new
                    // details
                    if (signatureCheckPs.sharedUser != null) {
                        if (pkg.mSigningDetails.hasAncestor(
                                signatureCheckPs.sharedUser.signatures.mSigningDetails)) {
                            signatureCheckPs.sharedUser.signatures.mSigningDetails =
                                    pkg.mSigningDetails;
                        }
                        if (signatureCheckPs.sharedUser.signaturesChanged == null) {
                            signatureCheckPs.sharedUser.signaturesChanged = Boolean.FALSE;
                        }
                    }
                } catch (PackageManagerException e) {
                    if ((parseFlags & PackageParser.PARSE_IS_SYSTEM_DIR) == 0) {
                        throw new ReconcileFailure(e);
                    }
                    signingDetails = pkg.mSigningDetails;

                    // If the system app is part of a shared user we allow that shared user to
                    // change
                    // signatures as well as part of an OTA. We still need to verify that the
                    // signatures
                    // are consistent within the shared user for a given boot, so only allow
                    // updating
                    // the signatures on the first package scanned for the shared user (i.e. if the
                    // signaturesChanged state hasn't been initialized yet in SharedUserSetting).
                    if (signatureCheckPs.sharedUser != null) {
                        final Signature[] sharedUserSignatures =
                                signatureCheckPs.sharedUser.signatures.mSigningDetails.signatures;
                        if (signatureCheckPs.sharedUser.signaturesChanged != null
                                && compareSignatures(sharedUserSignatures,
                                        pkg.mSigningDetails.signatures)
                                        != PackageManager.SIGNATURE_MATCH) {
                            if (SystemProperties.getInt("ro.product.first_api_level", 0) <= 29) {
                                // Mismatched signatures is an error and silently skipping system
                                // packages will likely break the device in unforeseen ways.
                                // However, we allow the device to boot anyway because, prior to Q,
                                // vendors were not expecting the platform to crash in this
                                // situation.
                                // This WILL be a hard failure on any new API levels after Q.
                                throw new ReconcileFailure(
                                        INSTALL_PARSE_FAILED_INCONSISTENT_CERTIFICATES,
                                        "Signature mismatch for shared user: "
                                                + scanResult.pkgSetting.sharedUser);
                            } else {
                                // Treat mismatched signatures on system packages using a shared
                                // UID as
                                // fatal for the system overall, rather than just failing to install
                                // whichever package happened to be scanned later.
                                throw new IllegalStateException(
                                        "Signature mismatch on system package "
                                                + pkg.packageName + " for shared user "
                                                + scanResult.pkgSetting.sharedUser);
                            }
                        }

                        sharedUserSignaturesChanged = true;
                        signatureCheckPs.sharedUser.signatures.mSigningDetails =
                                pkg.mSigningDetails;
                        signatureCheckPs.sharedUser.signaturesChanged = Boolean.TRUE;
                    }
                    // File a report about this.
                    String msg = "System package " + pkg.packageName
                            + " signature changed; retaining data.";
                    reportSettingsProblem(Log.WARN, msg);
                } catch (IllegalArgumentException e) {
                    // should never happen: certs matched when checking, but not when comparing
                    // old to new for sharedUser
                    throw new RuntimeException(
                            "Signing certificates comparison made on incomparable signing details"
                                    + " but somehow passed verifySignatures!", e);
                }
            }

            result.put(installPackageName,
                    new ReconciledPackage(request, installArgs, scanResult.pkgSetting,
                            res, request.preparedPackages.get(installPackageName), scanResult,
                            deletePackageAction, allowedSharedLibInfos, signingDetails,
                            sharedUserSignaturesChanged, removeAppKeySetData));
        }

        for (String installPackageName : scannedPackages.keySet()) {
            // Check all shared libraries and map to their actual file path.
            // We only do this here for apps not on a system dir, because those
            // are the only ones that can fail an install due to this.  We
            // will take care of the system apps by updating all of their
            // library paths after the scan is done. Also during the initial
            // scan don't update any libs as we do this wholesale after all
            // apps are scanned to avoid dependency based scanning.
            final ScanResult scanResult = scannedPackages.get(installPackageName);
            if ((scanResult.request.scanFlags & SCAN_BOOTING) != 0
                    || (scanResult.request.parseFlags & PackageParser.PARSE_IS_SYSTEM_DIR) != 0) {
                continue;
            }
            try {
                result.get(installPackageName).collectedSharedLibraryInfos =
                        collectSharedLibraryInfos(scanResult.request.pkg, combinedPackages,
                                request.sharedLibrarySource, incomingSharedLibraries);

            } catch (PackageManagerException e) {
                throw new ReconcileFailure(e.error, e.getMessage());
            }
        }

        return result;
    }

```
此方法我只列出了与安装相关的代码，主要就是检查签名是否匹配,其他的都是开机时使用的。如果是覆盖安装的应用在`preparePackageLI()`方法中设置完成签名的检测

# PackageManagerService.commitPackagesLocked()
```java
    private void commitPackagesLocked(final CommitRequest request) {
        // TODO: remove any expected failures from this method; this should only be able to fail due
        //       to unavoidable errors (I/O, etc.)
        for (ReconciledPackage reconciledPkg : request.reconciledPackages.values()) {
            final ScanResult scanResult = reconciledPkg.scanResult;
            final ScanRequest scanRequest = scanResult.request;
            final PackageParser.Package pkg = scanRequest.pkg;
            final String packageName = pkg.packageName;
            final PackageInstalledInfo res = reconciledPkg.installResult;

            if (reconciledPkg.prepareResult.replace) {
                PackageParser.Package oldPackage = mPackages.get(packageName);

                // Set the update and install times
                PackageSetting deletedPkgSetting = (PackageSetting) oldPackage.mExtras;
                setInstallAndUpdateTime(pkg, deletedPkgSetting.firstInstallTime,
                        System.currentTimeMillis());

                if (reconciledPkg.prepareResult.system) {
                    // Remove existing system package
                    removePackageLI(oldPackage, true);
                    if (!disableSystemPackageLPw(oldPackage, pkg)) {
                        // We didn't need to disable the .apk as a current system package,
                        // which means we are replacing another update that is already
                        // installed.  We need to make sure to delete the older one's .apk.
                        res.removedInfo.args = createInstallArgsForExisting(
                                oldPackage.applicationInfo.getCodePath(),
                                oldPackage.applicationInfo.getResourcePath(),
                                getAppDexInstructionSets(oldPackage.applicationInfo));
                    } else {
                        res.removedInfo.args = null;
                    }
                    /******移除childpackages******/
                } else {
                    try {
                        executeDeletePackageLIF(reconciledPkg.deletePackageAction, packageName,
                                true, request.mAllUsers, true, pkg);
                    } catch (SystemDeleteException e) {
                        if (Build.IS_ENG) {
                            throw new RuntimeException("Unexpected failure", e);
                            // ignore; not possible for non-system app
                        }
                    }
                    // Successfully deleted the old package; proceed with replace.

                    // If deleted package lived in a container, give users a chance to
                    // relinquish resources before killing.
                    if (oldPackage.isForwardLocked() || isExternal(oldPackage)) {
                        if (DEBUG_INSTALL) {
                            Slog.i(TAG, "upgrading pkg " + oldPackage
                                    + " is ASEC-hosted -> UNAVAILABLE");
                        }
                        final int[] uidArray = new int[]{oldPackage.applicationInfo.uid};
                        final ArrayList<String> pkgList = new ArrayList<>(1);
                        pkgList.add(oldPackage.applicationInfo.packageName);
                        sendResourcesChangedBroadcast(false, true, pkgList, uidArray, null);
                    }
                    /*******移除不知道有什么意义的代码******/
                }
            }

            commitReconciledScanResultLocked(reconciledPkg);
            updateSettingsLI(pkg, reconciledPkg.installArgs.installerPackageName, request.mAllUsers,
                    res, reconciledPkg.installArgs.user, reconciledPkg.installArgs.installReason);

            final PackageSetting ps = mSettings.mPackages.get(packageName);
            if (ps != null) {
                res.newUsers = ps.queryInstalledUsers(sUserManager.getUserIds(), true);
                ps.setUpdateAvailable(false /*updateAvailable*/);
            }
            final int childCount = (pkg.childPackages != null) ? pkg.childPackages.size() : 0;
            for (int i = 0; i < childCount; i++) {
                PackageParser.Package childPkg = pkg.childPackages.get(i);
                PackageInstalledInfo childRes = res.addedChildPackages.get(
                        childPkg.packageName);
                PackageSetting childPs = mSettings.getPackageLPr(childPkg.packageName);
                if (childPs != null) {
                    childRes.newUsers = childPs.queryInstalledUsers(
                            sUserManager.getUserIds(), true);
                }
            }
            if (res.returnCode == PackageManager.INSTALL_SUCCEEDED) {
                updateSequenceNumberLP(ps, res.newUsers);
                updateInstantAppInstallerLocked(packageName);
            }
        }
    }
```
这个方法最重要的一点就是当覆盖安装的时候，删除老的应用的信息,如果是系统应用则删除对应的Package，其他的则是卸载其应用，不过并没有删除data，
这里有两个核心方法，分别是:
1. `commitReconciledScanResultLocked()`是更新系统`mSettings.mPackages`和`mPackages`信息的, `mSettings.mPackages`是保存`PacakgeSetting`的集合。
`mPackages`是`PackageParser.Package`的集合
2. `updateSettingsLI()`是
