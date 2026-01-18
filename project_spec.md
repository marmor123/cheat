# PulseCheck: Project Specification Document
**Version:** 1.0  
**Status:** Awaiting Validation  
**Last Updated:** 2026-01-18

---

## PART 1: System Overview & Prime Directive

### Persona
**Principal Creative Technologist & Biomedical Systems Architect**
- Specialization: High-Performance Browser Engineering
- Domain Expertise: Computer Vision (WebGPU), Systems Programming (Rust/Wasm), Game Design

### Prime Directive
⚠️ **CRITICAL RULE**: No application code shall be generated until this specification is validated and locked.

This document serves as the **immutable "Source of Truth"** for the PulseCheck project. All implementation decisions must trace back to this specification.

---

## PART 2: Product Vision (The "Why")

### Overview
**PulseCheck** is a high-stakes, multiplayer adaptation of the card game "Cheat" that uses biological feedback to create a unique competitive experience.

### Core Mechanics

#### The Hook
Remote Photoplethysmography (rPPG) technology exposes players' biological reactions to their opponents in real-time, turning human physiology into a strategic game element.

#### The Operator Paradigm
- **No Avatars**: Players are treated as operators in a high-tech surveillance environment
- **Tactical Bio-HUD**: Each player gets a real-time biometric dashboard
- **Raw Data Philosophy**: Display authentic metrics instead of simplified emotional indicators

#### The Data Strategy
Players receive unfiltered biometric data, forcing analytical interpretation:

**Visual Components:**
- **Sparkline Graphs**: Heart Rate trends over the last 10 seconds
- **Real-time BPM Readout**: Current beats per minute
- **Signal Stability Gauge**: Quality indicator for rPPG readings
- **Ambient Atmosphere**: Collective stress influences game lighting/music intensity

#### The Psychological "Tell"
Players must learn individual baselines to detect deception:
- BPM spike could indicate lying OR excitement
- Ambiguity is intentional - forces strategic interpretation
- Pattern recognition becomes a meta-skill

---

## PART 3: Blind Bio-Feedback Architecture

### Technical Constraints (STRICT)

#### 1. Privacy (The "Black Box")
**Rule**: Webcam video stream NEVER leaves the client device.

**Implementation:**
- Video captured and processed locally within Web Worker
- MediaPipe face detection runs entirely client-side
- Only derived telemetry transmitted to server: `{ bpm: 80, stress_index: 0.45 }`
- Raw frames destroyed immediately after processing

**Rationale**: GDPR compliance, bandwidth optimization, trust-building

#### 2. Performance (The "Main Thread" Rule)
**Rule**: UI must maintain 60fps without jank.

**Implementation:**
- All Computer Vision (MediaPipe) executed in Web Worker
- Signal Processing (FFT/Filtering) offloaded to Worker
- OffscreenCanvas for all CV rendering
- Main thread reserved for React UI updates only

**Rationale**: Browser event loop protection, smooth UX

#### 3. Robustness (The "Fallback" Matrix)
**Rule**: Graceful degradation across hardware capabilities.

**Primary Path:**
```
WebGPU + ONNX Runtime → EfficientPhys Neural Network
```

**Fallback Path:**
```
WebAssembly (Rust) → POS Algorithm (Plane-Orthogonal-to-Skin)
```

**Detection Logic:**
```typescript
if (navigator.gpu && await checkWebGPUSupport()) {
  engine = new ONNXEngine();
} else {
  engine = new WasmPOSEngine();
}
```

### Hybrid Tech Stack (STRICT)

#### Orchestration Layer
- **React 19**: Latest features (use, useOptimistic for multiplayer state)
- **Vite**: Fast HMR, ESM-first bundling
- **TypeScript**: Strict mode, no implicit any

#### Vision Layer (The Eye)
- **Package**: `@mediapipe/face_mesh`
- **Execution Context**: Web Worker (via Comlink)
- **Responsibility**: Face detection, facial landmark extraction
- **Output**: ROI (Region of Interest) coordinates for forehead/cheeks

#### Inference Layer (The Brain)
- **Package**: `onnxruntime-web`
- **Backend**: `webgpu` (primary), `wasm` (fallback)
- **Model**: EfficientPhys (pre-trained ONNX format)
- **Input**: 10-second rolling window of RGB averages from facial ROI
- **Output**: Estimated BPM

#### Math Layer (The Filter)
**Package**: Custom Rust crate compiled via `wasm-pack`

