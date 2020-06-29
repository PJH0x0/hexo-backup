---
title:Android Q PackageInstaller安装Package流程(二)
date:2020/06/15
category:
  - android
  - framework
---

## PackageManagerService.installStage()

```java
    void installStage(ActiveInstallSession activeInstallSession) {
        if (DEBUG_INSTANT) {
            if ((activeInstallSession.getSessionParams().installFlags
                    & PackageManager.INSTALL_INSTANT_APP) != 0) {
                Slog.d(TAG, "Ephemeral install of " + activeInstallSession.getPackageName());
            }
        }
        final Message msg = mHandler.obtainMessage(INIT_COPY);
        final InstallParams params = new InstallParams(activeInstallSession);
        params.setTraceMethod("installStage").setTraceCookie(System.identityHashCode(params));
        msg.obj = params;

        Trace.asyncTraceBegin(TRACE_TAG_PACKAGE_MANAGER, "installStage",
                System.identityHashCode(msg.obj));
        Trace.asyncTraceBegin(TRACE_TAG_PACKAGE_MANAGER, "queueInstall",
                System.identityHashCode(msg.obj));

        mHandler.sendMessage(msg);
    }

```

## PackageManagerService.PackageHandler.doHandleMessage()

```java
    void doHandleMessage(Message msg) {
        switch (msg.what) {
            case INIT_COPY: {
                HandlerParams params = (HandlerParams) msg.obj;
                if (params != null) {
                    if (DEBUG_INSTALL) Slog.i(TAG, "init_copy: " + params);
                    Trace.asyncTraceEnd(TRACE_TAG_PACKAGE_MANAGER, "queueInstall",
                            System.identityHashCode(params));
                    Trace.traceBegin(TRACE_TAG_PACKAGE_MANAGER, "startCopy");
                    params.startCopy();
                    Trace.traceEnd(TRACE_TAG_PACKAGE_MANAGER);
                }
                break;
            }
        }
    }
```

这里`HandlerParams`指的是`installStage`中创建的`InstallParams`

## PackageManagerService.HandlerParams.startCopy()

```java
    final void startCopy() {
        if (DEBUG_INSTALL) Slog.i(TAG, "startCopy " + mUser + ": " + this);
        handleStartCopy();
        handleReturnCode();
    }

```

### PackageManagerService.InstallParams.handleStartCopy()

