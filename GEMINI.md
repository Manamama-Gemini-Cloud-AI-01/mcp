Below is a list of tips by ChatGPT.

The User selects:`lsof -ti:9009 | xargs --no-run-if-empty kill -9` to implement afterchecks.


### The root cause: Wrong `kill` command usage + error handling logic

File: `/src/ws.ts`

1. **What the code tries to do:**

The `killProcessOnPort(port)` function tries to forcibly kill any process currently listening on the specified port (here, port 9009) so that the BrowserMCP server can bind to it without conflict.

The common Unix approach is:

```bash
lsof -ti:<port> | xargs kill -9
```

* `lsof -ti:<port>` lists the PID(s) of any processes listening on that port.
* `xargs kill -9` sends SIGKILL to those PIDs.

2. **What goes wrong:**

* When **no process** is using port 9009 (the normal and expected state before the server starts), `lsof -ti:9009` returns **empty output**.
* `xargs kill -9` then tries to execute `kill -9` **without any PID arguments**.

This triggers the `kill` command's **usage error** because `kill` expects at least one PID argument. Hence you get the usage message:

```
Usage:
 kill [options] <pid> [...]
 ...
```

3. **Why this is a fatal error in the code:**

* The Node.js `execSync` call that runs `lsof -ti:9009 | xargs kill -9` throws an exception because the shell command fails with exit code 123 (command usage error).
* The error is not caught or handled gracefully inside `killProcessOnPort`.
* The function throws the error upward, which causes the MCP server initialization to fail or freeze, depending on error handling downstream.

4. **Why the "freeze" or stall occurs:**

* The server startup code tries to clear the port before binding.
* Because the error is unhandled, it doesn't recover.
* The client (e.g., Gemini CLI) waits indefinitely or throws errors, giving the appearance of freeze or hang.

---

### The offending snippet (in pseudocode):

```ts
function killProcessOnPort(port) {
  // runs this shell command synchronously:
  execSync(`lsof -ti:${port} | xargs kill -9`);
  // This will throw if no process is found (empty list, kill called without PID)
}
```

---

### How the community fixed it (see your pasted fix):

Add a **guard check** before attempting to kill:

```ts
async function createWebSocketServer(port = mcpConfig.defaultWsPort) {
  if (await isPortInUse(port)) {  // only try to kill if port is actually in use
    killProcessOnPort(port);
    while (await isPortInUse(port)) {
      await wait(100);
    }
  }
  return new WebSocketServer({ port });
}
```

* `isPortInUse(port)` performs a non-failing check for any process on the port.
* If no process is found, it **skips the kill call**, avoiding the erroneous empty kill call.
* This stops the fatal error and lets the server bind cleanly.

---

### Additional notes on the error itself:

* The actual error comes from the shell command line syntax — the kill command usage fails because `kill` requires a PID and none was supplied.
* This is a common pitfall when piping commands: `xargs` will call the target command even if input is empty, unless told otherwise.
* A better shell command might have been:

```bash
pids=$(lsof -ti:9009)
if [ -n "$pids" ]; then kill -9 $pids; fi
```

or

```bash
lsof -ti:9009 | xargs --no-run-if-empty kill -9
```

The flag `--no-run-if-empty` prevents running `kill` if no PIDs are found (but it is GNU-specific and not universal).

---

### Summary: Why does the "wrong kill" cause the problem?

* **Empty PID list → kill called without args → kill usage error → unhandled exception in Node.js → MCP server startup fails or stalls.**

---

### What to do for your compiled version?

1. Patch `killProcessOnPort` to catch this error or avoid it by checking port usage first.
2. Or patch the shell command to be safe for empty output.
3. Or upgrade/downgrade to the recommended `@browsermcp/mcp@0.1.2` version that already includes this fix.

---

If you want, I can help you write a **precise patch** snippet you can inject into your compiled code or source before rebuild, or suggest the minimal fix for your shell command.

---

Would you like me to do that?

