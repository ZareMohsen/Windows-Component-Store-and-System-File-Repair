# Windows Component Store and System File Repair

A practical PowerShell reference for diagnosing and repairing Windows
component-store and protected system-file corruption with DISM and SFC.

> **Scope:** Windows 10 and Windows 11 client editions. Most commands
> also apply to Windows Server, but source-media and servicing details
> can differ.
>
> **Important:** Run repair commands from **Windows PowerShell or
> PowerShell as Administrator**. Replace example drive letters, paths,
> indexes, and proxy addresses with values from your own system.

------------------------------------------------------------------------

## 1. Before you begin

### 1.1 Open an elevated PowerShell window

1.  Open Start.
2.  Search for **PowerShell** or **Terminal**.
3.  Select **Run as administrator**.
4.  Confirm the window title or prompt indicates an elevated session.

Check the current account and elevation:

``` powershell
# Display the current Windows identity.
whoami

# Check whether this PowerShell process is running with administrative privileges.
$principal = New-Object Security.Principal.WindowsPrincipal(
    [Security.Principal.WindowsIdentity]::GetCurrent()
)
$principal.IsInRole([Security.Principal.WindowsBuiltInRole]::Administrator)
```

The final command should return `True`.

### 1.2 Record Windows version, edition, language, and architecture

``` powershell
# Display the Windows product and servicing build information.
Get-ItemProperty 'HKLM:\SOFTWARE\Microsoft\Windows NT\CurrentVersion' |
    Select-Object ProductName, EditionID, DisplayVersion, CurrentBuild, UBR, InstallationType

# Display the operating-system architecture.
[System.Runtime.InteropServices.RuntimeInformation]::OSArchitecture

# Display the installed UI language.
Get-WinSystemLocale
```

**Do not rely on `ProductName` alone.** On some systems it may show an
older product label. `CurrentBuild`, `UBR`, `EditionID`, and the actual
installed image are more useful when identifying a repair source.

### 1.3 Recommended precautions

-   Back up important files before servicing operations.
-   Connect the computer to reliable power.
-   Do not interrupt DISM while it is running.
-   Make sure the Windows Update / repair-source network path is
    reachable if using online repair.
-   Use installation media that matches the installed Windows release as
    closely as possible. Prefer the same architecture, language,
    edition, and servicing/build level.
-   Do not use an older Windows 10 image to repair Windows 11, or an
    unrelated Windows 11 build as a repair source.
-   If a command reports an error, save the complete output and inspect
    the logs before trying a different source.

------------------------------------------------------------------------

## 2. Proxy configuration

Windows has more than one proxy configuration mechanism. These settings
are **not interchangeable**.

  -----------------------------------------------------------------------
  Proxy type              What it affects         Typical lifetime
  ----------------------- ----------------------- -----------------------
  PowerShell environment  Programs that read      Current PowerShell
  variables               proxy environment       process and child
                          variables, including    processes
                          many command-line tools 

  WinHTTP proxy           Windows components and  System-wide, persistent
                          applications that use   until changed
                          the WinHTTP stack       

  WinINET / user proxy    Many user-session       User profile;
  settings                applications and        configured separately
                          Windows settings        
  -----------------------------------------------------------------------

A successful browser connection does not prove that WinHTTP or a
command-line application can connect.

### 2.1 Set proxy variables for the current PowerShell session only

Example for a local HTTP proxy listening on `127.0.0.1:10808`:

``` powershell
# Set proxy environment variables for this PowerShell process and its child processes.
$env:HTTP_PROXY  = 'http://127.0.0.1:10808'
$env:HTTPS_PROXY = 'http://127.0.0.1:10808'
$env:ALL_PROXY   = 'socks5h://127.0.0.1:10808'

# Set lowercase variants for tools that only check lowercase variable names.
$env:http_proxy  = $env:HTTP_PROXY
$env:https_proxy = $env:HTTPS_PROXY
$env:all_proxy   = $env:ALL_PROXY

# Display the proxy variables currently defined in this PowerShell session.
Get-ChildItem Env: |
    Where-Object Name -Match '^(HTTP_PROXY|HTTPS_PROXY|ALL_PROXY|http_proxy|https_proxy|all_proxy)$'
```

