# 實戰項目：WASM 應用開發

## 項目一：圖像處理應用

構建一個高性能的瀏覽器端圖像處理工具，展示 WASM 的計算優勢。

### 項目結構

```
image-processor/
├── Cargo.toml
├── src/
│   ├── lib.rs
│   ├── filters.rs
│   └── utils.rs
├── www/
│   ├── index.html
│   ├── index.js
│   └── style.css
└── package.json
```

### Cargo.toml

```toml
[package]
name = "image-processor"
version = "0.1.0"
edition = "2021"

[lib]
crate-type = ["cdylib"]

[dependencies]
wasm-bindgen = "0.2"
console_error_panic_hook = "0.1"
wee_alloc = "0.4"

[dependencies.web-sys]
version = "0.3"
features = [
    "console",
    "ImageData",
]

[profile.release]
opt-level = 's'
lto = true
```

### src/lib.rs

```rust
use wasm_bindgen::prelude::*;

mod filters;
mod utils;

#[cfg(feature = "wee_alloc")]
#[global_allocator]
static ALLOC: wee_alloc::WeeAlloc = wee_alloc::WeeAlloc::INIT;

#[wasm_bindgen(start)]
pub fn main() {
    utils::set_panic_hook();
}

/// 灰階濾鏡
#[wasm_bindgen]
pub fn grayscale(data: &mut [u8]) {
    for pixel in data.chunks_exact_mut(4) {
        let gray = (0.299 * pixel[0] as f32 
                  + 0.587 * pixel[1] as f32 
                  + 0.114 * pixel[2] as f32) as u8;
        pixel[0] = gray;
        pixel[1] = gray;
        pixel[2] = gray;
        // pixel[3] (alpha) 保持不變
    }
}

/// 亮度調整
#[wasm_bindgen]
pub fn adjust_brightness(data: &mut [u8], amount: i16) {
    for pixel in data.chunks_exact_mut(4) {
        for i in 0..3 {
            let value = pixel[i] as i16 + amount;
            pixel[i] = value.clamp(0, 255) as u8;
        }
    }
}

/// 對比度調整
#[wasm_bindgen]
pub fn adjust_contrast(data: &mut [u8], factor: f32) {
    let factor = (259.0 * (factor + 255.0)) / (255.0 * (259.0 - factor));
    
    for pixel in data.chunks_exact_mut(4) {
        for i in 0..3 {
            let value = factor * (pixel[i] as f32 - 128.0) + 128.0;
            pixel[i] = value.clamp(0.0, 255.0) as u8;
        }
    }
}

/// 反轉顏色
#[wasm_bindgen]
pub fn invert(data: &mut [u8]) {
    for pixel in data.chunks_exact_mut(4) {
        pixel[0] = 255 - pixel[0];
        pixel[1] = 255 - pixel[1];
        pixel[2] = 255 - pixel[2];
    }
}

/// 模糊濾鏡（簡單平均）
#[wasm_bindgen]
pub fn blur(data: &[u8], width: u32, height: u32, radius: u32) -> Vec<u8> {
    let mut result = vec![0; data.len()];
    let stride = width * 4;
    
    for y in 0..height {
        for x in 0..width {
            let mut r = 0u32;
            let mut g = 0u32;
            let mut b = 0u32;
            let mut count = 0u32;
            
            // 遍歷鄰域
            for dy in -(radius as i32)..=(radius as i32) {
                for dx in -(radius as i32)..=(radius as i32) {
                    let nx = (x as i32 + dx).clamp(0, width as i32 - 1) as u32;
                    let ny = (y as i32 + dy).clamp(0, height as i32 - 1) as u32;
                    let idx = (ny * stride + nx * 4) as usize;
                    
                    r += data[idx] as u32;
                    g += data[idx + 1] as u32;
                    b += data[idx + 2] as u32;
                    count += 1;
                }
            }
            
            let idx = (y * stride + x * 4) as usize;
            result[idx] = (r / count) as u8;
            result[idx + 1] = (g / count) as u8;
            result[idx + 2] = (b / count) as u8;
            result[idx + 3] = data[idx + 3]; // alpha
        }
    }
    
    result
}

/// 圖像旋轉 90 度
#[wasm_bindgen]
pub fn rotate_90(data: &[u8], width: u32, height: u32) -> Vec<u8> {
    let mut result = vec![0; data.len()];
    
    for y in 0..height {
        for x in 0..width {
            let src_idx = (y * width + x) as usize * 4;
            let dst_idx = ((width - 1 - x) * height + y) as usize * 4;
            
            result[dst_idx..dst_idx + 4].copy_from_slice(&data[src_idx..src_idx + 4]);
        }
    }
    
    result
}
```

