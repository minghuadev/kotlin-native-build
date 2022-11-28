# kotlin-native-build

## arm32 soft float question

https://github.com/JetBrains/kotlin-native/issues/3350
How to make a linux arm32 soft float target?

sbogolepov commented Oct 2, 2020

There is a hacky way to create a static library for a soft float arm32 target when compiling for arm32_linux_hfp. See HACKING.md.
For example, by providing -Xoverride-clang-options=-cc1,target-cpu,cortex-a8,-mfloat-abi,soft,emit-obj,-disable-llvm-optzns,-x,ir compiler option when producing static library for linux_arm32_hfp you will get a static library for soft float arm32. Creation of dynamic library or executable involves linkage with files from sysroot for linux_arm32_hfp which will result in incorrect binary. But you can create a PIC binary by passing -mrelocation-model pic to Clang and then use an appropriate toolchain to create a dynamic library.

ref: kotlin-native repo


## arm32hfp coroutines question

https://github.com/Kotlin/kotlinx.coroutines/issues/855
Coroutines are not built for arm ( targets linuxArm32Hfp and others)

### by kotlinx.serialization, and on tizen

Thomas-Vos commented Dec 3, 2020

It would be great if the Linux arm32 target could be added. We would like to use this target for Tizen OS smart watches in our app. We are using kotlinx.coroutines but that does not appear to have the required target.

Kotlinx.serialization does have support for Linux arm targets, see the following: https://github.com/Kotlin/kotlinx.serialization/blob/fb65be2e43cd5df41ab1c8db0f456bcf57033e6f/gradle/native-targets.gradle#L95-L98

I am not sure how they are testing the Linux targets in their project.
harlem88, napperley, Animeshz, and sgalat-skyhook reacted with thumbs up emoji

Thomas-Vos commented Dec 4, 2020

@LouisCAD yes, I have Kotlin code working on physical Tizen OS watches (specifically the Galaxy Watch).

I am using the linuxArm32Hfp target. Cinterop with native Tizen libaries works too. Needed to use a workaround for soft float from here: JetBrains/kotlin-native#3350 (comment) (will be easier when Kotlin 1.4.30 is released with the new -Xoverride-konan-properties flag. However. it would be better if a new target is added for soft float). I am currently exporting a static lib and including that in Tizen Studio. The code works and I verified the Kotlin code can actually interop with Tizen code. Unfortunately, I am still waiting for the Linux targets to be added in AtomicFU, Coroutines, and Ktor, before I can actually start developing further.

The Tizen OS emulator is a bit more challenging. It is an x86 emulator. I think I would need a linuxX86 target but unfortunately that does not exist in Kotlin/Native. If you have any ideas, let me know.

### by pi and pi zero

soywiz commented Jul 16, 2021

Hey folks, we are also interested in this. We want to support Raspberry Pi and Raspberry Pi Zero in KorGE so we need 32-bit and 64-bit linux versions.

Made this PR here to add those targets to atomicfu: https://github.com/Kotlin/kotlinx.atomicfu/pull/193/files

Is anything we can do to move this forward, is there any blockers like not being able to execute tests, or it is just a matter of bandwidth?

ObsidianX commented Jul 25, 2021

Enabling Arm32/Arm64 is pretty simple: #2841 though based on one of the comments on the kotlinx.atomicfu issues board it sounds like it's a matter of testing and officially supporting the build, not just generating it.

### arm testing

dkhalanskyjb commented Sep 6, 2021

@JavierSegoviaCordoba please see https://youtrack.jetbrains.com/issue/KT-43996

### by danbrough

danbrough commented Nov 28, 2021

kotlinx.coroutines with linuxArm64 and linuxArm32Hfp

This fork has minimal changes to allow building linuxArm64 and linuxArm32Hfp kotlin native libraries.

Binaries are available on my maven repository at:
https://h1.danbrough.org/maven

The current versions available are:

    1.5.2-danbroid
    1.5.2-danbroid-native-mt
    1.6.0-RC-danbroid

With the source code at:
https://github.com/danbrough/kotlinx.coroutines
and https://github.com/danbrough/kotlinx.atomicfu

So far I've only tested the linuxArm64 binaries on a RPI3 running debian 64bit.

danbrough commented Sep 11, 2022

I've setup a github repo at https://github.com/danbrough/kotlinxtras/ as a hub for tweaked packages for linuxArm32Hfp,linuxArm64 and the android native targets.
It would be useful to anyone who wants to use kotlin native coroutines, okio, atomicfu, datetime, serialization, ktor with curl or who just needs curl or openssl support in their kotlin native app.
Those are all working and there are precompiled packages at:
https://s01.oss.sonatype.org/content/groups/staging/org/danbrough/ for pre-releases before they are published to: https://repo.maven.apache.org/maven2/org/danbrough/

### built on mac run on pi

clydebarrow commented Jan 26, 2022

    Binaries are available on my maven repository

Thanks! I've just built on a mac and run on a 32 bit RPI (original version with 512MB RAM) and it works! Awesome!

For the benefit of others, here is my build.gradle.kts with configs for host (macOS) and Pi targets:

plugins {
    kotlin("multiplatform") version "1.6.10"
    kotlin("plugin.serialization") version "1.6.10"
}

repositories {
    mavenCentral()
    maven(url = "https://h1.danbrough.org/maven/")
}

kotlin {
    val hostOs = System.getProperty("os.name")
    val isMingwX64 = hostOs.startsWith("Windows")
    val hostTarget = when {
        hostOs == "Mac OS X" -> macosX64("host")
        hostOs == "Linux" -> linuxX64("host")
        isMingwX64 -> mingwX64("host")
        else -> throw GradleException("Host OS is not supported in Kotlin/Native.")
    }

    hostTarget.apply {
        binaries {
            executable {
                entryPoint = "main"
            }
        }
    }
    linuxArm32Hfp("pi").apply {
        binaries {
            executable {
                entryPoint = "main"
            }
        }
    }
    sourceSets {
        commonMain {
            dependencies {
                implementation("org.jetbrains.kotlinx:kotlinx-serialization-json:1.3.2")
            }
        }
        val hostMain by getting {
            dependencies {
                implementation("org.jetbrains.kotlinx:kotlinx-datetime:0.3.2")
                implementation("org.jetbrains.kotlinx:kotlinx-coroutines-core:1.6.0")
            }
        }
        val piMain by getting {
            dependencies {
                implementation("org.jetbrains.kotlinx:kotlinx-coroutines-core-linuxarm32hfp:1.6.0-danbroid")
                implementation("org.jetbrains.kotlinx:kotlinx-datetime-linuxarm32hfp:0.3.1-danbroid")
            }
        }
    }
}