**Responsibilities:**
1. **Ring Buffer**: Circular buffer for signal history (10s @ 30fps = 300 samples)
2. **2nd-Order Butterworth Bandpass Filter**:
   - Low cutoff: 0.75 Hz (45 BPM)
   - High cutoff: 3.0 Hz (180 BPM)
   - Rationale: Isolates cardiac frequency band
3. **Cubic Spline Interpolation**: Upsamples to 60fps for smooth sparkline rendering

**Crates:**
```toml
[dependencies]
biquad = "0.4"
rustfft = "6.0"
wasm-bindgen = "0.2"
```

#### Networking Layer
- **Package**: `socket.io-client`
- **Protocol**: WebSocket with fallback to long-polling
- **Payload**: JSON telemetry packets at 1 Hz

---

## PART 4: SpecKit Requirements

### 4.1 The "Zero-Copy" Data Pipeline

#### Problem Statement
Video frames are large (1920×1080×4 bytes = 8.3 MB). Copying frame data from Main Thread to Worker creates memory pressure and GC pauses.

#### Solution Architecture
Use **Transferable Objects** (specifically `ImageBitmap`) to transfer ownership without copying.

#### Implementation Diagram
```
Main Thread                          Web Worker
───────────                          ──────────
<video>                              
   │
   ├─ requestAnimationFrame()
   │    │
   │    ├─ createImageBitmap(video)
   │    │     │
   │    │     └─ ImageBitmap (heap allocated)
   │    │
   │    └─ postMessage(bitmap, [bitmap])  ──┐
   │                                         │ TRANSFER (zero-copy)
   │                                         │
   │                                    ┌────┘
   │                                    │
   │                               onmessage(bitmap)
   │                                    │
   │                                    ├─ MediaPipe.process(bitmap)
   │                                    ├─ Extract ROI RGB
   │                                    ├─ FFT → BPM
   │                                    └─ postMessage({ bpm, quality })
   │                                         │
   ├──────────────────────────────────────────┘
   │
   └─ Update React state
```

#### Code Contract
**Main Thread:**
```typescript
const bitmap = await createImageBitmap(videoElement);
worker.postMessage({ type: 'FRAME', bitmap }, [bitmap]);
// bitmap is now neutered (unusable in main thread)
```

**Worker:**
```typescript
self.onmessage = async (e: MessageEvent) => {
  if (e.data.type === 'FRAME') {
    const bitmap: ImageBitmap = e.data.bitmap;
    const telemetry = await processFrame(bitmap);
    bitmap.close(); // Explicit cleanup
    self.postMessage({ type: 'TELEMETRY', data: telemetry });
  }
};
```

#### Performance Guarantee
- **Target**: Process 30fps stream with <5ms overhead per frame
- **Measurement**: Use `performance.mark()` to validate zero-copy path

---

### 4.2 The BioSensor Hook Interface

#### TypeScript Contract
```typescript
/**
 * React hook for accessing real-time biometric telemetry.
 * Manages Web Worker lifecycle and state synchronization.
 */
export interface UseBioSensorReturn {
  /** Current beats per minute (smoothed, 1s window) */
  bpm: number;

  /** 
   * Historical BPM samples for visualization.
   * Length: 300 samples (10s @ 30fps)
   * Type: Float32Array for WebGL compatibility
   */
  history: Float32Array;

  /** 
   * Signal-to-Noise Ratio (0.0 - 1.0)
   * <0.3: Poor (motion artifacts, low light)
   * 0.3-0.7: Fair
   * >0.7: Good
   */
  signal_quality: number;

  /** 
   * State machine status
   */
  status: 'CALIBRATING' | 'MEASURING' | 'OBSCURED';
}

export interface UseBioSensorOptions {
  /** Enable synthetic data injection for testing */
  synthetic?: {
    /** Pre-recorded RGB sequence (JSON format) */
    data: RGBSample[];
    /** Playback speed multiplier */
    speed?: number;
  };

  /** Webcam constraints */
  video?: MediaTrackConstraints;

  /** Callback for telemetry updates (for logging/debugging) */
  onTelemetry?: (t: TelemetryPacket) => void;
}

export function useBioSensor(
  options?: UseBioSensorOptions
): UseBioSensorReturn;
```

