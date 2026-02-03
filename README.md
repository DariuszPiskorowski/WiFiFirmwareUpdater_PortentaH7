
# README — Fixing Portenta H7 WiFi Firmware (QSPI / LittleFS) with the **Fixed** WiFi Firmware Updater (Arduino IDE)

## When you see this message…

If you ever see:

> `Failed to mount the filesystem containing the WiFi firmware. Usually that means that the WiFi firmware has not been installed yet or was overwritten…`

it means the updater **cannot mount the QSPI filesystem** (the flash partition where the WiFi firmware image is expected to live, usually **LittleFS**). In practice, this happens when:

* the QSPI filesystem was **never formatted**, or
* it became **corrupted**, or
* it was **overwritten** (e.g., by another sketch, a crash, or a different storage configuration).

When this mount fails, the updater cannot access the firmware blob and cannot proceed reliably until QSPI is fixed.

---

## What was fixed (the “Y/[n] loop” problem)

You also encountered:

> `A WiFi firmware is already installed. Do you want to install the firmware anyway? Y/[n]`

…and the updater got stuck repeatedly asking the same question.

### What that loop actually meant

The updater *did* detect an existing firmware, but it **never successfully consumed your answer** from Serial input. So the prompt logic kept re-triggering instead of moving forward.

### Where the bug was in the original updater

The original file had a **Serial input handling flaw**, typically one (or more) of these:

* It **did not flush** the Serial buffer (CR/LF remained and was read as “empty input”).
* It didn’t wait properly for a **real character** (`y` / `n`) and kept re-printing the prompt.
* It treated “nothing read” as “ask again” rather than defaulting to `[n]` and continuing.

Result: even if you typed `Y`, the code often read junk / newline / nothing and **re-entered the prompt**, creating an infinite loop.

---

## How the fixed updater works now (correct logic)

The fixed updater now behaves predictably:

### 1) Mount QSPI filesystem (LittleFS)

* The updater first tries to **mount** the QSPI filesystem.
* If mount fails, it **formats QSPI** (LittleFS) and tries to mount again.
* If it still fails after formatting, it stops and prints a clear failure.

This directly addresses the “Failed to mount the filesystem…” case.

### 2) “Firmware already installed” prompt that actually works

When you see:

> `A WiFi firmware is already installed. Do you want to install the firmware anyway? Y/[n]`

the fixed logic is:

* It **cleans Serial input** first (flushes leftover newline characters).
* It **waits for one valid key**:

  * `y` / `Y` → proceed with install (force re-install)
  * `n` / `N` → skip install and continue
* If you type nothing (or timeout), it defaults to **`n`**.
* It **never** re-prompts forever — it makes a decision and moves on.

So the loop is gone because input is handled deterministically.



## Step-by-step fix using Arduino IDE

### Step 1 — Select the correct board/port

1. Open **Arduino IDE**.
2. Go to **Tools → Board** and select:

   * **Arduino Portenta H7**
3. Go to **Tools → Port** and select the Portenta’s COM/USB port.

### Step 2 — Use the *fixed* WiFi firmware updater sketch

1. Open the **fixed** WiFi firmware updater sketch (this corrected version you already have).
2. Confirm you did **not** revert to the original broken file (the one that caused the `Y/[n]` loop).

### Step 3 — Upload the updater sketch

1. Click **Upload**.
2. Wait until Arduino IDE finishes uploading.

### Step 4 — Open Serial Monitor

1. Open **Tools → Serial Monitor**
2. Set baud rate to **115200** (or whatever the updater uses consistently).

### Step 5 — Watch for the mount / format sequence

You should see one of these flows:

#### ✅ Case A: QSPI mounts correctly

You’ll see output indicating the filesystem mounted successfully and the updater can read the firmware image.

#### ✅ Case B: QSPI mount fails, then it formats and mounts

If you see the mount failure message, the fixed code should then:

* announce formatting,
* format QSPI (LittleFS),
* mount again successfully,
* then continue with firmware installation flow.

This is the expected recovery path.

### Step 6 — Handle the “already installed” prompt (no loop anymore)

If you see:

> `A WiFi firmware is already installed. Do you want to install the firmware anyway? Y/[n]`

Do one of the following:

* Type `Y` and press Enter → it will **force reinstall** and proceed.
* Press Enter without typing anything → it defaults to `n` and **skips reinstall**.

You will not get stuck in an endless prompt anymore.

### Step 7 — Verify success

After the updater completes, you should see a clear confirmation message indicating completion (the exact string depends on your sketch’s prints). The key verification is:

* No repeated `Y/[n]` prompt loop
* No persistent mount failure after format
* Updater reaches the end of its flow cleanly



## Important notes / what you no longer need to do

* You **do not need** to put Portenta into “green breathing LED mode” anymore for this process.
* This README intentionally does **not** cover uploading your own application firmware — the goal here is strictly the WiFi firmware repair/install.



## Quick troubleshooting checklist (if you still get NaNs / WiFi issues later)

This README is about the WiFi firmware filesystem/updater, but if you later see WiFi failing:

* Make sure the updater completed successfully after formatting/mounting.
* Power cycle the board once after firmware install (often helps).
* Re-check that you’re using the correct board core version and correct Port.