```java
    public void handleStartCopy() {
        int ret = PackageManager.INSTALL_SUCCEEDED;

        // If we're already staged, we've firmly committed to an install location
        if (origin.staged) {
            if (origin.file != null) {
                installFlags |= PackageManager.INSTALL_INTERNAL;
            } else {
                throw new IllegalStateException("Invalid stage location");
            }
        }

        final boolean onInt = (installFlags & PackageManager.INSTALL_INTERNAL) != 0;
        final boolean ephemeral = (installFlags & PackageManager.INSTALL_INSTANT_APP) != 0;
        PackageInfoLite pkgLite = null;


        pkgLite = PackageManagerServiceUtils.getMinimalPackageInfo(mContext,
                origin.resolvedPath, installFlags, packageAbiOverride);

        if (DEBUG_INSTANT && ephemeral) {
            Slog.v(TAG, "pkgLite for install: " + pkgLite);
        }

        /*
         * If we have too little free space, try to free cache
         * before giving up.
         */
        if (!origin.staged && pkgLite.recommendedInstallLocation
                == PackageHelper.RECOMMEND_FAILED_INSUFFICIENT_STORAGE) {
            // TODO: focus freeing disk space on the target device
            final StorageManager storage = StorageManager.from(mContext);
            final long lowThreshold = storage.getStorageLowBytes(
                    Environment.getDataDirectory());

            final long sizeBytes = PackageManagerServiceUtils.calculateInstalledSize(
                    origin.resolvedPath, packageAbiOverride);
            if (sizeBytes >= 0) {
                try {
                    mInstaller.freeCache(null, sizeBytes + lowThreshold, 0, 0);
                    pkgLite = PackageManagerServiceUtils.getMinimalPackageInfo(mContext,
                            origin.resolvedPath, installFlags, packageAbiOverride);
                } catch (InstallerException e) {
                    Slog.w(TAG, "Failed to free cache", e);
                }
            }

            /*
             * The cache free must have deleted the file we downloaded to install.
             *
             * TODO: fix the "freeCache" call to not delete the file we care about.
             */
            if (pkgLite.recommendedInstallLocation
                    == PackageHelper.RECOMMEND_FAILED_INVALID_URI) {
                pkgLite.recommendedInstallLocation
                        = PackageHelper.RECOMMEND_FAILED_INSUFFICIENT_STORAGE;
            }
        }


        if (ret == PackageManager.INSTALL_SUCCEEDED) {
            int loc = pkgLite.recommendedInstallLocation;
            if (loc == PackageHelper.RECOMMEND_FAILED_INVALID_LOCATION) {
                ret = PackageManager.INSTALL_FAILED_INVALID_INSTALL_LOCATION;
            } else if (loc == PackageHelper.RECOMMEND_FAILED_ALREADY_EXISTS) {
                ret = PackageManager.INSTALL_FAILED_ALREADY_EXISTS;
            } else if (loc == PackageHelper.RECOMMEND_FAILED_INSUFFICIENT_STORAGE) {
                ret = PackageManager.INSTALL_FAILED_INSUFFICIENT_STORAGE;
            } else if (loc == PackageHelper.RECOMMEND_FAILED_INVALID_APK) {
                ret = PackageManager.INSTALL_FAILED_INVALID_APK;
            } else if (loc == PackageHelper.RECOMMEND_FAILED_INVALID_URI) {
                ret = PackageManager.INSTALL_FAILED_INVALID_URI;
            } else if (loc == PackageHelper.RECOMMEND_MEDIA_UNAVAILABLE) {
                ret = PackageManager.INSTALL_FAILED_MEDIA_UNAVAILABLE;
            } else {
                // Override with defaults if needed.
                loc = installLocationPolicy(pkgLite);
                if (loc == PackageHelper.RECOMMEND_FAILED_VERSION_DOWNGRADE) {
                    ret = PackageManager.INSTALL_FAILED_VERSION_DOWNGRADE;
                } else if (loc == PackageHelper.RECOMMEND_FAILED_WRONG_INSTALLED_VERSION) {
                    ret = PackageManager.INSTALL_FAILED_WRONG_INSTALLED_VERSION;
                } else if (!onInt) {
                    // Override install location with flags
                    if (loc == PackageHelper.RECOMMEND_INSTALL_EXTERNAL) {
                        // Set the flag to install on external media.
                        installFlags &= ~PackageManager.INSTALL_INTERNAL;
                    } else if (loc == PackageHelper.RECOMMEND_INSTALL_EPHEMERAL) {
                        if (DEBUG_INSTANT) {
                            Slog.v(TAG, "...setting INSTALL_EPHEMERAL install flag");
                        }
                        installFlags |= PackageManager.INSTALL_INSTANT_APP;
                        installFlags &= ~PackageManager.INSTALL_INTERNAL;
                    } else {
                        // Make sure the flag for installing on external
                        // media is unset
                        installFlags |= PackageManager.INSTALL_INTERNAL;
                    }
                }
            }
        }

        final InstallArgs args = createInstallArgs(this);
        mVerificationCompleted = true;
        mEnableRollbackCompleted = true;
        mArgs = args;

        if (ret == PackageManager.INSTALL_SUCCEEDED) {
            // TODO: http://b/22976637
            // Apps installed for "all" users use the device owner to verify the app
            UserHandle verifierUser = getUser();
            if (verifierUser == UserHandle.ALL) {
                verifierUser = UserHandle.SYSTEM;
            }

            /*
             * Determine if we have any installed package verifiers. If we
             * do, then we'll defer to them to verify the packages.
             */
            final int requiredUid = mRequiredVerifierPackage == null ? -1
                    : getPackageUid(mRequiredVerifierPackage, MATCH_DEBUG_TRIAGED_MISSING,
                            verifierUser.getIdentifier());
            final int installerUid =
                    verificationInfo == null ? -1 : verificationInfo.installerUid;
            if (!origin.existing && requiredUid != -1
                    && isVerificationEnabled(
                            verifierUser.getIdentifier(), installFlags, installerUid)) {
                final Intent verification = new Intent(
                        Intent.ACTION_PACKAGE_NEEDS_VERIFICATION);
                verification.addFlags(Intent.FLAG_RECEIVER_FOREGROUND);
                verification.setDataAndType(Uri.fromFile(new File(origin.resolvedPath)),
                        PACKAGE_MIME_TYPE);
                verification.addFlags(Intent.FLAG_GRANT_READ_URI_PERMISSION);

                // Query all live verifiers based on current user state
                final List<ResolveInfo> receivers = queryIntentReceiversInternal(verification,
                        PACKAGE_MIME_TYPE, 0, verifierUser.getIdentifier(),
                        false /*allowDynamicSplits*/);

                if (DEBUG_VERIFY) {
                    Slog.d(TAG, "Found " + receivers.size() + " verifiers for intent "
                            + verification.toString() + " with " + pkgLite.verifiers.length
                            + " optional verifiers");
                }

                final int verificationId = mPendingVerificationToken++;

                verification.putExtra(PackageManager.EXTRA_VERIFICATION_ID, verificationId);

                verification.putExtra(PackageManager.EXTRA_VERIFICATION_INSTALLER_PACKAGE,
                        installerPackageName);

                verification.putExtra(PackageManager.EXTRA_VERIFICATION_INSTALL_FLAGS,
                        installFlags);

                verification.putExtra(PackageManager.EXTRA_VERIFICATION_PACKAGE_NAME,
                        pkgLite.packageName);

                verification.putExtra(PackageManager.EXTRA_VERIFICATION_VERSION_CODE,
                        pkgLite.versionCode);

                verification.putExtra(PackageManager.EXTRA_VERIFICATION_LONG_VERSION_CODE,
                        pkgLite.getLongVersionCode());

                if (verificationInfo != null) {
                    if (verificationInfo.originatingUri != null) {
                        verification.putExtra(Intent.EXTRA_ORIGINATING_URI,
                                verificationInfo.originatingUri);
                    }
                    if (verificationInfo.referrer != null) {
                        verification.putExtra(Intent.EXTRA_REFERRER,
                                verificationInfo.referrer);
                    }
                    if (verificationInfo.originatingUid >= 0) {
                        verification.putExtra(Intent.EXTRA_ORIGINATING_UID,
                                verificationInfo.originatingUid);
                    }
                    if (verificationInfo.installerUid >= 0) {
                        verification.putExtra(PackageManager.EXTRA_VERIFICATION_INSTALLER_UID,
                                verificationInfo.installerUid);
                    }
                }

                final PackageVerificationState verificationState = new PackageVerificationState(
                        requiredUid, this);

                mPendingVerification.append(verificationId, verificationState);

                final List<ComponentName> sufficientVerifiers = matchVerifiers(pkgLite,
                        receivers, verificationState);

                DeviceIdleController.LocalService idleController = getDeviceIdleController();
                final long idleDuration = getVerificationTimeout();

                /*
                 * If any sufficient verifiers were listed in the package
                 * manifest, attempt to ask them.
                 */
                if (sufficientVerifiers != null) {
                    final int N = sufficientVerifiers.size();
                    if (N == 0) {
                        Slog.i(TAG, "Additional verifiers required, but none installed.");
                        ret = PackageManager.INSTALL_FAILED_VERIFICATION_FAILURE;
                    } else {
                        for (int i = 0; i < N; i++) {
                            final ComponentName verifierComponent = sufficientVerifiers.get(i);
                            idleController.addPowerSaveTempWhitelistApp(Process.myUid(),
                                    verifierComponent.getPackageName(), idleDuration,
                                    verifierUser.getIdentifier(), false, "package verifier");

                            final Intent sufficientIntent = new Intent(verification);
                            sufficientIntent.setComponent(verifierComponent);
                            mContext.sendBroadcastAsUser(sufficientIntent, verifierUser);
                        }
                    }
                }

                final ComponentName requiredVerifierComponent = matchComponentForVerifier(
                        mRequiredVerifierPackage, receivers);
                if (ret == PackageManager.INSTALL_SUCCEEDED
                        && mRequiredVerifierPackage != null) {
                    Trace.asyncTraceBegin(
                            TRACE_TAG_PACKAGE_MANAGER, "verification", verificationId);
                    /*
                     * Send the intent to the required verification agent,
                     * but only start the verification timeout after the
                     * target BroadcastReceivers have run.
                     */
                    verification.setComponent(requiredVerifierComponent);
                    idleController.addPowerSaveTempWhitelistApp(Process.myUid(),
                            mRequiredVerifierPackage, idleDuration,
                            verifierUser.getIdentifier(), false, "package verifier");
                    mContext.sendOrderedBroadcastAsUser(verification, verifierUser,
                            android.Manifest.permission.PACKAGE_VERIFICATION_AGENT,
                            new BroadcastReceiver() {
                                @Override
                                public void onReceive(Context context, Intent intent) {
                                    final Message msg = mHandler
                                            .obtainMessage(CHECK_PENDING_VERIFICATION);
                                    msg.arg1 = verificationId;
                                    mHandler.sendMessageDelayed(msg, getVerificationTimeout());
                                }
                            }, null, 0, null, null);

                    /*
                     * We don't want the copy to proceed until verification
                     * succeeds.
                     */
                    mVerificationCompleted = false;
                }
            }

            if ((installFlags & PackageManager.INSTALL_ENABLE_ROLLBACK) != 0) {
                /******隐藏ROLLBACK处理******/
            }
        }

        mRet = ret;
    }
```

