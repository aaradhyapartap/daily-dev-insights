# 📌 WebAssembly use cases
*March 24, 2026 · Daily Dev Insight*

## 🧠 Overview

WebAssembly (WASM) has evolved far beyond its initial promise of "native performance in the browser." After nearly a decade in production, we've seen it carve out specific niches where it genuinely shines—and plenty where it's overkill. The real magic isn't just speed; it's about bringing mature, battle-tested libraries from systems languages into web environments without rewrites.

The most compelling use cases today involve computationally intensive tasks, legacy code migration, and cross-platform deployment scenarios. Think image processing, cryptographic operations, game engines, and scientific computing. However, WASM isn't a silver bullet—it comes with overhead, debugging complexity, and integration challenges that make it unsuitable for typical CRUD applications or simple DOM manipulation.

## 💡 Key Concepts

• **Performance isolation**: WASM excels at CPU-intensive tasks but adds overhead for simple operations—measure before migrating
• **Language bridge**: Enables running C/C++/Rust code in browsers and Node.js without full rewrites
• **Sandboxed execution**: Provides security benefits over native binaries while maintaining near-native performance
• **Universal deployment**: Same WASM binary can run in browsers, server-side, and edge environments
• **Memory management**: Direct memory access allows optimization but requires careful handling to avoid leaks

## 🐍 Python Example

```python
import wasmtime
import numpy as np
from pathlib import Path

class WASMImageProcessor:
    """Image processing pipeline using WASM for performance-critical operations"""
    
    def __init__(self, wasm_path: str):
        # Initialize WASM runtime and load compiled module
        self.engine = wasmtime.Engine()
        self.store = wasmtime.Store(self.engine)
        
        # Load pre-compiled WASM module (e.g., from C++ or Rust)
        wasm_bytes = Path(wasm_path).read_bytes()
        self.module = wasmtime.Module(self.engine, wasm_bytes)
        self.instance = wasmtime.Instance(self.store, self.module, [])
        
        # Extract exported functions
        self.blur_filter = self.instance.exports(self.store)["blur_filter"]
        self.edge_detect = self.instance.exports(self.store)["edge_detect"]
    
    def process_batch(self, images: list[np.ndarray]) -> list[np.ndarray]:
        """Process multiple images using WASM for 3-5x speedup over pure Python"""
        results = []
        
        for img in images:
            # Flatten image data for WASM consumption
            height, width, channels = img.shape
            flat_data = img.flatten().astype(np.float32)
            
            # Allocate memory in WASM linear memory space
            memory = self.instance.exports(self.store)["memory"]
            data_ptr = self._allocate_buffer(len(flat_data) * 4)  # 4 bytes per float32
            
            # Copy image data to WASM memory
            memory_view = memory.data_ptr(self.store)
            np.frombuffer(memory_view[data_ptr:data_ptr + len(flat_data) * 4], 
                         dtype=np.float32)[:] = flat_data
            
            # Execute WASM function (blur + edge detection pipeline)
            result_ptr = self.blur_filter.call(self.store, data_ptr, width, height, channels)
            self.edge_detect.call(self.store, result_ptr, width, height)
            
            # Read processed data back from WASM memory
            processed_data = np.frombuffer(
                memory_view[result_ptr:result_ptr + len(flat_data) * 4], 
                dtype=np.float32
            ).reshape((height, width, channels))
            
            results.append(processed_data.copy())  # Copy to Python-managed memory
            
        return results
    
    def _allocate_buffer(self, size: int) -> int:
        """Allocate buffer in WASM linear memory"""
        malloc_fn = self.instance.exports(self.store)["malloc"]
        return malloc_fn.call(self.store, size)

# Usage example
processor = WASMImageProcessor("image_filters.wasm")
batch_results = processor.process_batch([img1, img2, img3])
```

## 🟨 JavaScript Example

```javascript
import { readFile } from 'fs/promises';
import { performance } from 'perf_hooks';

class CryptoWASM {
    constructor() {
        this.module = null;
        this.memory = null;
    }
    
    async initialize() {
        // Load WASM module compiled from Rust crypto library
        const wasmBytes = await readFile('./crypto_utils.wasm');
        const wasmModule = await WebAssembly.compile(wasmBytes);
        
        // Create instance with imported memory for large datasets
        const memory = new WebAssembly.Memory({ 
            initial: 256, // 16MB initial
            maximum: 1024 // 64MB max
        });
        
        this.instance = await WebAssembly.instantiate(wasmModule, {
            env: {
                memory: memory,
                console_log: (ptr, len) => {
                    const bytes = new Uint8Array(memory.buffer, ptr, len);
                    console.log(new TextDecoder().decode(bytes));
                }
            }
        });
        
        this.memory = memory;
        console.log('WASM crypto module loaded successfully');
    }
    
    async hashLargeFile(filePath) {
        const startTime = performance.now();
        
        // Read file in chunks to avoid memory issues
        const fileData = await readFile(filePath);
        const dataLength = fileData.length;
        
        // Allocate buffer in WASM linear memory
        const malloc = this.instance.exports.malloc;
        const free = this.instance.exports.free;
        const sha256_hash = this.instance.exports.sha256_hash;
        
        const inputPtr = malloc(dataLength);
        const outputPtr = malloc(32); // SHA256 = 32 bytes
        
        try {
            // Copy file data to WASM memory
            const wasmMemory = new Uint8Array(this.memory.buffer);
            wasmMemory.set(fileData, inputPtr);
            
            // Execute WASM hashing function (typically 2-3x faster than JS)
            const result = sha256_hash(inputPtr, dataLength, outputPtr);
            
            if (result !== 0) {
                throw new Error(`WASM hashing failed with code ${result}`);
            }
            
            // Read hash result back from WASM memory
            const hashBytes = wasmMemory.slice(outputPtr, outputPtr + 32);
            const hashHex = Array.from(hashBytes)
                .map(b => b.toString(16).padStart(2, '0'))
                .join('');
            
            const endTime = performance.now();
            console.log(`Hashed ${dataLength} bytes in ${endTime - startTime:.2f}ms`);
            
            return hashHex;
            
        } finally {
            // Always cleanup WASM memory allocations
            free(inputPtr);
            free(outputPtr);
        }
    }
    
    // Batch processing multiple files efficiently
    async hashBatch(filePaths) {
        const results = new Map();
        
        for (const filePath of filePaths) {
            results.set(filePath, await this.hashLargeFile(filePath));
        }
        
        return results;
    }
}

// Usage
const crypto = new CryptoWASM();
await crypto.initialize();
const hashes = await crypto.hashBatch(['file1.bin', 'file2.bin', 'file3.bin']);
```

## ⚖️ When To Use / When To Avoid

**✅