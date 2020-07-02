
---
title:Android Q 应用安装流程(三)--PkMS Package准备
date:2020/07/01
category:
  - android
  - framework
---


# PackageManagerService.preparePackageLI()
这个方法实在是太长了，只能简略的先看一下核心的代码了

## 1. 解析APK
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
## 2. 收集签名文件

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
## 3. 重命名文件夹

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

## 4. 冻结APP
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
## 5. 处理覆盖安装的情况

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

## 6. 处理新安装的情况

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