这个方法的代码过长，可以看做是start copy的前置动作，有以下三个主要功能:

1. 通过`PackageManagerServiceUtils.getMinimalPackageInfo()`方法获取PackageInfoLite, 感觉parsePackageLite使用次数是不是太多了些，好在解析的时间比较短
2. 处理安装Package时存储空间不足的情况，去除cache的然后判断是否有可供安装的空间
3. 调用`installLocationPolicy()`处理应用更新和覆盖安装的情况
4. `createInstallArgs()`创建`FileInstallArgs`
5. 验证应用，这步应该是专门为Google GMS或者OEM厂商的应用商店设计的，校验应用的安全性,这一步会耗费非常多的时间
6. 处理ROLL_BACK标志，暂时不清楚这个标志有什么用处

## PackageManagerService.InstallParams.handleReturnCode()

```java
    @Override
    void handleReturnCode() {
        if (mVerificationCompleted && mEnableRollbackCompleted) {
            if ((installFlags & PackageManager.INSTALL_DRY_RUN) != 0) {
                String packageName = "";
                try {
                    PackageLite packageInfo =
                            new PackageParser().parsePackageLite(origin.file, 0);
                    packageName = packageInfo.packageName;
                } catch (PackageParserException e) {
                    Slog.e(TAG, "Can't parse package at " + origin.file.getAbsolutePath(), e);
                }
                try {
                    observer.onPackageInstalled(packageName, mRet, "Dry run", new Bundle());
                } catch (RemoteException e) {
                    Slog.i(TAG, "Observer no longer exists.");
                }
                return;
            }
            if (mRet == PackageManager.INSTALL_SUCCEEDED) {
                mRet = mArgs.copyApk();
            }
            processPendingInstall(mArgs, mRet);
        }
    }
```
这里有一点要注意的是`mVerificationCompleted`要等Google应用验证完成之后才会为`true`, 具体看`handleVerificationFinished()`的调用链，暂时先不展开说了。
由于之前是有copy apk到/data/app中，所以不需要再copy apk, 所以mRet直接为`INSTALL_SUCCEEDED`

## PackageManagerService.processPendingInstall()

```java
    private void processPendingInstall(final InstallArgs args, final int currentStatus) {
        if (args.mMultiPackageInstallParams != null) {
            args.mMultiPackageInstallParams.tryProcessInstallRequest(args, currentStatus);
        } else {
            PackageInstalledInfo res = createPackageInstalledInfo(currentStatus);
            processInstallRequestsAsync(
                    res.returnCode == PackageManager.INSTALL_SUCCEEDED,
                    Collections.singletonList(new InstallRequest(args, res)));
        }
    }

    // Queue up an async operation since the package installation may take a little while.
    private void processInstallRequestsAsync(boolean success,
            List<InstallRequest> installRequests) {
        mHandler.post(() -> {
            if (success) {
                for (InstallRequest request : installRequests) {
                    request.args.doPreInstall(request.installResult.returnCode);
                }
                synchronized (mInstallLock) {
                    installPackagesTracedLI(installRequests);
                }
                for (InstallRequest request : installRequests) {
                    request.args.doPostInstall(
                            request.installResult.returnCode, request.installResult.uid);
                }
            }
            for (InstallRequest request : installRequests) {
                restoreAndPostInstall(request.args.user.getIdentifier(), request.installResult,
                        new PostInstallData(request.args, request.installResult, null));
            }
        });
    }
```
这个方法的核心就是`installPackagesTracedLI()`方法，`FileInstallArgs`中的`doPreInstall()`和`doPostInstall()`两个方法都是判断是否安装成功，然后对其进行清理

## PackageManagerService.installPackagesTracedLI()
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

## PackageManagerService.installPackagesLI()
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


### 1. PackageManagerService.preparePackageLI()
这个方法实在是太长了，只能简略的先看一下核心的代码了

#### 1. 解析APK
这次解析出来的`Package`才是全的，而不是解析`PackageLite`，这里有两个flag`PARSE_CHATTY`和`PARSE_ENFORCE_CODE`目前还不知道是什么意思，好像`PackageParser`中都没用到
```java
        @ParseFlags final int parseFlags = mDefParseFlags | PackageParser.PARSE_CHATTY
                | PackageParser.PARSE_ENFORCE_CODE
                | (onExternal ? PackageParser.PARSE_EXTERNAL_STORAGE : 0);

        PackageParser pp = new PackageParser();
        pp.setSeparateProcesses(mSeparateProcesses);
        pp.setDisplayMetrics(mMetrics);
        pp.setCallback(mPackageParserCallback);

        Trace.traceBegin(TRACE_TAG_PACKAGE_MANAGER, "parsePackage");
        final PackageParser.Package pkg;
        try {
            pkg = pp.parsePackage(tmpPackageFile, parseFlags);
            DexMetadataHelper.validatePackageDexMetadata(pkg);
        } catch (PackageParserException e) {
            throw new PrepareFailure("Failed parse during installPackageLI", e);
        } finally {
            Trace.traceEnd(TRACE_TAG_PACKAGE_MANAGER);
        }
```
#### 2. 收集签名文件

这步是收集和验证签名，签名的流程和原理单独的文章再说
```java
        try {
            // either use what we've been given or parse directly from the APK
            if (args.signingDetails != PackageParser.SigningDetails.UNKNOWN) {
                pkg.setSigningDetails(args.signingDetails);
            } else {
                PackageParser.collectCertificates(pkg, false /* skipVerify */);
            }
        } catch (PackageParserException e) {
            throw new PrepareFailure("Failed collect during installPackageLI", e);
        }
```
#### 3. 重命名文件夹

