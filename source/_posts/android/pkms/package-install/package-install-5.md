
---
title:Android Q 应用安装流程(四)--安装扫描
date:2020/07/01
category:
  - android
  - framework
---
# PackageManagerService.scanPackageTracedLI()

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

## PackageManagerService.scanPackageNewLI()

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

## PackageManagerService.scanPackageOnlyLI()

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