### src/utils.rs

```rust
use wasm_bindgen::prelude::*;

pub fn set_panic_hook() {
    #[cfg(feature = "console_error_panic_hook")]
    console_error_panic_hook::set_once();
}

#[wasm_bindgen]
extern "C" {
    #[wasm_bindgen(js_namespace = console)]
    pub fn log(s: &str);
}

#[macro_export]
macro_rules! console_log {
    ($($t:tt)*) => {
        $crate::utils::log(&format!($($t)*))
    }
}
```

### www/index.html

```html
<!DOCTYPE html>
<html lang="zh-TW">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>WASM 圖像處理器</title>
    <link rel="stylesheet" href="style.css">
</head>
<body>
    <div class="container">
        <h1>🎨 WASM 圖像處理器</h1>
        
        <div class="upload-section">
            <input type="file" id="fileInput" accept="image/*">
            <button id="loadBtn">載入圖片</button>
        </div>
        
        <div class="controls">
            <button id="grayscaleBtn">灰階</button>
            <button id="invertBtn">反轉</button>
            <button id="blurBtn">模糊</button>
            <button id="rotate90Btn">旋轉 90°</button>
            
            <div class="slider-group">
                <label for="brightness">亮度: <span id="brightnessValue">0</span></label>
                <input type="range" id="brightness" min="-100" max="100" value="0">
            </div>
            
            <div class="slider-group">
                <label for="contrast">對比度: <span id="contrastValue">0</span></label>
                <input type="range" id="contrast" min="-100" max="100" value="0">
            </div>
            
            <button id="resetBtn">重置</button>
        </div>
        
        <div class="canvas-container">
            <canvas id="canvas"></canvas>
        </div>
        
        <div id="performance"></div>
    </div>
    
    <script type="module" src="./index.js"></script>
</body>
</html>
```

### www/index.js

```javascript
import init, * as wasm from './pkg/image_processor.js';

let canvas, ctx, originalImageData, currentImageData;

async function run() {
    // 初始化 WASM
    await init();
    console.log('WASM initialized');
    
    canvas = document.getElementById('canvas');
    ctx = canvas.getContext('2d');
    
    setupEventListeners();
}

function setupEventListeners() {
    document.getElementById('loadBtn').addEventListener('click', loadImage);
    document.getElementById('grayscaleBtn').addEventListener('click', applyGrayscale);
    document.getElementById('invertBtn').addEventListener('click', applyInvert);
    document.getElementById('blurBtn').addEventListener('click', applyBlur);
    document.getElementById('rotate90Btn').addEventListener('click', applyRotate90);
    document.getElementById('resetBtn').addEventListener('click', reset);
    
    document.getElementById('brightness').addEventListener('input', (e) => {
        document.getElementById('brightnessValue').textContent = e.target.value;
        applyBrightness(parseInt(e.target.value));
    });
    
    document.getElementById('contrast').addEventListener('input', (e) => {
        document.getElementById('contrastValue').textContent = e.target.value;
        applyContrast(parseFloat(e.target.value));
    });
}

function loadImage() {
    const fileInput = document.getElementById('fileInput');
    const file = fileInput.files[0];
    
    if (!file) {
        alert('請選擇圖片');
        return;
    }
    
    const reader = new FileReader();
    reader.onload = (e) => {
        const img = new Image();
        img.onload = () => {
            canvas.width = img.width;
            canvas.height = img.height;
            ctx.drawImage(img, 0, 0);
            
            originalImageData = ctx.getImageData(0, 0, canvas.width, canvas.height);
            currentImageData = ctx.getImageData(0, 0, canvas.width, canvas.height);
        };
        img.src = e.target.result;
    };
    reader.readAsDataURL(file);
}

function measurePerformance(name, fn) {
    const start = performance.now();
    fn();
    const elapsed = performance.now() - start;
    
    document.getElementById('performance').textContent = 
        `${name} 執行時間: ${elapsed.toFixed(2)}ms`;
}

function applyGrayscale() {
    if (!currentImageData) return;
    
    measurePerformance('灰階', () => {
        const data = currentImageData.data;
        wasm.grayscale(data);
        ctx.putImageData(currentImageData, 0, 0);
    });
}

function applyInvert() {
    if (!currentImageData) return;
    
    measurePerformance('反轉', () => {
        const data = currentImageData.data;
        wasm.invert(data);
        ctx.putImageData(currentImageData, 0, 0);
    });
}

function applyBrightness(amount) {
    if (!originalImageData) return;
    
    currentImageData = ctx.getImageData(0, 0, canvas.width, canvas.height);
    currentImageData.data.set(originalImageData.data);
    
    measurePerformance('亮度調整', () => {
        wasm.adjust_brightness(currentImageData.data, amount);
        ctx.putImageData(currentImageData, 0, 0);
    });
}

function applyContrast(factor) {
    if (!originalImageData) return;
    
    currentImageData = ctx.getImageData(0, 0, canvas.width, canvas.height);
    currentImageData.data.set(originalImageData.data);
    
    measurePerformance('對比度調整', () => {
        wasm.adjust_contrast(currentImageData.data, factor);
        ctx.putImageData(currentImageData, 0, 0);
    });
}

function applyBlur() {
    if (!currentImageData) return;
    
    measurePerformance('模糊', () => {
        const result = wasm.blur(
            currentImageData.data,
            canvas.width,
            canvas.height,
            2  // radius
        );
        currentImageData.data.set(result);
        ctx.putImageData(currentImageData, 0, 0);
    });
}

function applyRotate90() {
    if (!currentImageData) return;
    
    measurePerformance('旋轉', () => {
        const result = wasm.rotate_90(
            currentImageData.data,
            canvas.width,
            canvas.height
        );
        
        // 交換寬高
        [canvas.width, canvas.height] = [canvas.height, canvas.width];
        
        currentImageData = ctx.getImageData(0, 0, canvas.width, canvas.height);
        currentImageData.data.set(result);
        ctx.putImageData(currentImageData, 0, 0);
        
        originalImageData = currentImageData;
    });
}

function reset() {
    if (!originalImageData) return;
    
    currentImageData = ctx.getImageData(0, 0, canvas.width, canvas.height);
    currentImageData.data.set(originalImageData.data);
    ctx.putImageData(currentImageData, 0, 0);
    
    document.getElementById('brightness').value = 0;
    document.getElementById('contrast').value = 0;
    document.getElementById('brightnessValue').textContent = '0';
    document.getElementById('contrastValue').textContent = '0';
}

run();
```