这步是将之前的`vmdl+sessionId.tmp`文件夹重命名为`packageName-random`, 注意看`getNextCodePath()`方法, 
`setUpFsVerityIfPossible()`涉及到`fs-verity`，需要内核支持，暂时不讨论

```java
//preparePackageLI()
    if (!args.doRename(res.returnCode, pkg)) {
        throw new PrepareFailure(INSTALL_FAILED_INSUFFICIENT_STORAGE, "Failed rename");
    }

    try {
        setUpFsVerityIfPossible(pkg);
    } catch (InstallerException | IOException | DigestException | NoSuchAlgorithmException e) {
        throw new PrepareFailure(INSTALL_FAILED_INTERNAL_ERROR,
                "Failed to set up verity: " + e);
    }
//doRename()
    boolean doRename(int status, PackageParser.Package pkg) {
        if (status != PackageManager.INSTALL_SUCCEEDED) {
            cleanUp();
            return false;
        }

        final File targetDir = codeFile.getParentFile();
        final File beforeCodeFile = codeFile;
        final File afterCodeFile = getNextCodePath(targetDir, pkg.packageName);

        if (DEBUG_INSTALL) Slog.d(TAG, "Renaming " + beforeCodeFile + " to " + afterCodeFile);
        try {
            Os.rename(beforeCodeFile.getAbsolutePath(), afterCodeFile.getAbsolutePath());
        } catch (ErrnoException e) {
            Slog.w(TAG, "Failed to rename", e);
            return false;
        }

        if (!SELinux.restoreconRecursive(afterCodeFile)) {
            Slog.w(TAG, "Failed to restorecon");
            return false;
        }

        // Reflect the rename internally
        codeFile = afterCodeFile;
        resourceFile = afterCodeFile;

        // Reflect the rename in scanned details
        try {
            pkg.setCodePath(afterCodeFile.getCanonicalPath());
        } catch (IOException e) {
            Slog.e(TAG, "Failed to get path: " + afterCodeFile, e);
            return false;
        }
        pkg.setBaseCodePath(FileUtils.rewriteAfterRename(beforeCodeFile,
                afterCodeFile, pkg.baseCodePath));
        pkg.setSplitCodePaths(FileUtils.rewriteAfterRename(beforeCodeFile,
                afterCodeFile, pkg.splitCodePaths));

        // Reflect the rename in app info
        pkg.setApplicationVolumeUuid(pkg.volumeUuid);
        pkg.setApplicationInfoCodePath(pkg.codePath);
        pkg.setApplicationInfoBaseCodePath(pkg.baseCodePath);
        pkg.setApplicationInfoSplitCodePaths(pkg.splitCodePaths);
        pkg.setApplicationInfoResourcePath(pkg.codePath);
        pkg.setApplicationInfoBaseResourcePath(pkg.baseCodePath);
        pkg.setApplicationInfoSplitResourcePaths(pkg.splitCodePaths);

        return true;
    }
//getNextCodePath()
    private File getNextCodePath(File targetDir, String packageName) {
        File result;
        SecureRandom random = new SecureRandom();
        byte[] bytes = new byte[16];
        do {
            random.nextBytes(bytes);
            String suffix = Base64.encodeToString(bytes, Base64.URL_SAFE | Base64.NO_WRAP);
            result = new File(targetDir, packageName + "-" + suffix);
        } while (result.exists());
        return result;
    }
```

#### 4. 冻结APP
这部分很好理解安装应用的时候，需要kill应用，最终调用是到`PackageFreezer`的构造方法
```java
    final PackageFreezer freezer =
            freezePackageForInstall(pkgName, installFlags, "installPackageLI");

//preparePackageLI->PackageFreezer.constructor
    public PackageFreezer(String packageName, int userId, String killReason) {
        synchronized (mPackages) {
            mPackageName = packageName;
            mWeFroze = mFrozenPackages.add(mPackageName);

            final PackageSetting ps = mSettings.mPackages.get(mPackageName);
            if (ps != null) {
                killApplication(ps.name, ps.appId, userId, killReason);
            }

            final PackageParser.Package p = mPackages.get(packageName);
            if (p != null && p.childPackages != null) {
                final int N = p.childPackages.size();
                mChildren = new PackageFreezer[N];
                for (int i = 0; i < N; i++) {
                    mChildren[i] = new PackageFreezer(p.childPackages.get(i).packageName,
                            userId, killReason);
                }
            } else {
                mChildren = null;
            }
        }
        mCloseGuard.open("close");
    }
```
#### 5. 处理覆盖安装的情况

