# Debugging Ray Cluster with VSCode

A step-by-step guide to debugging Ray’s core components (`GCS server` & `Raylet`) using VSCode + GDB, with manual startup and minimal source modifications.

## Motivation

When developing or contributing to Ray, it's often necessary to debug processes like `raylet` and `gcs_server`. However, Ray's default launch mechanism makes it difficult to attach debuggers like GDB. This guide shows how to manually start and debug these components in an IDE-friendly way.

---

## 1. Modifying Ray's Startup Scripts

To debug `raylet`, you can patch `services.py` and `node.py` to suppress process auto-launching and expose CLI arguments.

### `services.py`

- Increase default timeout for better debug experience.
- Patch `start_gcs_server` and `start_raylet` to print CLI commands instead of launching processes.

```diff
# python/ray/_private/services.py
@@ -30,9 +30,9 @@
 if sys.platform != "win32":
-    _timeout = 30
+    _timeout = 600
 else:
-    _timeout = 60
+    _timeout = 600

@@ def start_gcs_server(...):
+    # print(command)  # print to manually copy into launch.json

@@ def start_raylet(...):
+    print(command)
+    return None  # prevent auto-spawn during `ray start`
```

### `node.py`

- Suppress log monitor to reduce noise.

```diff
# python/ray/_private/node.py
@@ class Node:
-        if self._ray_params.include_log_monitor:
-            self.start_log_monitor()
+        # if self._ray_params.include_log_monitor:
+        #     self.start_log_monitor()
```

---

## 2. VSCode Debug Config (`.vscode/launch.json`)

Two launch configurations:

- **Debug GCS Server**
- **Debug Raylet**

These replicate the full command-line arguments normally passed by `ray start`, allowing you to attach VSCode + GDB with full context.

Make sure to extract the command printed from the above `print(command)` modification.

---

## 3. Build Ray with Debug Symbols

Use Bazel with debug mode enabled:

```bash
bazel build -c dbg //:gcs_server //:raylet
```

You can also append:

```bash
--strip=never
```

...for full symbol info.

---

## 4. Enable `buildifier` in VSCode

To improve Bazel file formatting support:

1. Press `Ctrl+Shift+P`, select `Preferences: Open Settings (Remote)`.
2. Add the following entry:

```json
"bazel.buildifierExecutable": "/home/bizon/go/bin/buildifier"
```

---

## 5. Start Ray Cluster Partially

1. Run patched `ray start --head`, which prints out the `gcs_server` and `raylet` commands.
2. Let it initialize the cluster and session directory.
3. Copy those commands into VSCode `launch.json`.
4. Use the VSCode debugger to launch and inspect GCS or Raylet manually.


