# Embedding in Node.js, Deno, Bun Application
This example demonstrates how to run Javy in a Node.js (v20+), Deno, and Bun host application.

## Warning
This example does NOT show how to run a Node.js, Deno, and Bun application in Javy. This is
useful for when you want to run untrusted user generated code in a sandbox. This
code is meant to be an example not production-ready code.

It's also important to note that the WASI implementation in Node.js is currently
considered [experimental]. In Deno's current implementation of `node:wasi` all [exports are non-functional stubs].
In Bun's current implementation of `node:wasi` [is just a quick hack to get WASI working]. 
`wasi.js`, used here for Deno and Bun support is a modified version of [`deno-wasi`] [adapted to support `node`, `deno`, and  `bun`], and is intended to be JavaScript runtime agnostic.

[experimental]: https://nodejs.org/api/wasi.html#webassembly-system-interface-wasi
[exports are non-functional stubs]: https://docs.deno.com/api/node/wasi/
[is just a quick hack to get WASI working]: https://github.com/oven-sh/bun/blob/main/src/js/node/wasi.ts#L7
[`deno-wasi`]: https://github.com/caspervonb/deno-wasi
[adapted to support `node`, `deno`, and  `bun`]: https://github.com/guest271314/deno-wasi/tree/runtime-agnostic-nodejs-api