### package.json

```json
{
  "name": "image-processor",
  "version": "0.1.0",
  "scripts": {
    "build": "wasm-pack build --target web",
    "serve": "python3 -m http.server 8080"
  }
}
```

### 構建與運行

```bash
# 構建 WASM
npm run build

# 啟動開發伺服器
npm run serve

# 瀏覽器訪問 http://localhost:8080/www/
```

---

## 項目二：實時遊戲引擎（Conway's Game of Life）

實現經典的生命遊戲，展示 WASM 的高幀率渲染能力。

### src/lib.rs

```rust
use wasm_bindgen::prelude::*;
use std::fmt;

#[wasm_bindgen]
pub struct Universe {
    width: u32,
    height: u32,
    cells: Vec<Cell>,
}

#[wasm_bindgen]
#[repr(u8)]
#[derive(Clone, Copy, Debug, PartialEq, Eq)]
pub enum Cell {
    Dead = 0,
    Alive = 1,
}

#[wasm_bindgen]
impl Universe {
    pub fn new(width: u32, height: u32) -> Universe {
        let cells = (0..width * height)
            .map(|i| {
                if i % 2 == 0 || i % 7 == 0 {
                    Cell::Alive
                } else {
                    Cell::Dead
                }
            })
            .collect();

        Universe { width, height, cells }
    }

    pub fn width(&self) -> u32 {
        self.width
    }

    pub fn height(&self) -> u32 {
        self.height
    }

    pub fn cells(&self) -> *const Cell {
        self.cells.as_ptr()
    }

    /// 獲取細胞索引
    fn get_index(&self, row: u32, column: u32) -> usize {
        (row * self.width + column) as usize
    }

    /// 計算活鄰居數量
    fn live_neighbor_count(&self, row: u32, column: u32) -> u8 {
        let mut count = 0;

        for delta_row in [self.height - 1, 0, 1].iter().cloned() {
            for delta_col in [self.width - 1, 0, 1].iter().cloned() {
                if delta_row == 0 && delta_col == 0 {
                    continue;
                }

                let neighbor_row = (row + delta_row) % self.height;
                let neighbor_col = (column + delta_col) % self.width;
                let idx = self.get_index(neighbor_row, neighbor_col);
                count += self.cells[idx] as u8;
            }
        }
        count
    }

    /// 演化一步
    pub fn tick(&mut self) {
        let mut next = self.cells.clone();

        for row in 0..self.height {
            for col in 0..self.width {
                let idx = self.get_index(row, col);
                let cell = self.cells[idx];
                let live_neighbors = self.live_neighbor_count(row, col);

                let next_cell = match (cell, live_neighbors) {
                    // Rule 1: 少於兩個活鄰居，細胞死亡
                    (Cell::Alive, x) if x < 2 => Cell::Dead,
                    // Rule 2: 兩個或三個活鄰居，細胞存活
                    (Cell::Alive, 2) | (Cell::Alive, 3) => Cell::Alive,
                    // Rule 3: 超過三個活鄰居，細胞死亡
                    (Cell::Alive, x) if x > 3 => Cell::Dead,
                    // Rule 4: 恰好三個活鄰居，死細胞復活
                    (Cell::Dead, 3) => Cell::Alive,
                    // 其他情況保持不變
                    (otherwise, _) => otherwise,
                };

                next[idx] = next_cell;
            }
        }

        self.cells = next;
    }

    /// 切換細胞狀態
    pub fn toggle_cell(&mut self, row: u32, column: u32) {
        let idx = self.get_index(row, column);
        self.cells[idx] = match self.cells[idx] {
            Cell::Dead => Cell::Alive,
            Cell::Alive => Cell::Dead,
        };
    }

    /// 隨機初始化
    pub fn randomize(&mut self) {
        use js_sys::Math;
        
        self.cells = (0..self.width * self.height)
            .map(|_| {
                if Math::random() < 0.5 {
                    Cell::Alive
                } else {
                    Cell::Dead
                }
            })
            .collect();
    }

    /// 清空
    pub fn clear(&mut self) {
        self.cells = vec![Cell::Dead; (self.width * self.height) as usize];
    }
}

impl fmt::Display for Universe {
    fn fmt(&self, f: &mut fmt::Formatter) -> fmt::Result {
        for line in self.cells.as_slice().chunks(self.width as usize) {
            for &cell in line {
                let symbol = if cell == Cell::Dead { '◻' } else { '◼' };
                write!(f, "{}", symbol)?;
            }
            write!(f, "\n")?;
        }
        Ok(())
    }
}
```