```java
    if (replace) {
        targetVolumeUuid = null;
        if (pkg.applicationInfo.isStaticSharedLibrary()) {
            // Static libs have a synthetic package name containing the version
            // and cannot be updated as an update would get a new package name,
            // unless this is the exact same version code which is useful for
            // development.
            PackageParser.Package existingPkg = mPackages.get(pkg.packageName);
            if (existingPkg != null
                    && existingPkg.getLongVersionCode() != pkg.getLongVersionCode()) {
                throw new PrepareFailure(INSTALL_FAILED_DUPLICATE_PACKAGE,
                        "Packages declaring "
                                + "static-shared libs cannot be updated");
            }
        }

        final boolean isInstantApp = (scanFlags & SCAN_AS_INSTANT_APP) != 0;

        final PackageParser.Package oldPackage;
        final String pkgName11 = pkg.packageName;
        final int[] allUsers;
        final int[] installedUsers;

        synchronized (mPackages) {
            oldPackage = mPackages.get(pkgName11);
            existingPackage = oldPackage;
            if (DEBUG_INSTALL) {
                Slog.d(TAG,
                        "replacePackageLI: new=" + pkg + ", old=" + oldPackage);
            }

            ps = mSettings.mPackages.get(pkgName11);
            disabledPs = mSettings.getDisabledSystemPkgLPr(ps);

            // verify signatures are valid
            final KeySetManagerService ksms = mSettings.mKeySetManagerService;
            if (ksms.shouldCheckUpgradeKeySetLocked(ps, scanFlags)) {
                if (!ksms.checkUpgradeKeySetLocked(ps, pkg)) {
                    throw new PrepareFailure(INSTALL_FAILED_UPDATE_INCOMPATIBLE,
                            "New package not signed by keys specified by upgrade-keysets: "
                                    + pkgName11);
                }
            } else {
                // default to original signature matching
                if (!pkg.mSigningDetails.checkCapability(oldPackage.mSigningDetails,
                        SigningDetails.CertCapabilities.INSTALLED_DATA)
                        && !oldPackage.mSigningDetails.checkCapability(
                        pkg.mSigningDetails,
                        SigningDetails.CertCapabilities.ROLLBACK)) {
                    throw new PrepareFailure(INSTALL_FAILED_UPDATE_INCOMPATIBLE,
                            "New package has a different signature: " + pkgName11);
                }
            }

            /******移除对于restrict-update tag的处理******/
            // Check for shared user id changes
            String invalidPackageName =
                    getParentOrChildPackageChangedSharedUser(oldPackage, pkg);
            if (invalidPackageName != null) {
                throw new PrepareFailure(INSTALL_FAILED_SHARED_USER_INCOMPATIBLE,
                        "Package " + invalidPackageName + " tried to change user "
                                + oldPackage.mSharedUserId);
            }

            // In case of rollback, remember per-user/profile install state
            allUsers = sUserManager.getUserIds();
            installedUsers = ps.queryInstalledUsers(allUsers, true);

            /******移除对于instantApp的处理******/
        }

        // Update what is removed
        res.removedInfo = new PackageRemovedInfo(this);
        res.removedInfo.uid = oldPackage.applicationInfo.uid;
        res.removedInfo.removedPackage = oldPackage.packageName;
        res.removedInfo.installerPackageName = ps.installerPackageName;
        res.removedInfo.isStaticSharedLib = pkg.staticSharedLibName != null;
        res.removedInfo.isUpdate = true;
        res.removedInfo.origUsers = installedUsers;
        res.removedInfo.installReasons = new SparseArray<>(installedUsers.length);
        for (int i = 0; i < installedUsers.length; i++) {
            final int userId = installedUsers[i];
            res.removedInfo.installReasons.put(userId, ps.getInstallReason(userId));
        }

        sysPkg = (isSystemApp(oldPackage));
        if (sysPkg) {
            // Set the system/privileged/oem/vendor/product flags as needed
            final boolean privileged = isPrivilegedApp(oldPackage);
            final boolean oem = isOemApp(oldPackage);
            final boolean vendor = isVendorApp(oldPackage);
            final boolean product = isProductApp(oldPackage);
            final boolean odm = isOdmApp(oldPackage);
            final @ParseFlags int systemParseFlags = parseFlags;
            final @ScanFlags int systemScanFlags = scanFlags
                    | SCAN_AS_SYSTEM
                    | (privileged ? SCAN_AS_PRIVILEGED : 0)
                    | (oem ? SCAN_AS_OEM : 0)
                    | (vendor ? SCAN_AS_VENDOR : 0)
                    | (product ? SCAN_AS_PRODUCT : 0)
                    | (odm ? SCAN_AS_ODM : 0);

            if (DEBUG_INSTALL) {
                Slog.d(TAG, "replaceSystemPackageLI: new=" + pkg
                        + ", old=" + oldPackage);
            }
            res.setReturnCode(PackageManager.INSTALL_SUCCEEDED);
            pkg.setApplicationInfoFlags(ApplicationInfo.FLAG_UPDATED_SYSTEM_APP,
                    ApplicationInfo.FLAG_UPDATED_SYSTEM_APP);
            targetParseFlags = systemParseFlags;
            targetScanFlags = systemScanFlags;
        } else { // non system replace
            replace = true;
            if (DEBUG_INSTALL) {
                Slog.d(TAG,
                        "replaceNonSystemPackageLI: new=" + pkg + ", old="
                                + oldPackage);
            }

            String pkgName1 = oldPackage.packageName;
            boolean deletedPkg = true;
            boolean addedPkg = false;
            boolean updatedSettings = false;

            final long origUpdateTime = (pkg.mExtras != null)
                    ? ((PackageSetting) pkg.mExtras).lastUpdateTime : 0;

        }
    } 
```
这部分代码比较长，但是逻辑比较简单，需要注意的就是临时变量建的比较多，但是感觉没有必要。具体逻辑如下
1. 处理静态共享库，共享库不可以升级version
2. 验证签名是否相同
3. 检测是否修改了shared userid，如果更改了shared user id则会出现更新不成功的现象
4. 更新PackageRemovedInfo
5. 更新覆盖系统应用时的scanFlags, 为重启时的扫描做准备

#### 6. 处理新安装的情况

```java
    ps = null;
    childPackages = null;
    disabledPs = null;
    replace = false;
    existingPackage = null;
    // Remember this for later, in case we need to rollback this install
    String pkgName1 = pkg.packageName;

    if (DEBUG_INSTALL) Slog.d(TAG, "installNewPackageLI: " + pkg);

    // TODO(patb): MOVE TO RECONCILE
    synchronized (mPackages) {
        renamedPackage = mSettings.getRenamedPackageLPr(pkgName1);
        if (renamedPackage != null) {
            // A package with the same name is already installed, though
            // it has been renamed to an older name.  The package we
            // are trying to install should be installed as an update to
            // the existing one, but that has not been requested, so bail.
            throw new PrepareFailure(INSTALL_FAILED_ALREADY_EXISTS,
                    "Attempt to re-install " + pkgName1
                            + " without first uninstalling package running as "
                            + renamedPackage);
        }
        if (mPackages.containsKey(pkgName1)) {
            // Don't allow installation over an existing package with the same name.
            throw new PrepareFailure(INSTALL_FAILED_ALREADY_EXISTS,
                    "Attempt to re-install " + pkgName1
                            + " without first uninstalling.");
        }
    }
```
这里处理了一下两个异常，New install的应用不可以在系统中有已安装的应用，这种情况通常用于在adb install没有加-r参数的情况下会出现

### 2. PackageManagerService.scanPackageTracedLI()

这里要看清楚参数和返回值，不要和重载的另一个方法混淆了，那一个是在开机时调用的

