# Rayon 數據並行 (Rayon Data Parallelism)

## 核心概念

Rayon 是 Rust 最流行的數據並行庫，提供簡單的 API 將順序計算轉為並行計算。

**核心特性**:
- 工作竊取 (Work-Stealing) 調度器
- 零成本抽象 (只需改 `iter()` 為 `par_iter()`)
- 數據競爭安全 (編譯期保證)
- 自動負載均衡

```toml
[dependencies]
rayon = "1.8"
```

---

## 並行迭代器

### 基本使用

```rust
use rayon::prelude::*;

fn main() {
    let numbers: Vec<i32> = (0..1_000_000).collect();
    
    // 順序迭代
    let sum_seq: i32 = numbers.iter().sum();
    
    // 並行迭代 (只需改為 par_iter)
    let sum_par: i32 = numbers.par_iter().sum();
    
    assert_eq!(sum_seq, sum_par);
}
```

### 常用操作

```rust
use rayon::prelude::*;

fn parallel_operations() {
    let data: Vec<i32> = (0..10000).collect();
    
    // map - 並行映射
    let doubled: Vec<i32> = data.par_iter()
        .map(|&x| x * 2)
        .collect();
    
    // filter - 並行過濾
    let evens: Vec<i32> = data.par_iter()
        .filter(|&&x| x % 2 == 0)
        .copied()
        .collect();
    
    // filter_map - 組合過濾和映射
    let results: Vec<i32> = data.par_iter()
        .filter_map(|&x| {
            if x % 2 == 0 {
                Some(x * 2)
            } else {
                None
            }
        })
        .collect();
    
    // for_each - 並行遍歷 (副作用操作)
    data.par_iter().for_each(|&x| {
        println!("Processing: {}", x);
    });
    
    // find_any - 並行查找 (返回任意匹配項)
    let found = data.par_iter().find_any(|&&x| x == 5000);
    
    // find_first - 並行查找 (返回第一個匹配項)
    let first = data.par_iter().find_first(|&&x| x > 100);
}
```

---

## 歸約操作

### fold 與 reduce

```rust
use rayon::prelude::*;

fn fold_reduce_examples() {
    let data: Vec<i32> = (1..=1000).collect();
    
    // reduce - 簡單歸約
    let sum = data.par_iter()
        .copied()
        .reduce(|| 0, |a, b| a + b);
    
    // fold - 每個線程獨立累加，最後合併
    let sum = data.par_iter()
        .fold(|| 0, |acc, &x| acc + x)
        .sum::<i32>();
    
    // fold_with - 指定初始值
    let product = data.par_iter()
        .fold_with(1, |acc, &x| acc * x)
        .reduce(|| 1, |a, b| a * b);
    
    // try_fold - 可失敗的 fold
    let result: Result<i32, &str> = data.par_iter()
        .try_fold(
            || 0,
            |acc, &x| {
                if x > 500 {
                    Err("Value too large")
                } else {
                    Ok(acc + x)
                }
            }
        )
        .try_reduce(|| 0, |a, b| Ok(a + b));
}
```

### 自定義歸約

```rust
use rayon::prelude::*;
use std::collections::HashMap;

fn custom_reduce() {
    let words = vec!["hello", "world", "hello", "rust", "world"];
    
    // 並行統計詞頻
    let word_count: HashMap<&str, usize> = words
        .par_iter()
        .fold(
            || HashMap::new(),
            |mut map, &word| {
                *map.entry(word).or_insert(0) += 1;
                map
            }
        )
        .reduce(
            || HashMap::new(),
            |mut map1, map2| {
                for (word, count) in map2 {
                    *map1.entry(word).or_insert(0) += count;
                }
                map1
            }
        );
    
    for (word, count) in word_count {
        println!("{}: {}", word, count);
    }
}
```

---

## 並行排序

### 各種排序變體

```rust
use rayon::prelude::*;

fn parallel_sorting() {
    let mut data: Vec<i32> = (0..100000).rev().collect();
    
    // 並行不穩定排序 (最快)
    data.par_sort_unstable();
    
    // 並行穩定排序
    data.par_sort();
    
    // 自定義比較器
    data.par_sort_by(|a, b| b.cmp(a));  // 降序
    
    // 按鍵排序
    let mut people = vec![
        ("Alice", 30),
        ("Bob", 25),
        ("Charlie", 35),
    ];
    people.par_sort_by_key(|&(_, age)| age);
}
```

### 性能對比

```rust
use rayon::prelude::*;
use std::time::Instant;

fn benchmark_sorting() {
    let data: Vec<i32> = (0..10_000_000).rev().collect();
    
    // 順序排序
    let mut seq_data = data.clone();
    let start = Instant::now();
    seq_data.sort_unstable();
    println!("Sequential: {:?}", start.elapsed());
    
    // 並行排序
    let mut par_data = data.clone();
    let start = Instant::now();
    par_data.par_sort_unstable();
    println!("Parallel: {:?}", start.elapsed());
    
    // 典型結果 (8 核):
    // Sequential: ~800ms
    // Parallel:   ~150ms (5x 加速)
}
```

