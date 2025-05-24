---
title: "Microsoft Defender Throttled Full Scan Experiment"
#description: Examples of text, typography, math equations, diagrams, flowcharts, pictures, videos, and more.
date: 2025-05-25 00:00:00 +0800
last_modified_at: 2025-05-25 00:00:00 +0800
categories: [Miscellaneous]
tags: [Microsoft,Defender]
pin: true
image:
  path: "/assets/img/2025-05-24-microsoft-defender-fullscan-experiment/blog-image.png"
---
## Introduction

In this article, we will test different flags for the command `Set-MpPreference` to see the behavior of CPU throttling when performing a Full Scan. I took inspiration from my team at Intelligent Technical Solutions for the commands and wanted to see how those behave using different configurations. 

You may be asking, why do we need to throttle our full scans? We may be in a situation where a user is actively using the computer and we need to perform a full scan in the background. It is important for us to throttle the Full Scan to avoid taking up most of the CPU utilization and slow down the computer's performance for the user's tasks.

A quick breakdown of the flags that we will use for the `Set-MpPreference` command:

#### -DisableCpuThrottleOnIdleScans
{: data-toc-skip='' .mt-4 .mb-0 }
<br>
As per Microsoft, this flag indicates whether the CPU will be throttled for scheduled scans while the device is idle. This parameter is enabled by default, thus ensuring that the CPU won't be throttled for scheduled scans performed when the device is idle, regardless of what `ScanAvgCPULoadFactor` is set to. For all other scheduled scans, this flag does not have any impact and normal throttling will occur. The default value is `True`.

#### -ScanAvgCPULoadFactor
{: data-toc-skip='' .mt-4 .mb-0 }
<br>
As per Microsoft, this flag specifies the maximum percentage CPU usage for a scan. The acceptable values for this parameter are: integers from 5 through 100, and the value 0, which disables CPU throttling. Windows Defender does not exceed the percentage of CPU usage that you specify. The default value is 50.

This is not a hard limit but rather a guidance for the scanning engine to not exceed this maximum on average. If ScanOnlyIfIdleEnabled (instructing the product to scan only when the computer is not in use) and DisableCpuThrottleOnIdleScans (instructing the product to disable CPU throttling on idle scans) are both enabled, then the value of ScanAvgCPULoadFactor is ignored.

#### -ScanOnlyIfIdleEnabled
{: data-toc-skip='' .mt-4 .mb-0 }
<br>
As per Microsoft, this flag Indicates whether to start scheduled scans only when the computer is not in use. If you specify a value of $True or do not specify a value, Windows Defender runs schedules scans when the computer is on, but not in use. The Default value is `None`.

#### -ThrottleForScheduledScanOnly
{: data-toc-skip='' .mt-4 .mb-0 }
<br>
As per Microsoft, A CPU usage limit can be applied to scheduled scans only, or to scheduled and custom scans. The default value applies a CPU usage limit to scheduled scans only. The acceptable values for this parameter are:
- 1 (Default) - If you enable this setting, CPU throttling will apply only to scheduled scans.
- 0 - If you disable this setting, CPU throttling will apply to scheduled and custom scans.

<br>
Now that the flags have been discussed, we can now proceed with the experiment.

## Full Scan Configurations

#### Normal Full Scan

```powershell
Start-MpScan -ScanType FullScan
```

This will be our control. In this configuration, we are using all the Default values and running a on-demand Full Scan. As you can see in the image below, the CPU Utilization of the `Antimalware Service Executable` which is the Microsoft Defender is around ~96% CPU Utilization and our overall utilization is at 98%.

![Desktop View](/assets/img/2025-05-24-microsoft-defender-fullscan-experiment/Control.png)

#### Configuration 1

```powershell
Set-MpPreference -ScanAvgCPULoadFactor 40
Start-MpScan -ScanType FullScan
```

In this configuration, we have set the `-ScanAvgCPULoadFactor` to 40 to limit the CPU Utilization of the scan to 40%. We noticed that the CPU Utilization for the process is still around 95%.

![Desktop View](/assets/img/2025-05-24-microsoft-defender-fullscan-experiment/Config-1.png)

#### Configuration 2

```powershell
Set-MpPreference -ScanAvgCPULoadFactor 40
Set-MpPreference -DisableCpuThrottleOnIdleScans $False
Start-MpScan -ScanType FullScan
```

In this configuration, we have set the `-DisableCpuThrottleOnIdleScans` to False to allow us to throttle the process for scheduled scans when the device is idle. As you can see, the CPU utilization for the Defender process is around 96%.

![Desktop View](/assets/img/2025-05-24-microsoft-defender-fullscan-experiment/Config-2.png)

#### Configuration 3

```powershell
Set-MpPreference -ScanAvgCPULoadFactor 40
Set-MpPreference -DisableCpuThrottleOnIdleScans $False
Set-MpPreference -ScanOnlyIfIdleEnabled $False
Start-MpScan -ScanType FullScan
```