Notes:

-   Use the proxy URL scheme supported by your local proxy listener. For
    an HTTP proxy, use `http://...`; for a SOCKS proxy, use the
    appropriate `socks5://` or `socks5h://` scheme.
-   `socks5h` asks the proxy to resolve hostnames remotely. Not every
    application supports SOCKS proxy variables.
-   Do not assume that DISM or Windows Update honors these environment
    variables. WinHTTP configuration may be required for components that
    use WinHTTP.
-   These variables disappear when the PowerShell process closes. They
    do not permanently configure Windows.

Remove the proxy variables from the current PowerShell session:

``` powershell
# Remove proxy environment variables from this PowerShell process.
'HTTP_PROXY', 'HTTPS_PROXY', 'ALL_PROXY',
'http_proxy', 'https_proxy', 'all_proxy' |
    ForEach-Object {
        Remove-Item -Path "Env:$_" -ErrorAction SilentlyContinue
    }

# Confirm that the listed variables are no longer defined in this session.
Get-ChildItem Env: |
    Where-Object Name -Match '^(HTTP_PROXY|HTTPS_PROXY|ALL_PROXY|http_proxy|https_proxy|all_proxy)$'
```

### 2.2 Test a local proxy listener

``` powershell
# Check whether a process is listening on local TCP port 10808.
Get-NetTCPConnection -State Listen -LocalPort 10808 -ErrorAction SilentlyContinue |
    Select-Object LocalAddress, LocalPort, State, OwningProcess

# Show the executable associated with the listener process, if one exists.
$listener = Get-NetTCPConnection -State Listen -LocalPort 10808 -ErrorAction SilentlyContinue |
    Select-Object -First 1

if ($listener) {
    Get-Process -Id $listener.OwningProcess |
        Select-Object Id, ProcessName, Path
}
```

Test an HTTP proxy endpoint with `curl.exe`:

``` powershell
# Send an HTTP HEAD request through the local HTTP proxy.
curl.exe -I --proxy http://127.0.0.1:10808 https://www.microsoft.com
```

A successful HTTP response indicates that this test request reached the
destination through the configured proxy. It does not prove that every
Windows component uses the same proxy.

### 2.3 Configure the WinHTTP proxy

Use this when a Windows component that relies on WinHTTP needs the local
proxy.

``` powershell
# Configure persistent WinHTTP proxy settings for HTTP and HTTPS traffic.
netsh winhttp set proxy proxy-server="http=127.0.0.1:10808;https=127.0.0.1:10808"

# Display the current WinHTTP proxy configuration.
netsh winhttp show proxy
```

This example assumes that the listener accepts **HTTP proxy requests**
on port `10808`. Do not put a SOCKS-only listener here unless it also
provides an HTTP proxy interface.

### 2.4 Reset WinHTTP proxy settings

``` powershell
# Remove the configured WinHTTP proxy and return to direct access.
netsh winhttp reset proxy

# Verify that WinHTTP is configured for direct access.
netsh winhttp show proxy
```

Expected result:

``` text
Direct access (no proxy server).
```

Resetting WinHTTP does not remove PowerShell environment variables and
does not necessarily change the current user's browser proxy settings.

### 2.5 Import the current user's proxy settings into WinHTTP

This is an alternative to manually specifying a proxy. Use it only when
the current user's Windows proxy settings are correct and should be
copied to WinHTTP.

``` powershell
# Import the current user's proxy settings into the WinHTTP configuration.
netsh winhttp import proxy source=ie

# Display the resulting WinHTTP proxy configuration.
netsh winhttp show proxy
```

------------------------------------------------------------------------

## 3. DISM: inspect the Windows image

DISM (Deployment Image Servicing and Management) can inspect and repair
the Windows component store.

### 3.1 Check component-store status

``` powershell
# Check whether Windows has detected component-store corruption.
DISM.exe /Online /Cleanup-Image /CheckHealth
```

`CheckHealth` reports whether corruption has already been flagged. It is
a quick check and does not perform a full scan.