```java
    private List<ScanResult> scanPackageTracedLI(PackageParser.Package pkg,
            final @ParseFlags int parseFlags, @ScanFlags int scanFlags, long currentTime,
            @Nullable UserHandle user) throws PackageManagerException {
        Trace.traceBegin(TRACE_TAG_PACKAGE_MANAGER, "scanPackage");
        // If the package has children and this is the first dive in the function
        // we recursively scan the package with the SCAN_CHECK_ONLY flag set to see
        // whether all packages (parent and children) would be successfully scanned
        // before the actual scan since scanning mutates internal state and we want
        // to atomically install the package and its children.
        if ((scanFlags & SCAN_CHECK_ONLY) == 0) {
            if (pkg.childPackages != null && pkg.childPackages.size() > 0) {
                scanFlags |= SCAN_CHECK_ONLY;
            }
        } else {
            scanFlags &= ~SCAN_CHECK_ONLY;
        }

        final int childCount = (pkg.childPackages != null) ? pkg.childPackages.size() : 0;
        final List<ScanResult> scanResults = new ArrayList<>(1 + childCount);
        try {
            // Scan the parent
            scanResults.add(scanPackageNewLI(pkg, parseFlags, scanFlags, currentTime, user));
            // Scan the children
            for (int i = 0; i < childCount; i++) {
                PackageParser.Package childPkg = pkg.childPackages.get(i);
                scanResults.add(scanPackageNewLI(childPkg, parseFlags,
                        scanFlags, currentTime, user));
            }
        } finally {
            Trace.traceEnd(TRACE_TAG_PACKAGE_MANAGER);
        }

        if ((scanFlags & SCAN_CHECK_ONLY) != 0) {
            return scanPackageTracedLI(pkg, parseFlags, scanFlags, currentTime, user);
        }

        return scanResults;
    }
```
主要核心方法是`scanPackageNewLI()`

### PackageManagerService.scanPackageNewLI()

```java
    private ScanResult scanPackageNewLI(@NonNull PackageParser.Package pkg,
            final @ParseFlags int parseFlags, @ScanFlags int scanFlags, long currentTime,
            @Nullable UserHandle user) throws PackageManagerException {

        final String renamedPkgName = mSettings.getRenamedPackageLPr(pkg.mRealPackage);
        final String realPkgName = getRealPackageName(pkg, renamedPkgName);
        if (realPkgName != null) {
            ensurePackageRenamed(pkg, renamedPkgName);
        }
        final PackageSetting originalPkgSetting = getOriginalPackageLocked(pkg, renamedPkgName);
        final PackageSetting pkgSetting = mSettings.getPackageLPr(pkg.packageName);
        final PackageSetting disabledPkgSetting =
                mSettings.getDisabledSystemPkgLPr(pkg.packageName);

        if (mTransferedPackages.contains(pkg.packageName)) {
            Slog.w(TAG, "Package " + pkg.packageName
                    + " was transferred to another, but its .apk remains");
        }

        scanFlags = adjustScanFlags(scanFlags, pkgSetting, disabledPkgSetting, user, pkg);
        synchronized (mPackages) {
            applyPolicy(pkg, parseFlags, scanFlags, mPlatformPackage);
            assertPackageIsValid(pkg, parseFlags, scanFlags);

            SharedUserSetting sharedUserSetting = null;
            if (pkg.mSharedUserId != null) {
                // SIDE EFFECTS; may potentially allocate a new shared user
                sharedUserSetting = mSettings.getSharedUserLPw(
                        pkg.mSharedUserId, 0 /*pkgFlags*/, 0 /*pkgPrivateFlags*/, true /*create*/);
                if (DEBUG_PACKAGE_SCANNING) {
                    if ((parseFlags & PackageParser.PARSE_CHATTY) != 0)
                        Log.d(TAG, "Shared UserID " + pkg.mSharedUserId
                                + " (uid=" + sharedUserSetting.userId + "):"
                                + " packages=" + sharedUserSetting.packages);
                }
            }
            final ScanRequest request = new ScanRequest(pkg, sharedUserSetting,
                    pkgSetting == null ? null : pkgSetting.pkg, pkgSetting, disabledPkgSetting,
                    originalPkgSetting, realPkgName, parseFlags, scanFlags,
                    (pkg == mPlatformPackage), user);
            return scanPackageOnlyLI(request, mFactoryTest, currentTime);
        }
    }
```
这里有解释一下, `renamedPackageName`和`orignalPackage`其实是相关联的，都是由应用中的`AndroidManifest`解析`original-package`这个TAG得到，
这个TAG通常用于新apk对于老apk并且包名改变更新的时候用，常用于GMS包，例如:GooglePackageInstaller, DocumentUI等一些原本在AOSP后被移植到GMS中的应用，
通常看做为空即可。
`applyPolicy()`是对于`Package`中的一些属性进行更新，`assertPackageIsValid()`是验证Package是否合法。
注意`final PackageSetting pkgSetting = mSettings.getPackageLPr(pkg.packageName);`这里面packageSetting是获取的是存储的，
如果是New install的应用这个packageSetting为空

### PackageManagerService.scanPackageOnlyLI()