## Summary
This example shows how to use a dynamically linked Javy compiled Wasm module. We
use std in/out/error to communicate with the embedded JavaScript. See [this blog
post](https://k33g.hashnode.dev/wasi-communication-between-nodejs-and-wasm-modules-another-way-with-stdin-and-stdout)
for details.


### Steps

1. Emit the Javy plugin
```shell
javy emit-plugin -o plugin.wasm
```
2. Compile the `embedded.js` with Javy using dynamic linking:
```shell
javy build -C dynamic -C plugin=plugin.wasm -o embedded.wasm embedded.js
```
3. Run `host.mjs`
```shell
node --no-warnings=ExperimentalWarning host.js
```


`embedded.js`
```javascript
// Read input from stdin
const input = readInput();
// Call the function with the input
const result = foo(input);
// Write the result to stdout
writeOutput(result);

// The main function.
function foo(input) {
  if (input && typeof input === "object" && typeof input.n === "number") {
    return { n: input.n + 1 };
  }
  return { n: 0 };
}

// Read input from stdin
function readInput() {
  const chunkSize = 1024;
  const inputChunks = [];
  let totalBytes = 0;

  // Read all the available bytes
  while (1) {
    const buffer = new Uint8Array(chunkSize);
    // Stdin file descriptor
    const fd = 0;
    const bytesRead = Javy.IO.readSync(fd, buffer);

    totalBytes += bytesRead;
    if (bytesRead === 0) {
      break;
    }
    inputChunks.push(buffer.subarray(0, bytesRead));
  }

  // Assemble input into a single Uint8Array
  const { finalBuffer } = inputChunks.reduce(
    (context, chunk) => {
      context.finalBuffer.set(chunk, context.bufferOffset);
      context.bufferOffset += chunk.length;
      return context;
    },
    { bufferOffset: 0, finalBuffer: new Uint8Array(totalBytes) },
  );

  const maybeJson = new TextDecoder().decode(finalBuffer);
  try {
    return JSON.parse(maybeJson);
  } catch {
    return;
  }
}

// Write output to stdout
function writeOutput(output) {
  const encodedOutput = new TextEncoder().encode(JSON.stringify(output));
  const buffer = new Uint8Array(encodedOutput);
  // Stdout file descriptor
  const fd = 1;
  Javy.IO.writeSync(fd, buffer);
}
```


`host.js`
```javascript
import { open, readFile, rm, writeFile } from "node:fs/promises";
import { join } from "node:path";
import { tmpdir } from "node:os";
import { randomUUID } from "node:crypto";
import { WASI } from "node:wasi";

try {
  const [embeddedModule, pluginModule] = await Promise.all([
    compileModule("./embedded.wasm"),
    compileModule("./plugin.wasm"),
  ]);
  const result = await runJavy(pluginModule, embeddedModule, { n: 100 });
  console.log("Success!", JSON.stringify(result, null, 2));
} catch (e) {
  console.log(e);
}

async function compileModule(wasmPath) {
  const bytes = await readFile(new URL(wasmPath, import.meta.url));
  return WebAssembly.compile(bytes);
}

async function runJavy(pluginModule, embeddedModule, input) {
  const uniqueId = randomUUID();
  // Use stdin/stdout/stderr to communicate with Wasm instance
  // See https://k33g.hashnode.dev/wasi-communication-between-nodejs-and-wasm-modules-another-way-with-stdin-and-stdout
  const workDir = tmpdir();
  const stdinFilePath = join(workDir, `stdin.wasm.${uniqueId}.txt`);
  const stdoutFilePath = join(workDir, `stdout.wasm.${uniqueId}.txt`);
  const stderrFilePath = join(workDir, `stderr.wasm.${uniqueId}.txt`);

  // 👋 send data to the Wasm instance
  await writeFile(stdinFilePath, JSON.stringify(input), { encoding: "utf8" });

  const [stdinFileFd, stdoutFileFd, stderrFileFd] = await Promise.all([
    open(stdinFilePath, "r"),
    open(stdoutFilePath, "a"),
    open(stderrFilePath, "a"),
  ]);

  // console.log(stdinFileFd.fd, stdoutFileFd.fd, stderrFileFd.fd);

  const wasiOptions = {
    version: "preview1",
    returnOnExit: true,
    args: [],
    env: {},
    stdin: stdinFileFd.fd,
    stdout: stdoutFileFd.fd,
    stderr: stderrFileFd.fd,
  };

  try {
    // Deno's "node:wasi" is a stub, not implemented
    // https://docs.deno.com/api/node/wasi/
    // https://github.com/denoland/deno/issues/21025
    let wasi = null;

    try {
      wasi = navigator.userAgent.startsWith("Node.js")
        ? new WASI(wasiOptions)
        : wasi;
    } catch (e) {
      // Deno
      // https://docs.deno.com/api/node/wasi/
      // All exports are non-functional stubs.
      // Error: Context is currently not supported at new Context (node:wasi:6:11)
      console.log(e);
    } finally {
      wasi ??= new (await import("./wasi.js")).default(wasiOptions);
    }

    const pluginInstance = await WebAssembly.instantiate(
      pluginModule,
      { wasi_snapshot_preview1: wasi?.wasiImport || wasi?.exports },
    ).catch(console.log);

    const instance = await WebAssembly.instantiate(embeddedModule, {
      "javy-default-plugin-v3": pluginInstance.exports,
    }).catch(console.log);
    wasi.memory = pluginInstance.exports.memory;
    // Javy plugin is a WASI reactor see https://github.com/WebAssembly/WASI/blob/main/legacy/application-abi.md?plain=1
    // Bun's "node:wasi" module doesn't have an `initialize` method, hangs here
    // Bun's documentation says the method is implemented
    // https://bun.com/reference/node/wasi
    wasi.initialize ??= wasi?.start;
    wasi?.initialize?.(pluginInstance);
    instance.exports._start();

    const [out, err] = await Promise.all([
      readOutput(stdoutFilePath),
      readOutput(stderrFilePath),
    ]);
    if (err) {
      throw new Error(err);
    }

    return out;
  } catch (e) {
    if (e instanceof WebAssembly.RuntimeError) {
      const errorMessage = await readOutput(stderrFilePath);
      if (errorMessage) {
        throw new Error(errorMessage);
      }
    }
    throw e;
  } finally {
    Promise.all([
      stdinFileFd.close(),
      stdoutFileFd.close(),
      stderrFileFd.close(),
      rm(stdinFilePath),
      rm(stdoutFilePath),
      rm(stderrFilePath),
    ]);
  }
}

async function readOutput(filePath) {
  const str = (await readFile(filePath, "utf8")).trim();
  try {
    return JSON.parse(str);
  } catch {
    return str;
  }
}

```
`wasi.js`
```javascript
// Modified deno-wasi implementation for Deno, Bun, Node.js
// https://github.com/caspervonb/deno-wasi
// https://github.com/guest271314/deno-wasi/tree/runtime-agnostic-nodejs-api
import process from "node:process";
import fs from "node:fs";
import crypto from "node:crypto";

const encoder = new TextEncoder();

const CLOCKID_REALTIME = 0;
const CLOCKID_MONOTONIC = 1;
const CLOCKID_PROCESS_CPUTIME_ID = 2;
const CLOCKID_THREAD_CPUTIME_ID = 3;
const ERRNO_SUCCESS = 0;
const ERRNO_BADF = 8;
const ERRNO_INVAL = 28;
const ERRNO_NOSYS = 52;
const ERRNO_NOTDIR = 54;
const RIGHTS_FD_DATASYNC = 0x0000000000000001n;
const RIGHTS_FD_READ = 0x0000000000000002n;
const RIGHTS_FD_WRITE = 0x0000000000000040n;
const RIGHTS_FD_ALLOCATE = 0x0000000000000100n;
const RIGHTS_FD_READDIR = 0x0000000000004000n;
const RIGHTS_FD_FILESTAT_SET_SIZE = 0x0000000000400000n;
const FILETYPE_UNKNOWN = 0;
const FILETYPE_CHARACTER_DEVICE = 2;
const FILETYPE_DIRECTORY = 3;
const FILETYPE_REGULAR_FILE = 4;
const FILETYPE_SYMBOLIC_LINK = 7;
const FDFLAGS_APPEND = 0x0001;
const FDFLAGS_DSYNC = 0x0002;
const FDFLAGS_NONBLOCK = 0x0004;
const FDFLAGS_RSYNC = 0x0008;
const FDFLAGS_SYNC = 0x0010;
const FSTFLAGS_ATIM_NOW = 0x0002;
const FSTFLAGS_MTIM_NOW = 0x0008;
const OFLAGS_CREAT = 0x0001;
const OFLAGS_DIRECTORY = 0x0002;
const OFLAGS_EXCL = 0x0004;
const OFLAGS_TRUNC = 0x0008;
const PREOPENTYPE_DIR = 0;
const clock_res_realtime = function() {
  return BigInt(1e6);
};
const clock_res_monotonic = function() {
  return BigInt(1e3);
};
const clock_res_process = clock_res_monotonic;
const clock_res_thread = clock_res_monotonic;
const clock_time_realtime = function() {
  return BigInt(Date.now()) * BigInt(1e6);
};
const clock_time_monotonic = function() {
  const t = performance.now();
  const s = Math.trunc(t);
  const ms = Math.floor((t - s) * 1e3);
  return BigInt(s) * BigInt(1e9) + BigInt(ms) * BigInt(1e6);
};
const clock_time_process = clock_time_monotonic;
const clock_time_thread = clock_time_monotonic;

function errno(err) {
  switch (err.name) {
    case "NotFound":
      return 44;
    case "PermissionDenied":
      return 2;
    case "ConnectionRefused":
      return 14;
    case "ConnectionReset":
      return 15;
    case "ConnectionAborted":
      return 13;
    case "NotConnected":
      return 53;
    case "AddrInUse":
      return 3;
    case "AddrNotAvailable":
      return 4;
    case "BrokenPipe":
      return 64;
    case "InvalidData":
      return 28;
    case "TimedOut":
      return 73;
    case "Interrupted":
      return 27;
    case "BadResource":
      return 8;
    case "Busy":
      return 10;
    default:
      return 28;
  }
}
class WASI {
  args;
  env;
  memory;
  fds;
  exports;
  constructor(options) {
    this.args = options?.args ? options.args : [];
    this.env = options?.env ? options.env : {};
    this.memory = options?.memory;
    this.fds = [
      {
        type: FILETYPE_CHARACTER_DEVICE,
        handle: options?.stdin ? { fd: options.stdin } : process.stdin
      },
      {
        type: FILETYPE_CHARACTER_DEVICE,
        handle: options?.stdout ? { fd: options.stdout } : process.stdout
      },
      {
        type: FILETYPE_CHARACTER_DEVICE,
        handle: options?.stderr ? { fd: options.stderr } : process.stderr
      }
    ];
    this.exports = {
      clock_time_get: (id, precision, time_out) => {
        const view = new DataView(this.memory.buffer);
        switch (id) {
          case CLOCKID_REALTIME:
            view.setBigUint64(time_out, clock_time_realtime(), true);
            break;
          case CLOCKID_MONOTONIC:
            view.setBigUint64(time_out, clock_time_monotonic(), true);
            break;
          case CLOCKID_PROCESS_CPUTIME_ID:
            view.setBigUint64(time_out, clock_time_process(), true);
            break;
          case CLOCKID_THREAD_CPUTIME_ID:
            view.setBigUint64(time_out, clock_time_thread(), true);
            break;
          default:
            return ERRNO_INVAL;
        }
        return ERRNO_SUCCESS;
      },
      environ_get: (environ_ptr, environ_buf_ptr) => {
        const entries = Object.entries(this.env);
        const heap = new Uint8Array(this.memory.buffer);
        const view = new DataView(this.memory.buffer);
        for (let [key, value] of entries) {
          view.setUint32(environ_ptr, environ_buf_ptr, true);
          environ_ptr += 4;
          const data = encoder.encode(`${key}=${value}\x00`);
          heap.set(data, environ_buf_ptr);
          environ_buf_ptr += data.length;
        }
        return ERRNO_SUCCESS;
      },
      environ_sizes_get: (environc_out, environ_buf_size_out) => {
        const entries = Object.entries(this.env);
        const view = new DataView(this.memory.buffer);
        view.setUint32(environc_out, entries.length, true);
        view.setUint32(environ_buf_size_out, entries.reduce(function(acc, [key, value]) {
          return acc + encoder.encode(`${key}=${value}\x00`).length;
        }, 0), true);
        return ERRNO_SUCCESS;
      },
      fd_close: (fd) => {
        const entry = this.fds[fd];
        if (!entry) {
          return ERRNO_BADF;
        }
        fs.close(entry.handle.fd);
        delete this.fds[fd];
        return ERRNO_SUCCESS;
      },
      fd_fdstat_get: (fd, stat_out) => {
        const entry = this.fds[fd];
        if (!entry) {
          return ERRNO_BADF;
        }
        const view = new DataView(this.memory.buffer);
        view.setUint8(stat_out, entry.type);
        view.setUint16(stat_out + 4, 0, true);
        view.setBigUint64(stat_out + 8, 0n, true);
        view.setBigUint64(stat_out + 16, 0n, true);
        return ERRNO_SUCCESS;
      },
      fd_read: (fd, iovs_ptr, iovs_len, nread_out) => {
        const entry = this.fds[fd];
        if (!entry) {
          return ERRNO_BADF;
        }
        const view = new DataView(this.memory.buffer);
        let nread = 0;
        for (let i = 0;i < iovs_len; i++) {
          const data_ptr = view.getUint32(iovs_ptr, true);
          iovs_ptr += 4;
          const data_len = view.getUint32(iovs_ptr, true);
          iovs_ptr += 4;
          const data = new Uint8Array(this.memory.buffer, data_ptr, data_len);
          nread += fs.readSync(entry.handle.fd, data);
        }
        view.setUint32(nread_out, nread, true);
        return ERRNO_SUCCESS;
      },
      fd_seek: (fd, offset, whence, newoffset_out) => {
        const entry = this.fds[fd];
        if (!entry) {
          return ERRNO_BADF;
        }
        const view = new DataView(this.memory.buffer);
        try {
          const newoffset = entry.handle.seekSync(Number(offset), whence);
          view.setBigUint64(newoffset_out, BigInt(newoffset), true);
        } catch (err) {
          return ERRNO_INVAL;
        }
        return ERRNO_SUCCESS;
      },
      fd_write: (fd, iovs_ptr, iovs_len, nwritten_out) => {
        const entry = this.fds[fd];
        if (!entry) {
          return ERRNO_BADF;
        }
        const view = new DataView(this.memory.buffer);
        let nwritten = 0;
        for (let i = 0;i < iovs_len; i++) {
          const data_ptr = view.getUint32(iovs_ptr, true);
          iovs_ptr += 4;
          const data_len = view.getUint32(iovs_ptr, true);
          iovs_ptr += 4;
          nwritten += fs.writeSync(entry.handle.fd || entry.handle, new Uint8Array(this.memory.buffer, data_ptr, data_len));
        }
        view.setUint32(nwritten_out, nwritten, true);
        return ERRNO_SUCCESS;
      },
      proc_exit: (rval) => {
        process.exit(rval);
      },
      random_get: (buf_ptr, buf_len) => {
        const buffer = new Uint8Array(this.memory.buffer, buf_ptr, buf_len);
        crypto.getRandomValues(buffer);
        return ERRNO_SUCCESS;
      }
    };
  }
}
export {
  WASI as default, WASI
};
```
