---
title: pkms中部分数据结构LOG
date: 2020/03/26
categories: '-android -framework'
tags: '-PackageMangerService'
abbrlink: da7adbcf
---
# ScanFlags track log
```java
public static void trackScanFlags(int scanFlags, String pkgName) {
    String log = "";
    String caller = new Exception().getStackTrace()[1].getMethodName();
    log += "caller {" + caller + "} {" + pkgName + "} ";
    if (0 != (scanFlags & SCAN_NO_DEX)) log += " SCAN_NO_DEX ";
    if (0 != (scanFlags & SCAN_UPDATE_SIGNATURE)) log += " SCAN_UPDATE_SIGNATURE ";
    if (0 != (scanFlags & SCAN_NEW_INSTALL)) log += " SCAN_NEW_INSTALL ";
    if (0 != (scanFlags & SCAN_UPDATE_TIME)) log += " SCAN_UPDATE_TIME ";
    if (0 != (scanFlags & SCAN_BOOTING)) log += " SCAN_BOOTING ";
    if (0 != (scanFlags & SCAN_REQUIRE_KNOWN)) log += " SCAN_REQUIRE_KNOWN ";
    if (0 != (scanFlags & SCAN_MOVE)) log += " SCAN_MOVE ";
    if (0 != (scanFlags & SCAN_INITIAL)) log += " SCAN_INITIAL ";
    if (0 != (scanFlags & SCAN_CHECK_ONLY)) log += " SCAN_CHECK_ONLY ";
    if (0 != (scanFlags & SCAN_DONT_KILL_APP)) log += " SCAN_DONT_KILL_APP ";
    if (0 != (scanFlags & SCAN_IGNORE_FROZEN)) log += " SCAN_IGNORE_FROZEN ";
    if (0 != (scanFlags & SCAN_FIRST_BOOT_OR_UPGRADE)) log += " SCAN_FIRST_BOOT_OR_UPGRADE ";
    if (0 != (scanFlags & SCAN_AS_INSTANT_APP)) log += " SCAN_AS_INSTANT_APP ";
    if (0 != (scanFlags & SCAN_AS_FULL_APP)) log += " SCAN_AS_FULL_APP ";
    if (0 != (scanFlags & SCAN_AS_VIRTUAL_PRELOAD)) log += " SCAN_AS_VIRTUAL_PRELOAD ";
    Log.d("jonny", "trackScanFlags " + log + "");
}
```
# ParseFlags track log
```java
public static void trackParseFlags(int parseFlags, String pkgName) {
    String log = "";
    String caller = new Exception().getStackTrace()[1].getMethodName();
    log += "caller {" + caller + "} {" + pkgName + "} ";
    if (0 != (parseFlags & PARSE_CHATTY)) {log += "PARSE_CHATTY | ";}
    if (0 != (parseFlags & PARSE_COLLECT_CERTIFICATES)) { log += "PARSE_COLLECT_CERTIFICATES | "; }
    if (0 != (parseFlags & PARSE_ENFORCE_CODE)) { log += "PARSE_ENFORCE_CODE | "; }
    if (0 != (parseFlags & PARSE_EXTERNAL_STORAGE)) { log += "PARSE_EXTERNAL_STORAGE | "; }
    if (0 != (parseFlags & PARSE_IGNORE_PROCESSES)) { log += "PARSE_IGNORE_PROCESSES | "; }
    if (0 != (parseFlags & PARSE_IS_SYSTEM_DIR)) { log += "PARSE_IS_SYSTEM_DIR | "; }
    if (0 != (parseFlags & PARSE_MUST_BE_APK)) { log += "PARSE_MUST_BE_APK | "; }
    Log.d("jonny", "trackParseFlags " + log + "");
}
```
# ApkLite structure log
```java
@Override
public String toString() {
    StringBuffer buffer = new StringBuffer(200);
    buffer.append("ApkLite{");
    buffer.append("codePath=").append(codePath).append(", ");
    buffer.append("packageName=").append(packageName).append(", ");
    buffer.append("splitName=").append(splitName).append(", ");
    buffer.append("isFeatureSplit=").append(isFeatureSplit).append(", ");
    buffer.append("configForSplit=").append(configForSplit).append(", ");
    buffer.append("usesSplitName=").append(usesSplitName).append(", ");
    buffer.append("versionCode=").append(versionCode).append(", ");
    buffer.append("versionCodeMajor=").append(versionCodeMajor).append(", ");
    buffer.append("revisionCode=").append(revisionCode).append(", ");
    buffer.append("installLocation=").append(installLocation).append(", ");
    buffer.append("minSdkVersion=").append(minSdkVersion).append(", ");
    buffer.append("targetSdkVersion=").append(targetSdkVersion).append(", ");
    buffer.append("verifiers size=").append(verifiers.size()).append(", ");
    buffer.append("signingDetails=").append(signingDetails).append(", ");
    buffer.append("coreApp=").append(coreApp).append(", ");
    buffer.append("debuggable=").append(debuggable).append(", ");
    buffer.append("multiArch=").append(multiArch).append(", ");
    buffer.append("use32bitAbi=").append(use32bitAbi).append(", ");
    buffer.append("extractNativeLibs=").append(extractNativeLibs).append(", ");
    buffer.append("isolatedSplits=").append(isolatedSplits).append(", ");
    buffer.append("isSplitRequired=").append(isSplitRequired).append(", ");
    buffer.append("useEmbeddedDex=").append(useEmbeddedDex).append(", ");
    buffer.append("}");
    return buffer.toString();
}

```
# PackageLite structure log
```java
@Override
public String toString() {
    StringBuffer buffer = new StringBuffer(200);
    buffer.append("ApkLite{");
    buffer.append("packageName=").append(packageName).append(", ");
    buffer.append("versionCode=").append(versionCode).append(", ");
    buffer.append("versionCodeMajor=").append(versionCodeMajor).append(", ");
    buffer.append("installLocation=").append(installLocation).append(", ");
    buffer.append("verifiers.length=").append(verifiers.length).append(", ");
    buffer.append("splitNames=[").append(Arrays.toString(splitNames)).append(", ");
    buffer.append("isFeatureSplits=").append(Arrays.toString(isFeatureSplits)).append(", ");
    buffer.append("usesSplitNames=[").append(Arrays.toString(usesSplitNames)).append("], ");
    buffer.append("configForSplit=[").append(Arrays.toString(configForSplit)).append("], ");
    buffer.append("codePath=").append(codePath).append(", ");
    buffer.append("baseCodePath=").append(baseCodePath).append(", ");
    buffer.append("splitCodePaths=[").append(splitCodePaths).append("], ");
    buffer.append("baseRevisionCode=").append(baseRevisionCode).append(", ");
    buffer.append("splitRevisionCodes=[").append(splitRevisionCodes).append("], ");
    buffer.append("coreApp=").append(coreApp).append(", ");
    buffer.append("debuggable=").append(debuggable).append(", ");
    buffer.append("multiArch=").append(multiArch).append(", ");
    buffer.append("use32bitAbi=").append(use32bitAbi).append(", ");
    buffer.append("extractNativeLibs=").append(extractNativeLibs).append(", ");
    buffer.append("isolatedSplits=").append(isolatedSplits).append(", ");
    buffer.append("}");
    return buffer.toString();
}

```
# Package structure log
```java
@Override
public String toString() {
    StringBuffer buffer = new StringBuffer(500);
    buffer.append("Package {");
    buffer.append("packageName=").append(packageName).append(", ");
    buffer.append("manifestPackageName=").append(manifestPackageName).append(", ");
    buffer.append("splitNames=[").append(Arrays.toString(splitNames)).append("], ");
    buffer.append("volumeUuid=").append(volumeUuid).append(", ");
    buffer.append("codePath=").append(codePath).append(", ");
    buffer.append("baseCodePath=").append(baseCodePath).append(", ");
    buffer.append("splitCodePaths=[").append(Arrays.toString(splitCodePaths)).append("], ");
    buffer.append("baseRevisionCode=").append(baseRevisionCode).append(", ");
    buffer.append("splitRevisionCodes=[").append(Arrays.toString(splitRevisionCodes)).append("], ");
    buffer.append("splitFlags=[").append(Arrays.toString(splitFlags)).append("], ");
    buffer.append("splitPrivateFlags=[").append(splitPrivateFlags).append("], ");
    buffer.append("baseHardwareAccelerated=").append(baseHardwareAccelerated).append(", ");
    buffer.append("applicationInfo={").append(applicationInfo).append("}, ");
    buffer.append("permissions=").append(permissions).append(", ");
    buffer.append("permissionGroups=").append(permissionGroups).append(", ");
    buffer.append("activities=").append(activities).append(", ");
    buffer.append("receivers=").append(receivers).append(", ");
    buffer.append("providers=").append(providers).append(", ");
    buffer.append("services=").append(services).append(", ");
    buffer.append("instrumentation=").append(instrumentation).append(", ");
    buffer.append("requestedPermissions=").append(requestedPermissions).append(", ");
    buffer.append("implicitPermissions=").append(implicitPermissions).append(", ");
    buffer.append("protectedBroadcasts=").append(protectedBroadcasts).append(", ");
    buffer.append("parentPackage={").append(parentPackage).append("}, ");
    buffer.append("childPackages=").append(childPackages).append(", ");
    buffer.append("staticSharedLibName=").append(staticSharedLibName).append(", ");
    buffer.append("staticSharedLibVersion=").append(staticSharedLibVersion).append(", ");
    buffer.append("libraryNames=").append(libraryNames).append(", ");
    buffer.append("usesLibraries=").append(usesLibraries).append(", ");
    buffer.append("usesStaticLibraries=").append(usesStaticLibraries).append(", ");
    buffer.append("usesStaticLibrariesVersions=").append(Arrays.toString(usesStaticLibrariesVersions)).append(", ");
    buffer.append("usesStaticLibrariesCertDigests=").append(Arrays.toString(usesStaticLibrariesCertDigests)).append(", ");
    buffer.append("usesOptionalLibraries=").append(usesOptionalLibraries).append(", ");
    buffer.append("usesLibraryFiles=").append(Arrays.toString(usesLibraryFiles)).append(", ");
    buffer.append("usesLibraryInfos=").append(usesLibraryInfos).append(", ");
    buffer.append("preferredActivityFilters=").append(preferredActivityFilters).append(", ");
    buffer.append("mOriginalPackages=").append(mOriginalPackages).append(", ");
    buffer.append("mRealPackage=").append(mRealPackage).append(", ");
    buffer.append("mAdoptPermissions=").append(mAdoptPermissions).append(", ");
    buffer.append("mAppMetaData=").append(mAppMetaData).append(", ");
    buffer.append("mVersionCode=").append(mVersionCode).append(", ");
    buffer.append("mVersionCodeMajor=").append(mVersionCodeMajor).append(", ");
    buffer.append("mVersionName=").append(mVersionName).append(", ");
    buffer.append("mSharedUserId=").append(mSharedUserId).append(", ");
    buffer.append("mSharedUserLabel=").append(mSharedUserLabel).append(", ");
    buffer.append("mSigningDetails=").append(mSigningDetails).append(", ");
    buffer.append("mPreferredOrder=").append(mPreferredOrder).append(", ");
    buffer.append("mLastPackageUsageTimeInMills=").append(Arrays.toString(mLastPackageUsageTimeInMills)).append(", ");
    buffer.append("mExtras=").append(mExtras).append(", ");
    buffer.append("configPreferences=").append(configPreferences).append(", ");
    buffer.append("reqFeatures=").append(reqFeatures).append(", ");
    buffer.append("featureGroups=").append(featureGroups).append(", ");
    buffer.append("installLocation=").append(installLocation).append(", ");
    buffer.append("coreApp=").append(coreApp).append(", ");
    buffer.append("mRequiredForAllUsers=").append(mRequiredForAllUsers).append(", ");
    buffer.append("mRestrictedAccountType=").append(mRestrictedAccountType).append(", ");
    buffer.append("mRequiredAccountType=").append(mRequiredAccountType).append(", ");
    buffer.append("mOverlayTarget=").append(mOverlayTarget).append(", ");
    buffer.append("mOverlayTargetName=").append(mOverlayTargetName).append(", ");
    buffer.append("mOverlayCategory=").append(mOverlayCategory).append(", ");
    buffer.append("mOverlayPriority=").append(mOverlayPriority).append(", ");
    buffer.append("mOverlayIsStatic=").append(mOverlayIsStatic).append(", ");
    buffer.append("mCompileSdkVersion=").append(mCompileSdkVersion).append(", ");
    buffer.append("mCompileSdkVersionCodename=").append(mCompileSdkVersionCodename).append(", ");
    buffer.append("mUpgradeKeySets=").append(mUpgradeKeySets).append(", ");
    buffer.append("mKeySetMapping=").append(mKeySetMapping).append(", ");
    buffer.append("cpuAbiOverride=").append(cpuAbiOverride).append(", ");
    buffer.append("use32bitAbi=").append(use32bitAbi).append(", ");
    buffer.append("restrictUpdateHash=").append(Arrays.toString(restrictUpdateHash)).append(", ");
    buffer.append("visibleToInstantApps=").append(visibleToInstantApps).append(", ");
    buffer.append("isStub=").append(isStub).append(", ");
    buffer.append("}");
    return buffer.toString();
}
```