#### Usage Example
```typescript
function PlayerCard({ playerId }: { playerId: string }) {
  const { bpm, history, signal_quality, status } = useBioSensor({
    video: { width: 640, height: 480, frameRate: 30 }
  });

  return (
    <div className="bio-hud">
      <div className="metric">
        <span className="label">BPM</span>
        <span className="value">{bpm}</span>
      </div>
      <Sparkline data={history} />
      <SignalStrength quality={signal_quality} />
      <StatusIndicator status={status} />
    </div>
  );
}
```

---

### 4.3 Synthetic Testing Strategy

#### Problem Statement
Testing signal processing algorithms requires:
1. Deterministic input (eliminates camera variability)
2. Ground truth labels (known BPM for validation)
3. Fast iteration (no need for camera setup)

#### Solution: Synthetic Injection Mode

#### Data Format
```typescript
interface RGBSample {
  /** Timestamp in milliseconds */
  timestamp: number;
  
  /** Average red channel value (0-255) from facial ROI */
  red: number;
  
  /** Average green channel value (0-255) from facial ROI */
  green: number;
  
  /** Average blue channel value (0-255) from facial ROI */
  blue: number;
  
  /** Ground truth BPM (for validation) */
  ground_truth_bpm?: number;
}

interface SyntheticSequence {
  /** Human-readable description */
  description: string;
  
  /** Expected BPM (for assertion) */
  expected_bpm: number;
  
  /** Array of RGB samples */
  samples: RGBSample[];
}
```

#### Example Synthetic File (`test_data/resting_75bpm.json`)
```json
{
  "description": "Resting state, 75 BPM, good lighting",
  "expected_bpm": 75,
  "samples": [
    { "timestamp": 0, "red": 180, "green": 120, "blue": 100 },
    { "timestamp": 33, "red": 182, "green": 121, "blue": 101 },
    ...
  ]
}
```

#### Test Harness
```typescript
import restingData from './test_data/resting_75bpm.json';

test('Rust filter correctly extracts 75 BPM from synthetic signal', async () => {
  const { bpm, history } = useBioSensor({
    synthetic: { data: restingData.samples }
  });

  // Wait for calibration (3 seconds)
  await waitFor(() => expect(status).toBe('MEASURING'));

  // Assert BPM within 5% tolerance
  expect(bpm).toBeGreaterThan(71);
  expect(bpm).toBeLessThan(79);

  // Validate FFT peaks align with expected frequency
  const fft = computeFFT(history);
  const peakFreq = findDominantFrequency(fft);
  expect(peakFreq).toBeCloseTo(75 / 60, 1); // 1.25 Hz
});
```

#### Test Data Generation Script
```python
# tools/generate_synthetic_signal.py
import numpy as np
import json

def generate_ppg_signal(duration_s, bpm, noise_level=0.05):
    """Generate synthetic PPG signal with realistic artifacts"""
    sample_rate = 30  # fps
    samples = duration_s * sample_rate
    
    # Cardiac component (sinusoidal approximation)
    freq = bpm / 60.0
    t = np.linspace(0, duration_s, samples)
    cardiac = np.sin(2 * np.pi * freq * t)
    
    # Add respiratory artifact (0.25 Hz)
    respiratory = 0.3 * np.sin(2 * np.pi * 0.25 * t)
    
    # Add noise
    noise = noise_level * np.random.randn(samples)
    
    # Combine and scale to RGB range (green channel is most sensitive)
    signal = cardiac + respiratory + noise
    green = 120 + 20 * signal
    red = 180 + 5 * signal  # Red channel has weaker signal
    blue = 100 + 2 * signal
    
    return [
        {
            "timestamp": int(i * 1000 / sample_rate),
            "red": int(np.clip(red[i], 0, 255)),
            "green": int(np.clip(green[i], 0, 255)),
            "blue": int(np.clip(blue[i], 0, 255))
        }
        for i in range(samples)
    ]

# Generate test cases
test_cases = [
    ("resting_75bpm.json", 75, 10),
    ("exercised_120bpm.json", 120, 10),
    ("stressed_95bpm.json", 95, 15),
]

for filename, bpm, duration in test_cases:
    data = {
        "description": f"{bpm} BPM synthetic signal",
        "expected_bpm": bpm,
        "samples": generate_ppg_signal(duration, bpm)
    }
    with open(f"test_data/{filename}", "w") as f:
        json.dump(data, f, indent=2)
```

---

### 4.4 Step-by-Step Implementation Plan

#### Phase 1: The Rust Core (Week 1)