### www/index.js

```javascript
import init, { Universe, Cell } from "./pkg/game_of_life.js";

const CELL_SIZE = 5; // px
const GRID_COLOR = "#CCCCCC";
const DEAD_COLOR = "#FFFFFF";
const ALIVE_COLOR = "#000000";

let universe, width, height, canvas, ctx, animationId = null;

async function run() {
    await init();

    // 創建宇宙
    width = 128;
    height = 128;
    universe = Universe.new(width, height);

    // 設置 canvas
    canvas = document.getElementById("game-of-life-canvas");
    canvas.height = (CELL_SIZE + 1) * height + 1;
    canvas.width = (CELL_SIZE + 1) * width + 1;
    ctx = canvas.getContext('2d');

    // 點擊切換細胞
    canvas.addEventListener("click", event => {
        const boundingRect = canvas.getBoundingClientRect();
        const scaleX = canvas.width / boundingRect.width;
        const scaleY = canvas.height / boundingRect.height;

        const canvasLeft = (event.clientX - boundingRect.left) * scaleX;
        const canvasTop = (event.clientY - boundingRect.top) * scaleY;

        const row = Math.min(Math.floor(canvasTop / (CELL_SIZE + 1)), height - 1);
        const col = Math.min(Math.floor(canvasLeft / (CELL_SIZE + 1)), width - 1);

        universe.toggle_cell(row, col);
        drawCells();
    });

    // 按鈕事件
    document.getElementById("play-pause").addEventListener("click", () => {
        if (isPaused()) {
            play();
        } else {
            pause();
        }
    });

    document.getElementById("step").addEventListener("click", () => {
        universe.tick();
        drawCells();
    });

    document.getElementById("random").addEventListener("click", () => {
        universe.randomize();
        drawCells();
    });

    document.getElementById("clear").addEventListener("click", () => {
        universe.clear();
        drawCells();
    });

    drawGrid();
    drawCells();
    play();
}

function isPaused() {
    return animationId === null;
}

function play() {
    document.getElementById("play-pause").textContent = "⏸";
    renderLoop();
}

function pause() {
    document.getElementById("play-pause").textContent = "▶";
    cancelAnimationFrame(animationId);
    animationId = null;
}

const fps = new class {
    constructor() {
        this.fps = document.getElementById("fps");
        this.frames = [];
        this.lastFrameTimeStamp = performance.now();
    }

    render() {
        const now = performance.now();
        const delta = now - this.lastFrameTimeStamp;
        this.lastFrameTimeStamp = now;
        const fps = 1 / delta * 1000;

        this.frames.push(fps);
        if (this.frames.length > 100) {
            this.frames.shift();
        }

        let sum = 0;
        for (let i = 0; i < this.frames.length; i++) {
            sum += this.frames[i];
        }
        let mean = sum / this.frames.length;

        this.fps.textContent = `FPS: ${Math.round(mean)}`;
    }
};

function renderLoop() {
    fps.render();

    universe.tick();
    drawCells();

    animationId = requestAnimationFrame(renderLoop);
}

function drawGrid() {
    ctx.beginPath();
    ctx.strokeStyle = GRID_COLOR;

    // 垂直線
    for (let i = 0; i <= width; i++) {
        ctx.moveTo(i * (CELL_SIZE + 1) + 1, 0);
        ctx.lineTo(i * (CELL_SIZE + 1) + 1, (CELL_SIZE + 1) * height + 1);
    }

    // 水平線
    for (let j = 0; j <= height; j++) {
        ctx.moveTo(0, j * (CELL_SIZE + 1) + 1);
        ctx.lineTo((CELL_SIZE + 1) * width + 1, j * (CELL_SIZE + 1) + 1);
    }

    ctx.stroke();
}

function getIndex(row, column) {
    return row * width + column;
}

function drawCells() {
    const cellsPtr = universe.cells();
    const cells = new Uint8Array(
        memory.buffer,
        cellsPtr,
        width * height
    );

    ctx.beginPath();

    for (let row = 0; row < height; row++) {
        for (let col = 0; col < width; col++) {
            const idx = getIndex(row, col);

            ctx.fillStyle = cells[idx] === Cell.Dead
                ? DEAD_COLOR
                : ALIVE_COLOR;

            ctx.fillRect(
                col * (CELL_SIZE + 1) + 1,
                row * (CELL_SIZE + 1) + 1,
                CELL_SIZE,
                CELL_SIZE
            );
        }
    }

    ctx.stroke();
}

run();
```