### 3.2 Scan the component store

``` powershell
# Perform a more thorough scan for component-store corruption.
DISM.exe /Online /Cleanup-Image /ScanHealth
```

`ScanHealth` performs a scan but does not repair the image.

### 3.3 Repair the online Windows image

``` powershell
# Repair the currently running Windows image using available repair sources.
DISM.exe /Online /Cleanup-Image /RestoreHealth
```

This is the normal first repair command when Windows can obtain the
required repair content from Windows Update or another configured
source.

Important:

-   The command may appear to pause at a percentage for a while. Do not
    terminate it just because progress is slow.
-   If Windows Update cannot provide the required content, DISM may fail
    with a source-related error.
-   A successful DISM repair should normally be followed by
    `sfc /scannow`.

### 3.4 Repair using Windows Update explicitly

In ordinary circumstances, `RestoreHealth` can use Windows Update
automatically. If a managed environment has configured repair-source
policies, behavior may differ. Check organizational policy before
changing it.

------------------------------------------------------------------------

## 4. DISM: repair from Windows installation media

Use installation media when online repair cannot obtain the required
files, or when an administrator wants to specify a known repair source.

### 4.1 Mount the ISO and identify its drive letter

After mounting the ISO in File Explorer, identify the mounted drive:

``` powershell
# List CD/DVD drives and their drive letters.
Get-CimInstance Win32_LogicalDisk -Filter 'DriveType = 5' |
    Select-Object DeviceID, VolumeName, FileSystem
```

For the examples below, assume the mounted ISO is drive `I:`. Replace
`I:` with the actual drive letter.

``` powershell
# Verify that the installation media contains a WIM or ESD image.
Test-Path 'I:\sources\install.wim'
Test-Path 'I:\sources\install.esd'

# List the source files if you need to inspect the media.
Get-ChildItem 'I:\sources\install.*'
```

### 4.2 Identify the correct image index

An installation image can contain multiple editions. Never assume that
Pro is always index 6.

For a WIM file:

``` powershell
# List the editions and indexes contained in install.wim.
DISM.exe /Get-WimInfo /WimFile:I:\sources\install.wim
```

For an ESD file:

``` powershell
# List the editions and indexes contained in install.esd.
DISM.exe /Get-WimInfo /WimFile:I:\sources\install.esd
```

Inspect a specific index:

``` powershell
# Replace 6 with the index you want to inspect.
DISM.exe /Get-WimInfo /WimFile:I:\sources\install.wim /Index:6
```

For ESD media:

``` powershell
# Replace 6 with the index you want to inspect.
DISM.exe /Get-WimInfo /WimFile:I:\sources\install.esd /Index:6
```

Confirm that the selected index matches the installed edition and that
the source is compatible with the installed Windows release,
architecture, language, and servicing level.

### 4.3 Repair from a WIM source

``` powershell
# Replace I: with the mounted media drive and 6 with the verified image index.
# /LimitAccess prevents DISM from contacting Windows Update.
DISM.exe /Online /Cleanup-Image /RestoreHealth /Source:wim:I:\sources\install.wim:6 /LimitAccess
```

### 4.4 Repair from an ESD source

``` powershell
# Replace I: with the mounted media drive and 6 with the verified image index.
# /LimitAccess prevents DISM from contacting Windows Update.
DISM.exe /Online /Cleanup-Image /RestoreHealth /Source:esd:I:\sources\install.esd:6 /LimitAccess
```

### 4.5 Repair from a mounted Windows image directory

If a compatible Windows image has already been mounted and its component
store is available, a directory source may be used in suitable servicing
scenarios. The source must contain the required repair content.

``` powershell
# Example only: use a valid, compatible Windows source directory.
DISM.exe /Online /Cleanup-Image /RestoreHealth /Source:C:\RepairSource\Windows /LimitAccess
```

Do not use an arbitrary `Windows` folder as a source.

### 4.6 Common source-media problems

A source may be rejected if it is:

-   From a different Windows release or build.
-   From a different architecture (for example, ARM64 instead of x64).
-   In a different language.
-   From a different edition or missing the required edition components.
-   Older than the installed servicing level and missing the required
    component versions.
