# AMD P-State and AMD P-State EPP Scaling Driver Configuration Guide

## 1. Requirements

Currently, some of the Zen2 and Zen3 processors support `amd-pstate` and the new `amd_pstate_epp` scaling driver. You also have to have CPPC support enabled in your UEFI. In the future, it will be supported on more and more AMD processors.

## 2. amd-pstate vs acpi-cpufreq

There are two methods for adjusting CPU performance on AMD CPU/APUs:
-	`amd-pstate` 
-	`acpi-cpufreq`

`acpi-cpufreq` is currently default for most distros, regardless of the CPU in use. on most AMD CPUs this is a limiting factor, as it offers limited performance options with only a few fixed levels for CPU speed.

On newer AMD CPUs and APUs (aka Zen2 and above), there is a more advanced method called *Collaborative Processor Performance Control* (CPPC mentioned in the requirements), which allows for fine-tuned and continuous adjustments of the CPU frequency, with the potential to provide better performance and energy efficiency compared to the older fixed levels.

And that's where `amd-pstate` comes in, as it is a new kernel module that supports the newer and more efficient AMD P-States mechanism.

There are 3 options available, listed below, in order of release:

-	`amd_pstate=passive` (Kernel 6.1+)

-	`amd_pstate=active` (Kernel 6.3+)

-	`amd_pstate=guided` (kernel 6.4+)


### Passive Mode

`amd_pstate=passive`

When you set `amd_pstate=passive`, the processor aims for a certain performance level relative to its maximum capacity. Below a specific point, the performance is average, while above it, the performance remains at its best.

### Active Mode

`amd_pstate=active`

Setting `amd_pstate=active` gives low-level control to the processor's firmware. It can prioritize either performance or energy efficiency based on software **hints** AND the `amd_pstate_epp` driver.
The `amd_pstate_epp` (Energy Performance Preference) driver provides the firmware with a **hint**. 
On most AMD CPUs, these **hints** are:
-	`default `
-	`performance`
-	`balance_performance`
-	`balance_power`
-	`power`

### Guided Mode

`amd_pstate=guided`

Choosing `amd_pstate=guided` lets the platform automatically select a suitable performance level within a given range based on the workload.


## 3a. Configure amd_pstate to either Passive or Guided

To enable the `amd_pstate_epp` scaling driver, which also includes instructions for the original `amd_pstate` scaling driver, you will need to add a kernel parameter. If you are using PopOS (like me) or any other distribution utilising kernelstub, this process can be easily accomplished with the following steps:

>**IMPORTANT: The option 'amd_pstate=guided' is only available on Kernel 6.4 or later versions.**

1.	Add the desired kernel parameter by running the following command:

```
# Add the desired Kernel Parameter
sudo kernelstub -a "amd_pstate=guided" # Change this to passive if preferred
```
2.	To confirm that the kernel parameter has been successfully added, use the following command:
```
# Verify that the kernel parameter has been added
sudo kernelstub -p 
```


### Verify amd_pstate
To verify that this is functioning correctly, reboot your machine, and run 
```
cat /sys/devices/system/cpu/cpu0/cpufreq/scaling_driver
```

If `amd_pstate` was set to either `passive` or `guided`, this should now show:
```
amd-pstate
```

## 3b. Configure amd_pstate_epp to Active

To enable the `amd_pstate_epp` scaling driver, which also includes instructions for the original `amd_pstate` scaling driver, you will need to add a kernel parameter. If you are using PopOS (like me) or any other distribution utilising kernelstub, this process can be easily accomplished with the following steps:

>**IMPORTANT: The option 'amd_pstate=active' is only available on Kernel 6.3 or later versions.**

1.	Add the desired kernel parameter by running the following command:

```
# Add the desired Kernel Parameter
sudo kernelstub -a "amd_pstate=active"
```
2.	To confirm that the kernel parameter has been successfully added, use the following command:
```
# Verify that the kernel parameter has been added
sudo kernelstub -p 
```


