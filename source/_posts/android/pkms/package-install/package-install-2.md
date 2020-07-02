---
title:Android Q 应用安装流程(一)--前期准备
date:2020/06/15
category:
  - android
  - framework
---
# 前期准备
对于`PackageInstallerSession`传递的Session进行处理，最终生成`FileInstallArgs`和`PackageInstalledInfo`, 又由这两个参数生成`InstallRequest`，
最后调用`installPackagesTracedLI()`方法进行最终的安装工作

# PackageManagerService.installStage()

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

# PackageManagerService.PackageHandler.doHandleMessage()

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

# PackageManagerService.HandlerParams.startCopy()

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


# 前言
Package install的流程是相当的长，分析的时候可能遇到的困难：
1. 涉及的参数类较多,PackageSetting, PackageParser,  FileInstallArgs, ScanRequest等等
2. Split packages, instantApp以及child packages处理比较复杂，要理清这些东西的目的和原理是什么
3. 对于共享库的处理，例如静态lib库，