---

## 分塊處理

### chunks 與 chunks_exact

```rust
use rayon::prelude::*;

fn chunk_processing() {
    let data = vec![0u8; 10_000_000];
    
    // 並行處理塊 (每塊可能大小不同)
    let checksums: Vec<u32> = data.par_chunks(1024)
        .map(|chunk| {
            chunk.iter().map(|&b| b as u32).sum()
        })
        .collect();
    
    // 並行處理固定大小塊
    let (chunks, remainder) = data.as_chunks::<1024>();
    let sums: Vec<u32> = chunks.par_iter()
        .map(|chunk| {
            chunk.iter().map(|&b| b as u32).sum()
        })
        .collect();
    
    // 可變塊處理
    let mut data = vec![0i32; 10000];
    data.par_chunks_mut(100).for_each(|chunk| {
        for x in chunk {
            *x += 1;
        }
    });
}
```

---

## 任務並行

### scope 與 spawn

```rust
use rayon;

fn task_parallelism() {
    let mut data1 = vec![1, 2, 3, 4, 5];
    let mut data2 = vec![6, 7, 8, 9, 10];
    
    rayon::scope(|s| {
        // 並行執行兩個任務
        s.spawn(|_| {
            data1.iter_mut().for_each(|x| *x *= 2);
        });
        
        s.spawn(|_| {
            data2.iter_mut().for_each(|x| *x *= 3);
        });
    });  // scope 結束時自動等待所有任務完成
    
    println!("Data1: {:?}", data1);
    println!("Data2: {:?}", data2);
}
```

### join - 分治法

```rust
use rayon;

fn quicksort<T: Ord + Send>(data: &mut [T]) {
    if data.len() <= 1 {
        return;
    }
    
    let pivot_index = partition(data);
    let (left, right) = data.split_at_mut(pivot_index);
    
    // 並行處理左右子數組
    rayon::join(
        || quicksort(left),
        || quicksort(right)
    );
}

fn partition<T: Ord>(data: &mut [T]) -> usize {
    let len = data.len();
    let pivot_index = len / 2;
    data.swap(pivot_index, len - 1);
    
    let mut i = 0;
    for j in 0..len - 1 {
        if data[j] <= data[len - 1] {
            data.swap(i, j);
            i += 1;
        }
    }
    data.swap(i, len - 1);
    i
}
```

---

## 實戰案例

### 案例 1：圖像處理

```rust
use rayon::prelude::*;

fn gaussian_blur_parallel(image: &[u8], width: usize, height: usize) -> Vec<u8> {
    let kernel = [
        [1, 2, 1],
        [2, 4, 2],
        [1, 2, 1],
    ];
    let kernel_sum: i32 = 16;
    
    // 並行處理每一行
    (0..height).into_par_iter().flat_map(|y| {
        (0..width).map(move |x| {
            let mut sum = 0i32;
            
            for ky in 0..3 {
                for kx in 0..3 {
                    let py = (y as i32 + ky - 1).clamp(0, height as i32 - 1) as usize;
                    let px = (x as i32 + kx - 1).clamp(0, width as i32 - 1) as usize;
                    sum += image[py * width + px] as i32 * kernel[ky as usize][kx as usize];
                }
            }
            
            (sum / kernel_sum) as u8
        })
    }).collect()
}

fn main() {
    let width = 1920;
    let height = 1080;
    let image = vec![128u8; width * height];
    
    let blurred = gaussian_blur_parallel(&image, width, height);
    println!("Processed {} pixels", blurred.len());
}
```

### 案例 2：矩陣乘法

```rust
use rayon::prelude::*;

fn matmul_parallel(a: &[f32], b: &[f32], n: usize) -> Vec<f32> {
    let mut c = vec![0.0; n * n];
    
    // 並行處理每一行
    c.par_chunks_mut(n)
        .enumerate()
        .for_each(|(i, row)| {
            for j in 0..n {
                let mut sum = 0.0;
                for k in 0..n {
                    sum += a[i * n + k] * b[k * n + j];
                }
                row[j] = sum;
            }
        });
    
    c
}

// 性能提升: 512x512 矩陣，8 核 CPU 約 6-7x 加速
```

### 案例 3：文本搜索

```rust
use rayon::prelude::*;
use std::fs;
use std::path::Path;

fn parallel_grep(pattern: &str, dir: &Path) -> Vec<(String, usize)> {
    let entries: Vec<_> = fs::read_dir(dir)
        .unwrap()
        .filter_map(|e| e.ok())
        .collect();
    
    entries
        .par_iter()
        .filter_map(|entry| {
            let path = entry.path();
            if path.is_file() {
                if let Ok(content) = fs::read_to_string(&path) {
                    let count = content.lines()
                        .filter(|line| line.contains(pattern))
                        .count();
                    
                    if count > 0 {
                        return Some((path.display().to_string(), count));
                    }
                }
            }
            None
        })
        .collect()
}
```

