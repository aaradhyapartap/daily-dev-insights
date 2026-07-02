# 📌 WebAssembly use cases
*July 02, 2026 · Daily Dev Insight*

## 🧠 Overview

WebAssembly (Wasm) has matured from a curious browser technology into a serious polyglot runtime that's reshaping how we think about performance-critical code deployment. The core insight here isn't just "it's fast" — it's that Wasm provides a **portable compilation target** that runs at near-native speed across browsers, servers, and edge environments. This means you can write compute-intensive logic in Rust, C++, or Go, compile to Wasm, and call it seamlessly from JavaScript, Python, or any host environment with a Wasm runtime.

The real power emerges when you stop thinking of WebAssembly as a JavaScript replacement and start seeing it as a **capability extender**. Legacy codebases in C/C++ can be brought to the web without rewrites. CPU-bound algorithms (image processing, cryptography, compression) can be offloaded from slower interpreted languages. And perhaps most importantly, Wasm's sandboxed security model makes it ideal for running untrusted code — think user plugins, serverless functions, or multi-tenant environments where isolation matters more than raw convenience.

The use cases have crystallized around a few key patterns: client-side performance (games, CAD tools, video editors), serverless edge computing (Cloudflare Workers, Fastly Compute), plugin systems (extending databases, editors, or apps with safe user code), and portable libraries (write once, call from any language). If you're hitting performance walls with interpreted languages or need security isolation without container overhead, Wasm deserves serious consideration.

## 💡 Key Concepts

- **Near-native performance**: Wasm runs 10-100x faster than JavaScript for CPU-intensive tasks like numerical computation, media processing, or parsing, making it ideal for performance bottlenecks in web apps
- **Language-agnostic interop**: Compile from 40+ languages (Rust, C++, Go, AssemblyScript) and call from any host — breaking down language silos while keeping each tool in its sweet spot
- **Sandboxed by default**: Wasm modules run in a capability-based security model with no ambient authority — they can only access what you explicitly provide, making it perfect for untrusted code execution
- **Portable across environments**: The same `.wasm` binary runs in browsers, Node.js, Deno, server runtimes (Wasmtime, WasmEdge), and embedded systems — true "write once, run anywhere"
- **Lightweight isolation**: Wasm instantiation is milliseconds vs seconds for containers, with KB overhead instead of MB, making it ideal for edge computing and multi-tenant serverless platforms

## 🟨 JavaScript Example

```javascript
// Using WebAssembly to accelerate image processing in the browser
// Assume we've compiled a Rust image filter to 'filter.wasm'

async function initWasmImageFilter() {
  // Fetch and instantiate the WebAssembly module
  const response = await fetch('filter.wasm');
  const buffer = await response.arrayBuffer();
  const wasmModule = await WebAssembly.instantiate(buffer, {
    env: {
      // Provide imports the Wasm module needs
      memory: new WebAssembly.Memory({ initial: 256, maximum: 512 })
    }
  });

  const { blur_image, memory } = wasmModule.instance.exports;

  // Process an image using the Wasm function
  async function applyBlur(imageData) {
    const { data, width, height } = imageData;
    
    // Allocate space in Wasm linear memory
    const imageSize = width * height * 4; // RGBA
    const inputPtr = 0;
    
    // Copy image data into Wasm memory
    const wasmMemory = new Uint8Array(memory.buffer);
    wasmMemory.set(data, inputPtr);
    
    // Call the Wasm blur function (runs 20-50x faster than JS)
    const outputPtr = blur_image(inputPtr, width, height, 5); // radius=5
    
    // Copy processed data back
    const processedData = new Uint8ClampedArray(
      memory.buffer, 
      outputPtr, 
      imageSize
    );
    
    return new ImageData(processedData, width, height);
  }

  return { applyBlur };
}

// Usage with Canvas API
const canvas = document.getElementById('myCanvas');
const ctx = canvas.getContext('2d');
const imageData = ctx.getImageData(0, 0, canvas.width, canvas.height);

const { applyBlur } = await initWasmImageFilter();
const blurred = await applyBlur(imageData);
ctx.putImageData(blurred, 0, 0);
```

## 🐍 Python Example

```python
# Using Wasmtime to run untrusted user plugins safely in a Python app
# Install: pip install wasmtime

from wasmtime import Store, Module, Instance, Engine, Func, FuncType, ValType
import json

class PluginRunner:
    """Safely execute user-provided WebAssembly plugins with controlled access"""
    
    def __init__(self, wasm_path):
        # Create isolated Wasm runtime
        self.engine = Engine()
        self.store = Store(self.engine)
        
        # Load the user's plugin module
        with open(wasm_path, 'rb') as f:
            self.module = Module(self.engine, f.read())
        
        # Define host functions the plugin can call
        def log_wrapper(caller, ptr, len):
            """Allow plugins to log messages (controlled I/O)"""
            memory = caller["memory"]
            bytes_data = memory.read(self.store, ptr, len)
            print(f"[Plugin]: {bytes_data.decode('utf-8')}")
        
        # Create function binding with type signature
        log_type = FuncType([ValType.i32(), ValType.i32()], [])
        log_func = Func(self.store, log_type, log_wrapper)
        
        # Instantiate with limited imports (capability-based security)
        self.instance = Instance(self.store, self.module, [log_func])
    
    def execute_plugin(self, input_data):
        """Run the plugin's main function with input"""
        # Get exported functions from the Wasm module
        process = self.instance.exports(self.store)["process"]
        alloc = self.instance.exports(self.store)["alloc"]
        memory = self.instance.exports(self.store)["memory"]
        
        # Allocate memory in Wasm and write input
        input_bytes = json.dumps(input_data).encode('utf-8')
        input_ptr = alloc(self.store, len(input_bytes))
        memory.write(self.store, input_bytes, input_ptr)
        
        # Execute the plugin (sandboxed, no file system or network access)
        result_ptr = process(self.store, input_ptr, len(input_bytes))
        
        return result_ptr

# Usage: Run untrusted data transformation safely
runner = PluginRunner('user_plugin.wasm')
result = runner.execute_plugin({"data": [1, 2, 3, 4, 5]})
print(f"Plugin result: {result}")
```

## ⚖️ When To Use / When To Avoid

**✅ Use WebAssembly when:**
- You have CPU-intensive algorithms (image/video processing, compression, cryptography, ML inference)
- Porting existing C/C++/Rust codebases to web or multi-platform environments
- Building plugin systems requiring strong security isolation without container overhead
- Edge computing where cold start time and memory footprint matter (serverless functions)
- Cross-language library sharing (write performance-critical code once, call from anywhere)

**❌ Avoid WebAssembly when:**
- Your code is mostly I/O-bound (database queries, network requests) — Wasm overhead isn't worth it
- Heavy DOM manipulation or async operations — JavaScript's event loop is better suited
- Rapid prototyping where native tooling and debugging experience matter more than performance
- Your team lacks experience with systems languages