In this configuration, we have set both the `-DisableCpuThrottleOnIdleScans` and `-ScanOnlyIfIdleEnabled` to False to allow us to throttle the process for scheduled scans when the device is idle or when the computer is on, but not in use. As you can see, the CPU utilization for the Defender process is still around 95%.

![Desktop View](/assets/img/2025-05-24-microsoft-defender-fullscan-experiment/Config-3.png)

#### Configuration 4

```powershell
Set-MpPreference -ScanAvgCPULoadFactor 40
Set-MpPreference -DisableCpuThrottleOnIdleScans $False
Set-MpPreference -ScanOnlyIfIdleEnabled $False
Set-MpPreference -ThrottleForScheduledScanOnly $False
Start-MpScan -ScanType FullScan
```

In this configuration, we have set the flags `-DisableCpuThrottleOnIdleScans`, `-ScanOnlyIfIdleEnabled`, and `-ThrottleForScheduledScanOnly` to False to allow us to throttle the process for scheduled scans when the device is idle and when we are running a custom scan. As you can see, the CPU utilization for the Defender process is around 31% which is below the 40% threshold we set  for the `ScanAvgCPULoadFactor`. This means that we have successfully throttled the Full Scan process.

![Desktop View](/assets/img/2025-05-24-microsoft-defender-fullscan-experiment/Config-4.png)

#### Configuration 5

```powershell
Set-MpPreference -ScanAvgCPULoadFactor 40
Set-MpPreference -DisableCpuThrottleOnIdleScans $True
Set-MpPreference -ScanOnlyIfIdleEnabled $False
Set-MpPreference -ThrottleForScheduledScanOnly $False
Start-MpScan -ScanType FullScan
```
In this configuration, we have set the flag  `-DisableCpuThrottleOnIdleScans` to True while `-ScanOnlyIfIdleEnabled` and `-ThrottleForScheduledScanOnly` are set to False. As you can see, the CPU utilization for the Defender process is still around 30%.

![Desktop View](/assets/img/2025-05-24-microsoft-defender-fullscan-experiment/Config-5.png)

#### Configuration 6

```powershell
Set-MpPreference -ScanAvgCPULoadFactor 40
Set-MpPreference -DisableCpuThrottleOnIdleScans $True
Set-MpPreference -ScanOnlyIfIdleEnabled $True
Set-MpPreference -ThrottleForScheduledScanOnly $False
Start-MpScan -ScanType FullScan
```
In this configuration, we have set the flags  `-DisableCpuThrottleOnIdleScans` and `-ScanOnlyIfIdleEnabled` to True or to their Default value and `-ThrottleForScheduledScanOnly` is still set to False. As you can see, the CPU utilization for the Defender process is still around 31%.

This means that the flag `ThrottleForScheduledScanOnly` is the one that actually allows us to throttle the Full Scan since we are running an On-demand scan and not a scheduled scan. Though based on the documentation, it does not say that it will throttle the on-demand scan as well, the on-demand scan might be under custom scans in this scenario.

![Desktop View](/assets/img/2025-05-24-microsoft-defender-fullscan-experiment/Config-6.png)


## Conclusion

In conclusion, the flag `ThrottleForScheduledScanOnly` along with `ScanAvgCPULoadFactor` allowed us to throttle the Defender Full Scan as we are running an on-demand scan and not a scheduled scan. You may also set the rest of the flags shown above if you wanted to set throttling for scheduled scans. Kudos to my team at ITS in creating the script which inspired me to create this simple experiment.

Disclaimer: Please do not take the information in this article as a fact as the results of this experiment are purely based on my observations and research. Please do research on your own and feel free to perform this experiment in your environment.

## References
- Set-MpPreference: <https://learn.microsoft.com/en-us/powershell/module/defender/set-mppreference?view=windowsserver2025-ps>
- Start-MpScan: <https://learn.microsoft.com/en-us/powershell/module/defender/start-mpscan?view=windowsserver2025-ps>
- DisableCpuThrottleOnIdleScans: <https://learn.microsoft.com/en-us/powershell/module/defender/set-mppreference?view=windowsserver2025-ps#-disablecputhrottleonidlescans>
- ScanAvgCPULoadFactor: <https://learn.microsoft.com/en-us/powershell/module/defender/set-mppreference?view=windowsserver2025-ps#-scanavgcpuloadfactor>
- ScanOnlyIfIdleEnabled: <https://learn.microsoft.com/en-us/powershell/module/defender/set-mppreference?view=windowsserver2025-ps#-scanonlyifidleenabled>
- ThrottleForScheduledScanOnly: <https://learn.microsoft.com/en-us/powershell/module/defender/set-mppreference?view=windowsserver2025-ps#-throttleforscheduledscanonly>
- Defender On-Demand Scan: <https://learn.microsoft.com/en-us/defender-endpoint/run-scan-microsoft-defender-antivirus>