### Verify amd_pstate

To verify that this is functioning correctly, reboot your machine, and run 
```
cat /sys/devices/system/cpu/cpu0/cpufreq/scaling_driver
```

If `amd_pstate` was set to `active`, this should now show:
```
amd-pstate-epp
```

### Configure amd_pstate_epp Energy Performance Preference

The amd_pstate_epp scaling driver introduces a new parameter known as "Energy Performance Preference" (EPP) hint. This setting can be adjusted through sysfs, with two main files controlling it:

 - `/sys/devices/system/cpu/cpu*/cpufreq/energy_performance_preference`: This file displays the current EPP hint for the respective CPU core.

 - `/sys/devices/system/cpu/cpu*/cpufreq/energy_performance_available_preferences`: This file provides the available EPP hints for the respective CPU core.    

To see your current EPP hints (note `*` = all CPU cores), use the following command:

```
cat /sys/devices/system/cpu/cpu*/cpufreq/energy_performance_preference

```

To view the available EPP hints (which should be the same for all cores), use this command:

```
cat /sys/devices/system/cpu/cpu0/cpufreq/energy_performance_available_preferences

# What you see below, is my results on my Ryzen 7 7735HS
default performance balance_performance balance_power power 
```

If you'd like to set the same EPP hint across all cores, for instance, setting EPP to "power" (like in my case), you can use this command:

```
echo "power" | sudo tee /sys/devices/system/cpu/cpu*/cpufreq/energy_performance_preference
power
```
>**NOTE: This is not permanent, and will be reverted upon reboot. To make this permanent, you can use multiple tools, or, create a cron job**

## 4. Scaling Driver vs CPU Governor

The Scaling Driver is *different* than the CPU governor (e.g. `powersave`, `performance`, `ondemand`, `schedulutil`, etc.), and the two can be mixed and matched to create your perfect combo.

To check what's the current `cpu governor`, use the command below:
```
cat /sys/devices/system/cpu/cpu*/cpufreq/scaling_governor
```

In my case, that's what I'm seeing:
```
user@machine ~> cat /sys/devices/system/cpu/cpu*/cpufreq/scaling_governor
powersave
powersave
powersave
powersave
powersave
powersave
powersave
powersave
powersave
powersave
powersave
powersave
powersave
powersave
powersave
powersave
```

If you've configured `amd_pstate=active`, you can mix and match governors with EPP hints. Phoronix has an excellent breakdown of all the combinations of governors + EPP hints (referenced in the resources section at the end of this post).

Personally, for my laptop usage, I still find `amd_pstate=passive` to be the best for my use case, but YMMV depending on the devices you're configuring this on, and your use case :)

## 5. [OPTIONAL] Automating EPP Switching when on Battery/AC

Thanks to the amazing work of [/jothiprasath](https://www.reddit.com/user/jothiprasath/), we've can now switch EPP Hints automatically when going from Battery to AC, and viceversa.

Here's the link to his amazing work [Auto-EPP](https://github.com/jothi-prasath/auto-epp)

>**NOTE: This hasn't been written by me and I've yet to test it, please make sure you have reviewed the code before deploying it to your machines**


### Resources:
- an amazing Redditor (whose post I cannot find anymore) that served as a basis for this very post (if anyone finds it, please do let me know, and I'll reference them right away)

- ChatGPT who helped me phrase some sentences a bit better

- Benchmarks for server using AMD P-State EPP: https://www.phoronix.com/review/linux-63-amd-epyc-epp
    
- Benchmarks for Ryzen mobile system using AMD P-State EPP: https://www.phoronix.com/review/amd-pstate-epp-ryzen-mobile

- Kernel.org documentation on new AMD P-State driver: https://www.kernel.org/doc/html/latest/admin-guide/pm/amd-pstate.html
    
- Arch Wiki page on CPU Scaling: [https://wiki.archlinux.org/title/CPU\_frequency\_scaling](https://wiki.archlinux.org/title/CPU_frequency_scaling)
