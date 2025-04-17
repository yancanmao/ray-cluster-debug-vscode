# Debugging Ray Cluster with VSCode

A step-by-step guide to debugging Ray’s core components (GCS server & Raylet) using VSCode + GDB, with manual startup and source modifications.

## Motivation

When developing or contributing to Ray, it's often necessary to debug processes like `raylet` and `gcs_server`. However, Ray's default launch mechanism makes it difficult to attach debuggers like GDB. This repo shows how to manually start and debug these processes in an IDE-friendly way.

## Modifications on Ray Python Start Scripts

If you want to debug `raylet`, you can disable raylet start in `services.py` and `node.py`

1. `services.py`

- Extended timeout (_timeout = 600)

- Disabled process spawning for `start_raylet`, `start_gcs_server` and printed commands instead.

```shell
# git diff python/ray/_private/services.py
diff --git a/python/ray/_private/services.py b/python/ray/_private/services.py
index f8b477e34d..b71bc6ac59 100644
--- a/python/ray/_private/services.py
+++ b/python/ray/_private/services.py
@@ -30,9 +30,9 @@ from ray._private.ray_constants import RAY_NODE_IP_FILENAME
 
 resource = None
 if sys.platform != "win32":
-    _timeout = 30
+    _timeout = 600
 else:
-    _timeout = 60
+    _timeout = 600
 
 EXE_SUFFIX = ".exe" if sys.platform == "win32" else ""
 
@@ -874,6 +874,8 @@ def start_ray_process(
         Information about the process that was started including a handle to
             the process that was started.
     """
+
+
     # Detect which flags are set through environment variables.
     valgrind_env_var = f"RAY_{process_type.upper()}_VALGRIND"
     if os.environ.get(valgrind_env_var) == "1":
@@ -1524,6 +1526,8 @@ def start_gcs_server(
     if stderr_filepath:
         stderr_file = open(os.devnull, "w")
 
+    # print(command)
+
     process_info = start_ray_process(
         command,
         ray_constants.PROCESS_TYPE_GCS_SERVER,
@@ -1898,19 +1902,23 @@ def start_raylet(
     if stderr_filepath:
         stderr_file = open(os.devnull, "w")
 
-    process_info = start_ray_process(
-        command,
-        ray_constants.PROCESS_TYPE_RAYLET,
-        use_valgrind=use_valgrind,
-        use_gdb=False,
-        use_valgrind_profiler=use_profiler,
-        use_perftools_profiler=("RAYLET_PERFTOOLS_PATH" in os.environ),
-        stdout_file=stdout_file,
-        stderr_file=stderr_file,
-        fate_share=fate_share,
-        env_updates=env_updates,
-    )
-    return process_info
+    print(command)
```


2. `node.py`
- Commented out log monitor to prevent noise

- Prevented some auto-start logic so you can launch components manually via VSCode

  ```shell
  # git diff python/ray/_private/node.py
  diff --git a/python/ray/_private/node.py b/python/ray/_private/node.py
index 5958fb1a95..a6a8ee7566 100644
--- a/python/ray/_private/node.py
+++ b/python/ray/_private/node.py
@@ -1486,8 +1486,8 @@ class Node:
             huge_pages=self._ray_params.huge_pages,
         )
         self.start_raylet(plasma_directory, fallback_directory, object_store_memory)
-        if self._ray_params.include_log_monitor:
-            self.start_log_monitor()
+        # if self._ray_params.include_log_monitor:
+        #     self.start_log_monitor()
 
     def _kill_process_type(
         self,
  ```

## VSCode Debug Config (`.vscode/launch.json`)

Includes two configurations:

- Debug Raylet
- Debug GCS Server

Each manually replicates the command line flags normally assembled by Ray's bootstrap logic.


## Building with Debug Info

Ensure Bazel builds GCS and Raylet with debug symbols:

```shell
bazel build -c dbg //:raylet //:gcs_server
```

## Set buildifier in VSCode

- In VSCode, run `ctrl + shift + p`, select `preferences: open (remote)`, in settings, add buildifier executable.

```shell
"bazel.buildifierExecutable": "/home/bizon/go/bin/buildifier"
```

## Start Ray Cluster Partially without Raylet

- Modify `ray start --head` to print commands instead of launching.
- Run the command once to generate session directory and config.
- Copy the printed command line args into launch.json.
- Use VSCode + GDB to launch Raylet or GCS.