**Objective**: Build and validate the signal processing Wasm module.

**Tasks:**
1. **Initialize Rust Project**
   ```bash
   cargo new --lib ppg-wasm
   cd ppg-wasm
   cargo add wasm-bindgen
   cargo add biquad    # For Butterworth filter
   cargo add rustfft   # For FFT
   ```

2. **Implement Ring Buffer**
   ```rust
   #[wasm_bindgen]
   pub struct SignalBuffer {
       data: Vec<f32>,
       capacity: usize,
       write_index: usize,
   }
   
   #[wasm_bindgen]
   impl SignalBuffer {
       pub fn new(capacity: usize) -> Self { /* ... */ }
       pub fn push(&mut self, value: f32) { /* ... */ }
       pub fn as_slice(&self) -> Vec<f32> { /* ... */ }
   }
   ```

3. **Implement Butterworth Bandpass Filter**
   ```rust
   use biquad::{Biquad, Coefficients, DirectForm2Transposed, Hertz, Q_BUTTERWORTH_F32, Type};
   
   #[wasm_bindgen]
   pub struct ButterworthFilter {
       low_pass: DirectForm2Transposed<f32>,
       high_pass: DirectForm2Transposed<f32>,
   }
   
   #[wasm_bindgen]
   impl ButterworthFilter {
       pub fn new(sample_rate: f32, low_cutoff: f32, high_cutoff: f32) -> Self {
           // 45 BPM = 0.75 Hz, 180 BPM = 3.0 Hz
           let fs = Hertz::<f32>::from_hz(sample_rate).unwrap();
           let low = Hertz::<f32>::from_hz(low_cutoff).unwrap();
           let high = Hertz::<f32>::from_hz(high_cutoff).unwrap();
           
           let coeffs_low = Coefficients::<f32>::from_params(
               Type::LowPass, fs, low, Q_BUTTERWORTH_F32
           ).unwrap();
           let coeffs_high = Coefficients::<f32>::from_params(
               Type::HighPass, fs, high, Q_BUTTERWORTH_F32
           ).unwrap();
           
           Self {
               low_pass: DirectForm2Transposed::<f32>::new(coeffs_low),
               high_pass: DirectForm2Transposed::<f32>::new(coeffs_high),
           }
       }
       
       pub fn process(&mut self, signal: &mut [f32]) {
           for sample in signal.iter_mut() {
               *sample = self.high_pass.run(*sample);
               *sample = self.low_pass.run(*sample);
           }
       }
   }
   ```

4. **Implement FFT-based BPM Estimator**
   ```rust
   use rustfft::FftPlanner;
   use rustfft::num_complex::Complex;
   
   #[wasm_bindgen]
   pub fn estimate_bpm(signal: &[f32], sample_rate: f32) -> f32 {
       let mut planner = FftPlanner::new();
       let fft = planner.plan_fft_forward(signal.len());
       
       let mut buffer: Vec<Complex<f32>> = signal.iter()
           .map(|&x| Complex { re: x, im: 0.0 })
           .collect();
       
       fft.process(&mut buffer);
       
       // Find peak frequency in cardiac band (0.75-3.0 Hz)
       let (peak_idx, _) = buffer.iter()
           .enumerate()
           .skip((0.75 * signal.len() as f32 / sample_rate) as usize)
           .take((3.0 * signal.len() as f32 / sample_rate) as usize)
           .max_by(|(_, a), (_, b)| {
               a.norm().partial_cmp(&b.norm()).unwrap()
           })
           .unwrap();
       
       let freq = peak_idx as f32 * sample_rate / signal.len() as f32;
       freq * 60.0  // Convert Hz to BPM
   }
   ```

5. **Build and Test**
   ```bash
   wasm-pack build --target web --out-dir pkg
   wasm-pack test --headless --firefox
   ```

**Deliverables:**
- `ppg-wasm/pkg/` directory with compiled Wasm + TypeScript bindings
- Unit tests validating filter response and FFT accuracy

**Success Criteria:**
- FFT correctly identifies 75 BPM sinusoid within ±2 BPM
- Butterworth filter attenuates 0.2 Hz signal by >20 dB

---

#### Phase 2: The Worker Loop (Week 2)

**Objective**: Implement MediaPipe face detection in a Web Worker.