# PackageSetting log

```java
@Override
public String toString() {
    StringBuffer buffer = new StringBuffer(200);
    buffer.append("PackageSetting {");
    buffer.append("name=").append(name).append(", ");
    buffer.append("realName=").append(realName).append(", ");
    buffer.append("codePath=").append(codePath).append(", ");
    buffer.append("resourcePath=").append(resourcePath).append(", ");
    buffer.append("legacyNativeLibraryPathString=").append(legacyNativeLibraryPathString).append(", ");
    buffer.append("primaryCpuAbiString=").append(primaryCpuAbiString).append(", ");
    buffer.append("secondaryCpuAbiString=").append(secondaryCpuAbiString).append(", ");
    buffer.append("cpuAbiOverrideString=").append(cpuAbiOverrideString).append(", ");
    buffer.append("pVersionCode=").append(pVersionCode).append(", ");
    buffer.append("pkgFlags=").append(pkgFlags).append(", ");
    buffer.append("privateFlags=").append(privateFlags).append(", ");
    buffer.append("parentPackageName=").append(parentPackageName).append(", ");
    buffer.append("childPackageNames=").append(childPackageNames).append(", ");
    buffer.append("sharedUserId=").append(sharedUserId).append(", ");
    buffer.append("usesStaticLibraries=").append(Arrays.toString(usesStaticLibraries)).append(", ");
    buffer.append("usesStaticLibrariesVersions=").append(usesStaticLibrariesVersions).append(", ");
    buffer.append("}");
    return buffer.toString();

```