这个`ScanRequest`作为参数还是好评，比`PackageParser`中超多的参数要好的多
```java
    private static @NonNull ScanResult scanPackageOnlyLI(@NonNull ScanRequest request,
            boolean isUnderFactoryTest, long currentTime)
                    throws PackageManagerException {
        final PackageParser.Package pkg = request.pkg;
        PackageSetting pkgSetting = request.pkgSetting;
        final PackageSetting disabledPkgSetting = request.disabledPkgSetting;
        final PackageSetting originalPkgSetting = request.originalPkgSetting;
        final @ParseFlags int parseFlags = request.parseFlags;
        final @ScanFlags int scanFlags = request.scanFlags;
        final String realPkgName = request.realPkgName;
        final SharedUserSetting sharedUserSetting = request.sharedUserSetting;
        final UserHandle user = request.user;
        final boolean isPlatformPackage = request.isPlatformPackage;

        List<String> changedAbiCodePath = null;

        if (DEBUG_PACKAGE_SCANNING) {
            if ((parseFlags & PackageParser.PARSE_CHATTY) != 0)
                Log.d(TAG, "Scanning package " + pkg.packageName);
        }

        final File scanFile = new File(pkg.codePath);
        final File destCodeFile = new File(pkg.applicationInfo.getCodePath());
        final File destResourceFile = new File(pkg.applicationInfo.getResourcePath());

        String primaryCpuAbiFromSettings = null;
        String secondaryCpuAbiFromSettings = null;
        boolean needToDeriveAbi = (scanFlags & SCAN_FIRST_BOOT_OR_UPGRADE) != 0;
        if (!needToDeriveAbi) {
            if (pkgSetting != null) {
                primaryCpuAbiFromSettings = pkgSetting.primaryCpuAbiString;
                secondaryCpuAbiFromSettings = pkgSetting.secondaryCpuAbiString;
            } else {
                // Re-scanning a system package after uninstalling updates; need to derive ABI
                needToDeriveAbi = true;
            }
        }

        if (pkgSetting != null && pkgSetting.sharedUser != sharedUserSetting) {
            PackageManagerService.reportSettingsProblem(Log.WARN,
                    "Package " + pkg.packageName + " shared user changed from "
                            + (pkgSetting.sharedUser != null
                            ? pkgSetting.sharedUser.name : "<nothing>")
                            + " to "
                            + (sharedUserSetting != null ? sharedUserSetting.name : "<nothing>")
                            + "; replacing with new");
            pkgSetting = null;
        }

        String[] usesStaticLibraries = null;
        if (pkg.usesStaticLibraries != null) {
            usesStaticLibraries = new String[pkg.usesStaticLibraries.size()];
            pkg.usesStaticLibraries.toArray(usesStaticLibraries);
        }
        final boolean createNewPackage = (pkgSetting == null);
        if (createNewPackage) {
            final String parentPackageName = (pkg.parentPackage != null)
                    ? pkg.parentPackage.packageName : null;
            final boolean instantApp = (scanFlags & SCAN_AS_INSTANT_APP) != 0;
            final boolean virtualPreload = (scanFlags & SCAN_AS_VIRTUAL_PRELOAD) != 0;
            // REMOVE SharedUserSetting from method; update in a separate call
            pkgSetting = Settings.createNewSetting(pkg.packageName, originalPkgSetting,
                    disabledPkgSetting, realPkgName, sharedUserSetting, destCodeFile,
                    destResourceFile, pkg.applicationInfo.nativeLibraryRootDir,
                    pkg.applicationInfo.primaryCpuAbi, pkg.applicationInfo.secondaryCpuAbi,
                    pkg.mVersionCode, pkg.applicationInfo.flags, pkg.applicationInfo.privateFlags,
                    user, true /*allowInstall*/, instantApp, virtualPreload,
                    parentPackageName, pkg.getChildPackageNames(),
                    UserManagerService.getInstance(), usesStaticLibraries,
                    pkg.usesStaticLibrariesVersions);
        } else {
            // make a deep copy to avoid modifying any existing system state.
            pkgSetting = new PackageSetting(pkgSetting);
            pkgSetting.pkg = pkg;

            Settings.updatePackageSetting(pkgSetting, disabledPkgSetting, sharedUserSetting,
                    destCodeFile, destResourceFile, pkg.applicationInfo.nativeLibraryDir,
                    pkg.applicationInfo.primaryCpuAbi, pkg.applicationInfo.secondaryCpuAbi,
                    pkg.applicationInfo.flags, pkg.applicationInfo.privateFlags,
                    pkg.getChildPackageNames(), UserManagerService.getInstance(),
                    usesStaticLibraries, pkg.usesStaticLibrariesVersions);
        }
        if (createNewPackage && originalPkgSetting != null) {
            pkg.setPackageName(originalPkgSetting.name);

            // File a report about this.
            String msg = "New package " + pkgSetting.realName
                    + " renamed to replace old package " + pkgSetting.name;
            reportSettingsProblem(Log.WARN, msg);
        }

        final int userId = (user == null ? UserHandle.USER_SYSTEM : user.getIdentifier());
        // for existing packages, change the install state; but, only if it's explicitly specified
        if (!createNewPackage) {
            final boolean instantApp = (scanFlags & SCAN_AS_INSTANT_APP) != 0;
            final boolean fullApp = (scanFlags & SCAN_AS_FULL_APP) != 0;
            setInstantAppForUser(pkgSetting, userId, instantApp, fullApp);
        }
        // TODO(patb): see if we can do away with disabled check here.
        if (disabledPkgSetting != null
                || (0 != (scanFlags & SCAN_NEW_INSTALL)
                && pkgSetting != null && pkgSetting.isSystem())) {
            pkg.applicationInfo.flags |= ApplicationInfo.FLAG_UPDATED_SYSTEM_APP;
        }

        final int targetSdkVersion =
            ((sharedUserSetting != null) && (sharedUserSetting.packages.size() != 0)) ?
            sharedUserSetting.seInfoTargetSdkVersion : pkg.applicationInfo.targetSdkVersion;
        // TODO(b/71593002): isPrivileged for sharedUser and appInfo should never be out of sync.
        // They currently can be if the sharedUser apps are signed with the platform key.
        final boolean isPrivileged = (sharedUserSetting != null) ?
            sharedUserSetting.isPrivileged() | pkg.isPrivileged() : pkg.isPrivileged();

        pkg.applicationInfo.seInfo = SELinuxMMAC.getSeInfo(pkg, isPrivileged,
                pkg.applicationInfo.targetSandboxVersion, targetSdkVersion);
        pkg.applicationInfo.seInfoUser = SELinuxUtil.assignSeinfoUser(pkgSetting.readUserState(
                userId == UserHandle.USER_ALL ? UserHandle.USER_SYSTEM : userId));

        pkg.mExtras = pkgSetting;
        pkg.applicationInfo.processName = fixProcessName(
                pkg.applicationInfo.packageName,
                pkg.applicationInfo.processName);

        if (!isPlatformPackage) {
            // Get all of our default paths setup
            pkg.applicationInfo.initForUser(UserHandle.USER_SYSTEM);
        }

        final String cpuAbiOverride = deriveAbiOverride(pkg.cpuAbiOverride, pkgSetting);

        if ((scanFlags & SCAN_NEW_INSTALL) == 0) {
            if (needToDeriveAbi) {
                Trace.traceBegin(TRACE_TAG_PACKAGE_MANAGER, "derivePackageAbi");
                final boolean extractNativeLibs = !pkg.isLibrary();
                derivePackageAbi(pkg, cpuAbiOverride, extractNativeLibs);
                Trace.traceEnd(TRACE_TAG_PACKAGE_MANAGER);

                if (isSystemApp(pkg) && !pkg.isUpdatedSystemApp() &&
                        pkg.applicationInfo.primaryCpuAbi == null) {
                    setBundledAppAbisAndRoots(pkg, pkgSetting);
                    setNativeLibraryPaths(pkg, sAppLib32InstallDir);
                }
            } else {
                pkg.applicationInfo.primaryCpuAbi = primaryCpuAbiFromSettings;
                pkg.applicationInfo.secondaryCpuAbi = secondaryCpuAbiFromSettings;

                setNativeLibraryPaths(pkg, sAppLib32InstallDir);

                if (DEBUG_ABI_SELECTION) {
                    Slog.i(TAG, "Using ABIS and native lib paths from settings : " +
                            pkg.packageName + " " + pkg.applicationInfo.primaryCpuAbi + ", " +
                            pkg.applicationInfo.secondaryCpuAbi);
                }
            }
        } else {
            if ((scanFlags & SCAN_MOVE) != 0) {
                pkg.applicationInfo.primaryCpuAbi = pkgSetting.primaryCpuAbiString;
                pkg.applicationInfo.secondaryCpuAbi = pkgSetting.secondaryCpuAbiString;
            }

            setNativeLibraryPaths(pkg, sAppLib32InstallDir);
        }

        if (isPlatformPackage) {
            pkg.applicationInfo.primaryCpuAbi = VMRuntime.getRuntime().is64Bit() ?
                    Build.SUPPORTED_64_BIT_ABIS[0] : Build.SUPPORTED_32_BIT_ABIS[0];
        }

        if ((scanFlags & SCAN_NO_DEX) == 0 && (scanFlags & SCAN_NEW_INSTALL) != 0) {
            if (cpuAbiOverride == null && pkg.packageName != null) {
                Slog.w(TAG, "Ignoring persisted ABI override " + cpuAbiOverride +
                        " for package " + pkg.packageName);
            }
        }

        pkgSetting.primaryCpuAbiString = pkg.applicationInfo.primaryCpuAbi;
        pkgSetting.secondaryCpuAbiString = pkg.applicationInfo.secondaryCpuAbi;
        pkgSetting.cpuAbiOverrideString = cpuAbiOverride;

        // Copy the derived override back to the parsed package, so that we can
        // update the package settings accordingly.
        pkg.cpuAbiOverride = cpuAbiOverride;

        if (DEBUG_ABI_SELECTION) {
            Slog.d(TAG, "Resolved nativeLibraryRoot for " + pkg.packageName
                    + " to root=" + pkg.applicationInfo.nativeLibraryRootDir + ", isa="
                    + pkg.applicationInfo.nativeLibraryRootRequiresIsa);
        }

        // Push the derived path down into PackageSettings so we know what to
        // clean up at uninstall time.
        pkgSetting.legacyNativeLibraryPathString = pkg.applicationInfo.nativeLibraryRootDir;

        if (DEBUG_ABI_SELECTION) {
            Log.d(TAG, "Abis for package[" + pkg.packageName + "] are" +
                    " primary=" + pkg.applicationInfo.primaryCpuAbi +
                    " secondary=" + pkg.applicationInfo.secondaryCpuAbi);
        }

        if ((scanFlags & SCAN_BOOTING) == 0 && pkgSetting.sharedUser != null) {
            changedAbiCodePath =
                    adjustCpuAbisForSharedUserLPw(pkgSetting.sharedUser.packages, pkg);
        }

        if (isUnderFactoryTest && pkg.requestedPermissions.contains(
                android.Manifest.permission.FACTORY_TEST)) {
            pkg.applicationInfo.flags |= ApplicationInfo.FLAG_FACTORY_TEST;
        }

        if (isSystemApp(pkg)) {
            pkgSetting.isOrphaned = true;
        }

        // Take care of first install / last update times.
        final long scanFileTime = getLastModifiedTime(pkg);
        if (currentTime != 0) {
            if (pkgSetting.firstInstallTime == 0) {
                pkgSetting.firstInstallTime = pkgSetting.lastUpdateTime = currentTime;
            } else if ((scanFlags & SCAN_UPDATE_TIME) != 0) {
                pkgSetting.lastUpdateTime = currentTime;
            }
        } else if (pkgSetting.firstInstallTime == 0) {
            // We need *something*.  Take time time stamp of the file.
            pkgSetting.firstInstallTime = pkgSetting.lastUpdateTime = scanFileTime;
        } else if ((parseFlags & PackageParser.PARSE_IS_SYSTEM_DIR) != 0) {
            if (scanFileTime != pkgSetting.timeStamp) {
                // A package on the system image has changed; consider this
                // to be an update.
                pkgSetting.lastUpdateTime = scanFileTime;
            }
        }
        pkgSetting.setTimeStamp(scanFileTime);

        pkgSetting.pkg = pkg;
        pkgSetting.pkgFlags = pkg.applicationInfo.flags;
        if (pkg.getLongVersionCode() != pkgSetting.versionCode) {
            pkgSetting.versionCode = pkg.getLongVersionCode();
        }
        // Update volume if needed
        final String volumeUuid = pkg.applicationInfo.volumeUuid;
        if (!Objects.equals(volumeUuid, pkgSetting.volumeUuid)) {
            Slog.i(PackageManagerService.TAG,
                    "Update" + (pkgSetting.isSystem() ? " system" : "")
                    + " package " + pkg.packageName
                    + " volume from " + pkgSetting.volumeUuid
                    + " to " + volumeUuid);
            pkgSetting.volumeUuid = volumeUuid;
        }

        SharedLibraryInfo staticSharedLibraryInfo = null;
        if (!TextUtils.isEmpty(pkg.staticSharedLibName)) {
            staticSharedLibraryInfo = SharedLibraryInfo.createForStatic(pkg);
        }
        List<SharedLibraryInfo> dynamicSharedLibraryInfos = null;
        if (!ArrayUtils.isEmpty(pkg.libraryNames)) {
            dynamicSharedLibraryInfos = new ArrayList<>(pkg.libraryNames.size());
            for (String name : pkg.libraryNames) {
                dynamicSharedLibraryInfos.add(SharedLibraryInfo.createForDynamic(pkg, name));
            }
        }

        return new ScanResult(request, true, pkgSetting, changedAbiCodePath,
                !createNewPackage /* existingSettingCopied */, staticSharedLibraryInfo,
                dynamicSharedLibraryInfos);
    }
```
又是一个超级长的方法, 分析这个方法的时候谨记两点：
1. `pkgSetting()`是存储在packages.list中的，所以New install的时候为空
2. `scanPackageOnlyLI()`是开机和安装都会调用,要看清楚启动的区别

方法的主要流程中有三点
1. 处理cpuAbi，即32位和64位的架构的cpu
2. 处理lib库，包括动态库，静态库以及native库
3. 最主要的功能就是创建或者更新`PackageSetting`，New install以及sharedUser有修改（开机）为创建, 开机以及覆盖安装为更新,`pkg.mExtras = pkgSetting;`,

### 3. PackageManagerService.reconcilePackagesLocked()
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