### 案例 4：數據分析

```rust
use rayon::prelude::*;

#[derive(Debug)]
struct Statistics {
    mean: f64,
    variance: f64,
    min: f64,
    max: f64,
}

fn compute_statistics_parallel(data: &[f64]) -> Statistics {
    let (sum, sum_sq, min, max) = data.par_iter()
        .fold(
            || (0.0, 0.0, f64::INFINITY, f64::NEG_INFINITY),
            |(sum, sum_sq, min, max), &x| {
                (
                    sum + x,
                    sum_sq + x * x,
                    min.min(x),
                    max.max(x),
                )
            }
        )
        .reduce(
            || (0.0, 0.0, f64::INFINITY, f64::NEG_INFINITY),
            |(s1, sq1, min1, max1), (s2, sq2, min2, max2)| {
                (
                    s1 + s2,
                    sq1 + sq2,
                    min1.min(min2),
                    max1.max(max2),
                )
            }
        );
    
    let n = data.len() as f64;
    let mean = sum / n;
    let variance = (sum_sq / n) - (mean * mean);
    
    Statistics { mean, variance, min, max }
}
```

---

## 線程池配置

### 全局配置

```rust
use rayon::ThreadPoolBuilder;

fn main() {
    // 配置全局線程池
    ThreadPoolBuilder::new()
        .num_threads(8)                        // 8 個工作線程
        .stack_size(2 * 1024 * 1024)          // 2MB 棧大小
        .thread_name(|i| format!("rayon-{}", i))
        .build_global()
        .unwrap();
    
    // 使用全局線程池
    let sum: i32 = (0..1000).into_par_iter().sum();
    println!("Sum: {}", sum);
}
```

### 自定義線程池

```rust
use rayon::ThreadPoolBuilder;

fn main() {
    // 創建自定義線程池
    let pool = ThreadPoolBuilder::new()
        .num_threads(4)
        .build()
        .unwrap();
    
    // 在特定線程池中執行
    let result = pool.install(|| {
        (0..1000).into_par_iter().sum::<i32>()
    });
    
    println!("Result: {}", result);
}
```

---

## 性能優化

### 1. 控制粒度

```rust
use rayon::prelude::*;

// ❌ 粒度太小：任務切分開銷大
fn bad_granularity(data: &[i32]) -> i32 {
    data.par_iter().map(|&x| x * 2).sum()
}

// ✅ 使用 chunks 增加粒度
fn good_granularity(data: &[i32]) -> i32 {
    data.par_chunks(1000)
        .map(|chunk| chunk.iter().map(|&x| x * 2).sum::<i32>())
        .sum()
}
```

### 2. 避免小數據集並行

```rust
use rayon::prelude::*;

fn smart_parallel(data: &[i32]) -> i32 {
    const THRESHOLD: usize = 10000;
    
    if data.len() < THRESHOLD {
        data.iter().sum()  // 順序處理
    } else {
        data.par_iter().sum()  // 並行處理
    }
}
```

### 3. with_min_len - 動態調整粒度

```rust
use rayon::prelude::*;

fn adaptive_parallel(data: &[i32]) -> i32 {
    data.par_iter()
        .with_min_len(1000)  // 每個任務至少 1000 個元素
        .sum()
}
```

---

## Benchmark

```rust
use criterion::{black_box, criterion_group, criterion_main, Criterion};
use rayon::prelude::*;

fn sequential(data: &[i32]) -> i32 {
    data.iter().map(|&x| expensive(x)).sum()
}

fn parallel(data: &[i32]) -> i32 {
    data.par_iter().map(|&x| expensive(x)).sum()
}

fn expensive(x: i32) -> i32 {
    (0..100).fold(x, |acc, i| acc.wrapping_mul(i))
}

fn benchmark(c: &mut Criterion) {
    let data: Vec<i32> = (0..10000).collect();
    
    c.bench_function("sequential", |b| {
        b.iter(|| sequential(black_box(&data)))
    });
    
    c.bench_function("parallel", |b| {
        b.iter(|| parallel(black_box(&data)))
    });
}

criterion_group!(benches, benchmark);
criterion_main!(benches);
```

---

## 參考資料 (References)

1. [Rayon Documentation](https://docs.rs/rayon/latest/rayon/)
2. [Rayon FAQ](https://github.com/rayon-rs/rayon/blob/master/FAQ.md)
3. [Data Parallelism in Rust](https://blog.rayon-rs.org/)
4. [The Rayon Book](https://smallcultfollowing.com/babysteps/blog/2015/12/18/rayon-data-parallelism-in-rust/)
5. 《Programming Rust》 (O'Reilly, 2021) - Chapter 19