# SharedUserSetting log
```java
@Override
public String toString() {
    StringBuffer buffer = new StringBuffer(100);
    buffer.append("SharedUserSetting {");
    buffer.append("name=").append(name).append(", ");
    buffer.append("uidFlags=").append(uidFlags).append(", ");
    buffer.append("uidPrivateFlags=").append(uidPrivateFlags).append(", ");
    buffer.append("seInfoTargetSdkVersion=").append(seInfoTargetSdkVersion).append(", ");
    buffer.append("packages=").append(packages).append(", ");
    buffer.append("signatures=").append(signatures).append(", ");
    buffer.append("signaturesChanged=").append(signaturesChanged).append(", ");
    buffer.append("}");
    return buffer.toString();
}

```

#Installer dexopt method log
```java
String params = new String("");
params += "apkPath=" + apkPath + ", ";
params += "uid=" + uid + ", ";
params += "pkgName=" + pkgName + ", ";
params += "instructionSet=" + instructionSet + ", ";
params += "dexoptNeeded=" + dexoptNeeded + ", ";
params += "outputPath=" + outputPath + ", ";
params += "dexFlags=" + Integer.toBinaryString(dexFlags) + ", ";
params += "compilerFilter=" + compilerFilter + ", ";
params += "volumeUuid=" + volumeUuid + ", ";
params += "sharedLibraries=" + sharedLibraries + ", ";
params += "seInfo=" + seInfo + ", ";
params += "downgrade=" + downgrade + ", ";
params += "targetSdkVersion=" + targetSdkVersion + ", ";
params += "profileName=" + profileName + ", ";
params += "dexMetadataPath=" + dexMetadataPath + ", ";
params += "compilationReason=" + compilationReason + ", ";
Log.d("dexopt", "params { " + params + " }");

```