-   Incomplete, damaged, or missing required files.

A source-related failure such as `0x800f081f` or `0x800f0915` does not
mean that you should try random ISO files. Verify the media, image
index, language, edition, architecture, and build first.

------------------------------------------------------------------------

## 5. DISM: offline repair

Offline servicing targets a Windows installation that is **not currently
running**. Examples include a Windows installation on another partition
or a Windows installation accessed from Windows Recovery Environment
(WinRE).

Do not use `/Online` for an offline installation.

### 5.1 Identify the offline Windows partition

Drive letters can change in WinRE. Do not assume that the Windows
partition is `C:`.

``` powershell
# List volumes so you can identify the Windows partition and installation media.
Get-Volume |
    Select-Object DriveLetter, FileSystemLabel, FileSystem, Size, SizeRemaining

# Check likely Windows directories. Replace D: with a candidate drive letter.
Test-Path 'D:\Windows'
Test-Path 'D:\Windows\System32\Config\SOFTWARE'
```

For the examples below:

-   `D:\Windows` is the offline Windows directory.
-   `I:` is the installation media.
-   `6` is an example image index only; verify it before use.

### 5.2 Check the offline image

``` powershell
# Check the health state of the offline Windows image.
DISM.exe /Image:D:\ /Cleanup-Image /CheckHealth
```

### 5.3 Scan the offline image

``` powershell
# Scan the offline Windows component store for corruption.
DISM.exe /Image:D:\ /Cleanup-Image /ScanHealth
```

### 5.4 Repair the offline image using Windows Update or configured sources

``` powershell
# Attempt to repair the offline image using available repair sources.
DISM.exe /Image:D:\ /Cleanup-Image /RestoreHealth
```

Offline repair source availability depends on the environment. If a
specific source is required, specify a compatible WIM or ESD source as
shown below.

### 5.5 Repair the offline image from WIM media

``` powershell
# Replace the drive letter and index with values verified from your installation media.
DISM.exe /Image:D:\ /Cleanup-Image /RestoreHealth /Source:wim:I:\sources\install.wim:6 /LimitAccess
```

### 5.6 Repair the offline image from ESD media

``` powershell
# Replace the drive letter and index with values verified from your installation media.
DISM.exe /Image:D:\ /Cleanup-Image /RestoreHealth /Source:esd:I:\sources\install.esd:6 /LimitAccess
```

**Offline servicing caution:** BitLocker may need to be unlocked before
the Windows partition can be serviced. Verify the target drive
carefully; using the wrong `/Image` path can service the wrong
installation.

------------------------------------------------------------------------

## 6. SFC: scan and repair protected system files

SFC (System File Checker) verifies protected Windows system files and
repairs them when possible.

### 6.1 Scan the currently running Windows installation

``` powershell
# Scan protected system files and repair detected integrity violations.
sfc /scannow
```

Recommended order after a successful DISM repair:

1.  Run `DISM.exe /Online /Cleanup-Image /RestoreHealth`.
2.  Run `sfc /scannow`.
3.  Restart Windows if requested or if repaired files require it.
4.  Optionally run `sfc /scannow` again after restarting to confirm the
    result.

### 6.2 Verify system files without repairing them

``` powershell
# Verify protected system files without attempting repairs.
sfc /verifyonly
```

### 6.3 Scan an offline Windows installation

Use the correct offline Windows and boot directories. In this example,
the offline Windows installation is on `D:` and its boot files are on
the same volume.

``` powershell
# Scan and repair the offline Windows installation.
sfc /scannow /offbootdir=D:\ /offwindir=D:\Windows
```

The correct `/offbootdir` can differ depending on the partition layout
and whether the system uses UEFI. Identify the actual boot/system
partition before running offline SFC. Do not copy the example blindly.

------------------------------------------------------------------------

## 7. Live monitoring: watch DISM and CBS logs

DISM and Windows servicing write diagnostic information to log files.

### 7.1 Watch the DISM log in real time

Open a **second elevated PowerShell window** while DISM is running:

``` powershell
# Display the latest DISM log entries and continue following new entries.
Get-Content "$env:windir\Logs\DISM\dism.log" -Tail 30 -Wait
```

Stop live monitoring with `Ctrl+C`. This stops the log viewer, not the
DISM operation running in the other window.

### 7.2 Watch the CBS log in real time

``` powershell
# Display the latest Component-Based Servicing log entries and follow new entries.
Get-Content "$env:windir\Logs\CBS\CBS.log" -Tail 30 -Wait
```

### 7.3 Read the latest log entries without following

``` powershell
# Display the latest 100 lines from the DISM log.
Get-Content "$env:windir\Logs\DISM\dism.log" -Tail 100

# Display the latest 100 lines from the CBS log.
Get-Content "$env:windir\Logs\CBS\CBS.log" -Tail 100
```

### 7.4 Search logs for common error and repair indicators

``` powershell
# Search the DISM log for common error and repair-related terms.
Select-String -Path "$env:windir\Logs\DISM\dism.log" `
    -Pattern 'Error|Failed|0x[0-9A-Fa-f]{8}|RestoreHealth|completed successfully' |
    Select-Object -Last 80

# Search the CBS log for corruption and repair-related terms.
Select-String -Path "$env:windir\Logs\CBS\CBS.log" `
    -Pattern 'Error|Failed|corrupt|repair|0x[0-9A-Fa-f]{8}' |
    Select-Object -Last 100
```

A line containing `Error` is not automatically proof that the final
operation failed. Review the surrounding entries and the final command
result.

### 7.5 Check whether DISM is still running

``` powershell
# Check whether a DISM process is currently running.
Get-Process -Name dism -ErrorAction SilentlyContinue |
    Select-Object Id, ProcessName, CPU, StartTime
```

------------------------------------------------------------------------

## 8. Useful servicing and system commands

### 8.1 Check pending reboot indicators

There is no single universal registry check that proves a reboot is or
is not required for every servicing operation. These checks can reveal
common pending-reboot indicators:

``` powershell
# Check the common Component-Based Servicing reboot-pending key.
Test-Path 'HKLM:\SOFTWARE\Microsoft\Windows\CurrentVersion\Component Based Servicing\RebootPending'

# Check the common Windows Update reboot-required key.
Test-Path 'HKLM:\SOFTWARE\Microsoft\Windows\CurrentVersion\WindowsUpdate\Auto Update\RebootRequired'

# Check for a pending file rename operation.
(Get-ItemProperty 'HKLM:\SYSTEM\CurrentControlSet\Control\Session Manager' `
    -Name PendingFileRenameOperations -ErrorAction SilentlyContinue).PendingFileRenameOperations
```

These are diagnostic indicators, not a complete or definitive
reboot-status test.

### 8.2 Restart Windows

``` powershell
# Restart Windows after saving work and closing applications.
Restart-Computer
```

### 8.3 Check free disk space

``` powershell
# Display free space on the Windows system drive.
Get-PSDrive -Name C |
    Select-Object Name,
        @{Name='UsedGB';Expression={[math]::Round($_.Used / 1GB, 2)}},
        @{Name='FreeGB';Expression={[math]::Round($_.Free / 1GB, 2)}}
```

### 8.4 Check Windows Update service state

``` powershell
# Display the status and startup type of common Windows Update servicing services.
Get-Service -Name wuauserv, bits, cryptsvc, TrustedInstaller |
    Select-Object Name, Status, StartType