**Tasks:**
1. **Set Up Worker with Comlink**
   ```typescript
   // workers/bio-worker.ts
   import { expose } from 'comlink';
   import { FaceMesh } from '@mediapipe/face_mesh';
   import * as ppgWasm from '../ppg-wasm/pkg';
   
   class BioWorker {
     private faceMesh: FaceMesh;
     private filter: ppgWasm.ButterworthFilter;
     private buffer: ppgWasm.SignalBuffer;
     
     async init() {
       await ppgWasm.default();  // Load Wasm
       
       this.faceMesh = new FaceMesh({
         locateFile: (file) => `https://cdn.jsdelivr.net/npm/@mediapipe/face_mesh/${file}`
       });
       
       this.faceMesh.setOptions({
         maxNumFaces: 1,
         refineLandmarks: false,
         minDetectionConfidence: 0.5,
       });
       
       this.filter = ppgWasm.ButterworthFilter.new(30.0, 0.75, 3.0);
       this.buffer = ppgWasm.SignalBuffer.new(300);  // 10s @ 30fps
     }
     
     async processFrame(bitmap: ImageBitmap) {
       const results = await this.faceMesh.send({ image: bitmap });
       
       if (!results.multiFaceLandmarks || results.multiFaceLandmarks.length === 0) {
         return { status: 'OBSCURED' };
       }
       
       const roi = this.extractROI(bitmap, results.multiFaceLandmarks[0]);
       const rgb = this.computeAverageRGB(roi);
       
       this.buffer.push(rgb.green);  // Green channel has strongest signal
       
       if (this.buffer.length < 90) {  // Need 3s for calibration
         return { status: 'CALIBRATING', progress: this.buffer.length / 90 };
       }
       
       const signal = new Float32Array(this.buffer.as_slice());
       this.filter.process(signal);
       
       const bpm = ppgWasm.estimate_bpm(signal, 30.0);
       const quality = this.estimateSignalQuality(signal);
       
       return {
         status: 'MEASURING',
         bpm,
         quality,
         history: signal,
       };
     }
     
     private extractROI(bitmap: ImageBitmap, landmarks: any[]): ImageData {
       // Use forehead region (landmarks 10, 338, 297, 332)
       // This region has good blood flow and minimal movement
       const canvas = new OffscreenCanvas(bitmap.width, bitmap.height);
       const ctx = canvas.getContext('2d')!;
       ctx.drawImage(bitmap, 0, 0);
       
       // Extract bounding box around forehead landmarks
       const foreheadIndices = [10, 338, 297, 332, 284, 251, 389, 356];
       const xs = foreheadIndices.map(i => landmarks[i].x * bitmap.width);
       const ys = foreheadIndices.map(i => landmarks[i].y * bitmap.height);
       
       const x = Math.min(...xs);
       const y = Math.min(...ys);
       const w = Math.max(...xs) - x;
       const h = Math.max(...ys) - y;
       
       return ctx.getImageData(x, y, w, h);
     }
     
     private computeAverageRGB(roi: ImageData): { red: number; green: number; blue: number } {
       let r = 0, g = 0, b = 0;
       const pixels = roi.data.length / 4;
       
       for (let i = 0; i < roi.data.length; i += 4) {
         r += roi.data[i];
         g += roi.data[i + 1];
         b += roi.data[i + 2];
       }
       
       return {
         red: r / pixels,
         green: g / pixels,
         blue: b / pixels,
       };
     }
     
     private estimateSignalQuality(signal: Float32Array): number {
       // Compute SNR: ratio of cardiac band power to total power
       const fft = /* ... compute FFT ... */;
       const cardiacPower = /* ... sum power in 0.75-3.0 Hz ... */;
       const totalPower = /* ... sum total power ... */;
       return cardiacPower / totalPower;
     }
   }
   
   expose(new BioWorker());
   ```

2. **Main Thread Integration**
   ```typescript
   // hooks/useBioSensor.ts
   import { wrap } from 'comlink';
   import { useEffect, useState, useRef } from 'react';
   
   export function useBioSensor(options?: UseBioSensorOptions): UseBioSensorReturn {
     const [state, setState] = useState<UseBioSensorReturn>({
       bpm: 0,
       history: new Float32Array(300),
       signal_quality: 0,
       status: 'CALIBRATING',
     });
     
     const workerRef = useRef<Worker>();
     const videoRef = useRef<HTMLVideoElement>();
     
     useEffect(() => {
       const worker = new Worker(
         new URL('../workers/bio-worker.ts', import.meta.url),
         { type: 'module' }
       );
       
       const bioWorker = wrap<BioWorker>(worker);
       await bioWorker.init();
       
       const video = document.createElement('video');
       const stream = await navigator.mediaDevices.getUserMedia({
         video: options?.video ?? { width: 640, height: 480, frameRate: 30 }
       });
       video.srcObject = stream;
       await video.play();
       
       const processLoop = async () => {
         const bitmap = await createImageBitmap(video);
         const result = await bioWorker.processFrame(bitmap);
         
         setState(prev => ({
           ...prev,
           ...result,
         }));
         
         requestAnimationFrame(processLoop);
       };
       
       processLoop();
       
       return () => {
         worker.terminate();
         stream.getTracks().forEach(t => t.stop());
       };
     }, []);
     
     return state;
   }
   ```

**Deliverables:**
- Working Web Worker with MediaPipe integration
- Zero-copy frame transfer using ImageBitmap
- React hook exposing telemetry state

**Success Criteria:**
- Main thread maintains 60fps during video processing
- Worker processes 30fps video stream with <10ms latency
- BPM readings stable within ±5 BPM on resting subject

---

#### Phase 3: The Neural Bridge (Week 3)

**Objective**: Integrate ONNX Runtime with WebGPU backend.

**Tasks:**
1. **Load EfficientPhys Model**
   ```typescript
   // utils/onnx-engine.ts
   import * as ort from 'onnxruntime-web';
   
   export class ONNXEngine {
     private session: ort.InferenceSession;
     
     async init() {
       // Configure WebGPU backend
       ort.env.wasm.numThreads = 1;
       ort.env.wasm.simd = true;
       
       if (navigator.gpu) {
         await ort.env.webgpu.initWebGPU();
         this.session = await ort.InferenceSession.create(
           '/models/efficientphys.onnx',
           { executionProviders: ['webgpu'] }
         );
       } else {
         this.session = await ort.InferenceSession.create(
           '/models/efficientphys.onnx',
           { executionProviders: ['wasm'] }
         );
       }
     }
     
     async estimateBPM(rgbSequence: Float32Array): Promise<number> {
       // EfficientPhys expects input shape: [1, 3, 300, 1]
       // (batch, channels, time_steps, spatial)
       const input = new ort.Tensor('float32', rgbSequence, [1, 3, 300, 1]);
       
       const outputs = await this.session.run({ input });
       const bpm = outputs.bpm.data[0] as number;
       
       return bpm;
     }
   }
   ```

2. **Integrate with Worker**
   ```typescript
   // In BioWorker class
   private engine: ONNXEngine | WasmPOSEngine;
   
   async init() {
     // Try WebGPU path first
     try {
       this.engine = new ONNXEngine();
       await this.engine.init();
       console.log('Using ONNX Runtime with WebGPU');
     } catch (e) {
       this.engine = new WasmPOSEngine();
       await this.engine.init();
       console.warn('Falling back to Wasm POS algorithm');
     }
     // ... rest of init
   }
   ```

3. **Implement POS Fallback**
   ```rust
   // In ppg-wasm crate
   #[wasm_bindgen]
   pub fn pos_algorithm(rgb_sequence: &[f32]) -> f32 {
       // Plane-Orthogonal-to-Skin algorithm
       // Reference: Wang et al., "Algorithmic Principles of Remote PPG"
       
       let n = rgb_sequence.len() / 3;
       let mut red = vec![0.0; n];
       let mut green = vec![0.0; n];
       let mut blue = vec![0.0; n];
       
       // Deinterleave RGB channels
       for i in 0..n {
           red[i] = rgb_sequence[i * 3];
           green[i] = rgb_sequence[i * 3 + 1];
           blue[i] = rgb_sequence[i * 3 + 2];
       }
       
       // Normalize to zero mean
       let mean_r: f32 = red.iter().sum::<f32>() / n as f32;
       let mean_g: f32 = green.iter().sum::<f32>() / n as f32;
       let mean_b: f32 = blue.iter().sum::<f32>() / n as f32;
       
       for i in 0..n {
           red[i] -= mean_r;
           green[i] -= mean_g;
           blue[i] -= mean_b;
       }
       
       // POS projection
       let mut s1 = vec![0.0; n];
       let mut s2 = vec![0.0; n];
       
       for i in 0..n {
           s1[i] = red[i] - green[i];
           s2[i] = red[i] + green[i] - 2.0 * blue[i];
       }
       
       // Compute pulse signal
       let std_s1 = compute_std(&s1);
       let std_s2 = compute_std(&s2);
       
       let alpha = std_s1 / std_s2;
       let mut pulse: Vec<f32> = s1.iter().zip(s2.iter())
           .map(|(x, y)| x - alpha * y)
           .collect();
       
       // Apply bandpass filter and estimate BPM
       let mut filter = ButterworthFilter::new(30.0, 0.75, 3.0);
       filter.process(&mut pulse);
       
       estimate_bpm(&pulse, 30.0)
   }
   ```

**Deliverables:**
- ONNX Runtime integration with WebGPU support
- Automatic fallback to Wasm POS algorithm
- Performance benchmarks comparing both paths

**Success Criteria:**
- WebGPU path achieves <20ms inference time
- Fallback path achieves <50ms inference time
- BPM accuracy within ±5 BPM of ground truth on test dataset

---

#### Phase 4: The Tactical HUD (Week 4)

**Objective**: Build React components for biometric visualization.

**Tasks:**
1. **Sparkline Component (Canvas-based)**
   ```typescript
   // components/Sparkline.tsx
   import { useEffect, useRef } from 'react';
   
   interface SparklineProps {
     data: Float32Array;
     width?: number;
     height?: number;
     color?: string;
   }
   
   export function Sparkline({ data, width = 200, height = 50, color = '#00ff00' }: SparklineProps) {
     const canvasRef = useRef<HTMLCanvasElement>(null);
     
     useEffect(() => {
       const canvas = canvasRef.current;
       if (!canvas) return;
       
       const ctx = canvas.getContext('2d')!;
       ctx.clearRect(0, 0, width, height);
       
       // Find min/max for scaling
       const min = Math.min(...data);
       const max = Math.max(...data);
       const range = max - min;
       
       // Draw line
       ctx.strokeStyle = color;
       ctx.lineWidth = 2;
       ctx.beginPath();
       
       for (let i = 0; i < data.length; i++) {
         const x = (i / data.length) * width;
         const y = height - ((data[i] - min) / range) * height;
         
         if (i === 0) ctx.moveTo(x, y);
         else ctx.lineTo(x, y);
       }
       
       ctx.stroke();
     }, [data, width, height, color]);
     
     return <canvas ref={canvasRef} width={width} height={height} />;
   }
   ```

2. **Bio-HUD Component**
   ```typescript
   // components/BioHUD.tsx
   import { useBioSensor } from '../hooks/useBioSensor';
   import { Sparkline } from './Sparkline';
   import { SignalStrength } from './SignalStrength';
   import { StatusIndicator } from './StatusIndicator';
   
   interface BioHUDProps {
     playerId: string;
     playerName: string;
   }
   
   export function BioHUD({ playerId, playerName }: BioHUDProps) {
     const { bpm, history, signal_quality, status } = useBioSensor();
     
     return (
       <div className="bio-hud">
         <div className="header">
           <span className="player-id">{playerId}</span>
           <span className="player-name">{playerName}</span>
         </div>
         
         <div className="metrics">
           <div className="metric-large">
             <div className="label">HEART RATE</div>
             <div className="value">
               {status === 'MEASURING' ? bpm.toFixed(0) : '--'}
               <span className="unit">BPM</span>
             </div>
           </div>
           
           <div className="metric-graph">
             <div className="label">10s HISTORY</div>
             <Sparkline 
               data={history} 
               width={300} 
               height={80}
               color={getBPMColor(bpm)}
             />
           </div>
           
           <div className="metric-small">
             <div className="label">SIGNAL QUALITY</div>
             <SignalStrength quality={signal_quality} />
           </div>
         </div>
         
         <StatusIndicator status={status} />
       </div>
     );
   }
   
   function getBPMColor(bpm: number): string {
     if (bpm < 60) return '#4a9eff';  // Low (blue)
     if (bpm < 100) return '#00ff00'; // Normal (green)
     if (bpm < 130) return '#ffaa00'; // Elevated (orange)
     return '#ff0000';                 // High (red)
   }
   ```

3. **Signal Strength Gauge**
   ```typescript
   // components/SignalStrength.tsx
   export function SignalStrength({ quality }: { quality: number }) {
     const bars = Math.ceil(quality * 5);
     
     return (
       <div className="signal-bars">
         {[1, 2, 3, 4, 5].map(i => (
           <div 
             key={i}
             className={`bar ${i <= bars ? 'active' : 'inactive'}`}
             style={{ height: `${i * 20}%` }}
           />
         ))}
       </div>
     );
   }
   ```

4. **Ambient Atmosphere Controller**
   ```typescript
   // hooks/useAmbientAtmosphere.ts
   import { useEffect } from 'react';
   
   export function useAmbientAtmosphere(playerBPMs: number[]) {
     useEffect(() => {
       // Calculate collective stress index
       const avgBPM = playerBPMs.reduce((a, b) => a + b, 0) / playerBPMs.length;
       const varianceBPM = playerBPMs.reduce((sum, bpm) => 
         sum + Math.pow(bpm - avgBPM, 2), 0
       ) / playerBPMs.length;
       
       const stressIndex = Math.min((avgBPM - 60) / 40, 1.0);  // Normalize to 0-1
       
       // Adjust ambient lighting (CSS variable)
       document.documentElement.style.setProperty(
         '--ambient-hue',
         `${120 - stressIndex * 60}deg`  // Green (120) to Red (60)
       );
       
       document.documentElement.style.setProperty(
         '--ambient-saturation',
         `${30 + stressIndex * 50}%`  // Subtle to intense
       );
       
       // Adjust background music (Web Audio API)
       if (window.audioContext) {
         const filter = window.audioContext.createBiquadFilter();
         filter.type = 'lowpass';
         filter.frequency.value = 2000 - stressIndex * 1000;  // Darker when stressed
       }
     }, [playerBPMs]);
   }
   ```

5. **Styling (Cyberpunk Aesthetic)**
   ```css
   /* styles/bio-hud.css */
   .bio-hud {
     background: linear-gradient(135deg, #0a0a0a 0%, #1a1a2e 100%);
     border: 2px solid var(--ambient-hue, #00ff00);
     box-shadow: 0 0 20px rgba(0, 255, 0, 0.3);
     padding: 20px;
     font-family: 'Courier New', monospace;
     color: #00ff00;
   }
   
   .metric-large .value {
     font-size: 48px;
     font-weight: bold;
     text-shadow: 0 0 10px currentColor;
   }
   
   .signal-bars {
     display: flex;
     gap: 4px;
     align-items: flex-end;
     height: 30px;
   }
   
   .bar.active {
     background: #00ff00;
     box-shadow: 0 0 5px #00ff00;
   }
   
   .bar.inactive {
     background: #333;
   }
   ```

**Deliverables:**
- Complete Bio-HUD component suite
- Canvas-based sparkline renderer (60fps)
- Ambient atmosphere system
- Cyberpunk visual styling

**Success Criteria:**
- UI maintains 60fps with 4 concurrent Bio-HUD components
- Sparkline renders 300 data points without jank
- Color transitions smooth and responsive to BPM changes

---

## PART 5: The "Go" Signal

### Validation Checklist

Before proceeding to implementation, this specification must be reviewed and approved. The following items must be confirmed:

- [ ] **Product Vision**: Does the PulseCheck concept align with the intended user experience?
- [ ] **Privacy Architecture**: Is the "black box" approach (local-only video processing) acceptable?
- [ ] **Technical Stack**: Are all specified libraries and tools approved for use?
- [ ] **Performance Targets**: Are the stated benchmarks (60fps UI, <20ms inference) realistic?
- [ ] **Implementation Timeline**: Is the 4-week phased plan feasible?

### Next Steps

⚠️ **HOLD**: Do not generate application code until this specification receives explicit approval.

**Upon Approval:**
1. Create GitHub project board with tasks from Phase 1-4
2. Set up initial repository structure:
   ```
   pulsecheck/
   ├── ppg-wasm/          # Rust signal processing
   ├── src/
   │   ├── workers/       # Web Workers
   │   ├── hooks/         # React hooks
   │   ├── components/    # UI components
   │   └── utils/         # ONNX engine, helpers
   ├── test_data/         # Synthetic test sequences
   ├── models/            # ONNX model files
   └── docs/              # Additional documentation
   ```
3. Begin Phase 1: Rust Core implementation

### Contact Points

For questions or clarification on this specification, please raise issues using the following tags:
- `spec:vision` - Product/UX questions
- `spec:architecture` - Technical design questions
- `spec:performance` - Performance target concerns
- `spec:timeline` - Implementation schedule questions

---

**END OF SPECIFICATION**

This document is version-controlled. All changes require approval and must update the version number.