---

## 性能對比測試

創建 JavaScript 與 WASM 的性能對比：

```javascript
// JavaScript 版本的灰階濾鏡
function grayscaleJS(data) {
    for (let i = 0; i < data.length; i += 4) {
        const gray = 0.299 * data[i] + 0.587 * data[i + 1] + 0.114 * data[i + 2];
        data[i] = gray;
        data[i + 1] = gray;
        data[i + 2] = gray;
    }
}

// 性能測試
function benchmarkGrayscale(imageData) {
    const wasmData = new Uint8ClampedArray(imageData.data);
    const jsData = new Uint8ClampedArray(imageData.data);
    
    // WASM
    const wasmStart = performance.now();
    wasm.grayscale(wasmData);
    const wasmTime = performance.now() - wasmStart;
    
    // JavaScript
    const jsStart = performance.now();
    grayscaleJS(jsData);
    const jsTime = performance.now() - jsStart;
    
    console.log(`WASM: ${wasmTime.toFixed(2)}ms`);
    console.log(`JS: ${jsTime.toFixed(2)}ms`);
    console.log(`Speedup: ${(jsTime / wasmTime).toFixed(2)}x`);
}
```

典型結果（1920x1080 圖片）：
- JavaScript: ~15ms
- WASM: ~3ms
- **加速比: ~5x**

---

## 部署到生產環境

### 1. 優化構建

```bash
# 使用最激進的優化
wasm-pack build --target web --release

# 後處理優化
wasm-opt -Oz pkg/my_project_bg.wasm -o pkg/optimized.wasm
```

### 2. CDN 部署

```html
<!-- 從 CDN 載入 -->
<script type="module">
    import init from 'https://cdn.example.com/pkg/my_project.js';
    
    init().then(() => {
        console.log('WASM loaded from CDN');
    });
</script>
```

### 3. 懶載入

```javascript
async function loadWasmWhenNeeded() {
    const button = document.getElementById('heavy-computation');
    
    button.addEventListener('click', async () => {
        if (!window.wasmModule) {
            const { default: init, ...wasm } = await import('./pkg/my_project.js');
            await init();
            window.wasmModule = wasm;
        }
        
        // 使用 WASM
        window.wasmModule.heavy_function();
    });
}
```

---

## 參考資料

1. [Rust and WebAssembly Book](https://rustwasm.github.io/docs/book/)
2. [Game of Life Tutorial](https://rustwasm.github.io/docs/book/game-of-life/introduction.html)
3. [wasm-pack Template](https://github.com/rustwasm/wasm-pack-template)
4. [Made with WebAssembly](https://madewithwebassembly.com/)
5. [WebAssembly Studio](https://webassembly.studio/)