```

Do not change service startup types or force-stop servicing services as
a routine troubleshooting step.

------------------------------------------------------------------------

## 9. Common errors and troubleshooting guidance

  -----------------------------------------------------------------------
  Error / symptom                     What to check
  ----------------------------------- -----------------------------------
  `0x800f081f`                        Required source files could not be
                                      found. Verify the source path,
                                      WIM/ESD index, language, edition,
                                      architecture, and build
                                      compatibility.

  `0x800f0915`                        The repair content could not be
                                      found. Verify the source and
                                      servicing compatibility; do not
                                      assume that any Windows ISO is
                                      suitable.

  `0x80240021`                        Windows Update timed out. Check
                                      network connectivity, proxy
                                      configuration, and Windows Update
                                      availability.

  DISM appears stuck at a percentage  Progress can remain unchanged while
                                      servicing continues. Check the live
                                      DISM/CBS logs and process activity;
                                      do not terminate it prematurely.

  `Access is denied`                  Confirm that PowerShell is elevated
                                      and that the target image or drive
                                      is accessible.

  Source path not found               Verify the mounted ISO drive letter
                                      and the exact spelling of
                                      `install.wim` or `install.esd`.
  -----------------------------------------------------------------------

### Proxy-specific troubleshooting

If online DISM cannot download repair content:

1.  Confirm that the proxy application is running.
2.  Confirm that the local listener is listening on the expected address
    and port.
3.  Test the HTTP proxy with `curl.exe`.
4.  Check `netsh winhttp show proxy`.
5.  Confirm that the configured WinHTTP proxy is an HTTP proxy endpoint,
    not a SOCKS-only endpoint.
6.  Retry DISM only after connectivity is verified.

If the proxy application is stopped while WinHTTP still points to
`127.0.0.1:10808`, applications using WinHTTP may fail to connect. Reset
WinHTTP proxy settings if direct access is desired.

------------------------------------------------------------------------

## 10. Recommended repair workflow

For a normally running Windows installation with suspected
component-store or system-file corruption:

``` powershell
# 1. Check whether Windows has already detected component-store corruption.
DISM.exe /Online /Cleanup-Image /CheckHealth

# 2. Perform a full component-store scan.
DISM.exe /Online /Cleanup-Image /ScanHealth

# 3. Repair the online Windows image.
DISM.exe /Online /Cleanup-Image /RestoreHealth

# 4. Verify and repair protected Windows system files.
sfc /scannow
```

If online repair fails because repair content cannot be downloaded:

1.  Check network connectivity and proxy configuration.
2.  If necessary, use a compatible Windows installation image as the
    repair source.
3.  Verify the image index with `DISM /Get-WimInfo`.
4.  Run `RestoreHealth` with the correct WIM/ESD source.
5.  Run `sfc /scannow` after DISM completes successfully.

For an offline Windows installation, use
`/Image:<offline-Windows-volume>\` with DISM and the offline SFC
parameters. Do not mix online and offline command syntax.

------------------------------------------------------------------------

## 11. Log file locations

  -------------------------------------------------------------------------------
  Log                     Location                        Purpose
  ----------------------- ------------------------------- -----------------------
  DISM                    `%windir%\Logs\DISM\dism.log`   DISM sessions, source
                                                          selection, and repair
                                                          operations

  CBS                     `%windir%\Logs\CBS\CBS.log`     Component-Based
                                                          Servicing, component
                                                          store, and SFC
                                                          servicing details
  -------------------------------------------------------------------------------

To open the log directory in File Explorer:

``` powershell
# Open the DISM log directory.
explorer.exe "$env:windir\Logs\DISM"

# Open the CBS log directory.
explorer.exe "$env:windir\Logs\CBS"
```

------------------------------------------------------------------------

## 12. Final notes

-   `DISM /Online` means the currently running Windows installation.
-   `DISM /Image:<path>` means an offline Windows installation.
-   `/LimitAccess` prevents DISM from contacting Windows Update; use it
    only when you intentionally want to rely on the specified source.
-   A WIM/ESD index is an image index, not a drive number. Always
    identify it with `/Get-WimInfo`.
-   Do not use a source simply because its filename says Windows 11.
    Verify its actual version, architecture, language, and edition.
-   `CheckHealth`, `ScanHealth`, and `RestoreHealth` have different
    purposes: quick status check, detailed scan, and repair.
-   A successful DISM result followed by an SFC result of **"Windows
    Resource Protection did not find any integrity violations"** is a
    good repair outcome. It does not guarantee that every possible
    Windows, driver, application, or hardware issue is absent.
-   If repair repeatedly fails, preserve the full command output and
    relevant DISM/CBS logs before attempting more invasive recovery
    options.

**Use these commands carefully.** In particular, double-check offline
drive letters and source-image indexes before running repair commands.
