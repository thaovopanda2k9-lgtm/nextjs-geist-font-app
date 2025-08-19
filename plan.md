```markdown
# Detailed Implementation Plan for Voice Deepfake Detection PWA

This plan outlines the modifications and additions needed to build a Progressive Web App (PWA) within the existing Next.js project. The app will allow users to record their voice, analyze it to detect whether it is real or an AI deepfake, display detailed metrics (authentication rate, naturalness, stability), and provide final verdicts. A separate download page will simulate providing APK and IPA files.

---

## 1. PWA Integration

### File: next.config.ts
- **Changes:**
  - Install the next-pwa package:  
    Run: `npm install next-pwa`
  - Import and configure next-pwa:
    ```typescript
    const withPWA = require('next-pwa')({
      dest: 'public',
      disable: process.env.NODE_ENV === 'development'
    });
    module.exports = withPWA({
      reactStrictMode: true,
      // other configs as needed
    });
    ```
- **Error Handling:**
  - Ensure that if the service worker registration fails, fallback to standard Next.js behavior.

### File: public/manifest.json
- **Add New File:**
  - Create `public/manifest.json` with content similar to:
    ```json
    {
      "short_name": "VoiceCheck",
      "name": "Voice Deepfake Detection App",
      "start_url": "/voice-detection",
      "display": "standalone",
      "background_color": "#ffffff",
      "theme_color": "#2f855a",
      "icons": [
        {
          "src": "/icons/icon-192x192.png",
          "sizes": "192x192",
          "type": "image/png"
        },
        {
          "src": "/icons/icon-512x512.png",
          "sizes": "512x512",
          "type": "image/png"
        }
      ]
    }
    ```
  - **Note:** Ensure these icons exist in `public/icons/` or replace with your own assets.

---

## 2. Voice Recording and Analysis Feature

### A. New Page for Voice Detection

#### File: src/app/voice-detection/page.tsx
- **Purpose:** Main page for recording voice and displaying results.
- **Changes:**
  - Create a new page with a modern layout (centered header, clear spacing).
  - Import and use the `VoiceRecorder` and `AnalysisResult` components.
  - Maintain a state to store the audio blob and analysis results.
  - Example structure:
    ```tsx
    "use client";
    import { useState } from 'react';
    import VoiceRecorder from '@/components/VoiceRecorder';
    import AnalysisResult, { AnalysisResultData } from '@/components/AnalysisResult';
    import '@/app/globals.css';

    export default function VoiceDetectionPage() {
      const [audioBlob, setAudioBlob] = useState<Blob | null>(null);
      const [analysis, setAnalysis] = useState<AnalysisResultData | null>(null);
      const [error, setError] = useState<string>('');

      const handleRecordingComplete = async (blob: Blob) => {
        setAudioBlob(blob);
        try {
          // Call analysis function (simulate or integrate TensorFlow.js)
          const { analyzeVoice } = await import('@/lib/voiceAnalysis');
          const result = await analyzeVoice(blob);
          setAnalysis(result);
        } catch (err) {
          setError("Phân tích giọng nói gặp lỗi.");
        }
      };

      const handleReset = () => {
        setAudioBlob(null);
        setAnalysis(null);
        setError('');
      };

      return (
        <div style={{ maxWidth: '600px', margin: '0 auto', padding: '2rem' }}>
          <h1 style={{ textAlign: 'center', fontSize: '2rem', marginBottom: '1rem' }}>
            Phát Hiện Giọng Nói Deepfake
          </h1>
          {!audioBlob && (
            <VoiceRecorder onRecordingComplete={handleRecordingComplete} onError={setError} />
          )}
          {audioBlob && !analysis && !error && (
            <p style={{ textAlign: 'center' }}>Đang phân tích, vui lòng chờ...</p>
          )}
          {analysis && (
            <AnalysisResult data={analysis} onReset={handleReset} />
          )}
          {error && (
            <div style={{ color: 'red', textAlign: 'center', marginTop: '1rem' }}>
              {error}
              <button onClick={handleReset} style={{ marginLeft: '1rem', padding: '0.5rem 1rem' }}>
                Thử lại
              </button>
            </div>
          )}
        </div>
      );
    }
    ```
- **UI/UX Considerations:**
  - Use simple typography and ample spacing.
  - Provide clear call-to-action buttons with descriptive labels.

---

### B. Voice Recorder Component

#### File: src/components/VoiceRecorder.tsx
- **Purpose:** Handle voice recording using the Web Audio API.
- **Changes:**
  - Create the component with buttons: “Record”, “Stop”, and display recording status.
  - Use the MediaRecorder API to capture audio.
  - Include error handling if microphone access is denied.
  - Example implementation:
    ```tsx
    "use client";
    import { useState, useRef } from 'react';

    interface VoiceRecorderProps {
      onRecordingComplete: (blob: Blob) => void;
      onError: (errorMsg: string) => void;
    }

    const VoiceRecorder: React.FC<VoiceRecorderProps> = ({ onRecordingComplete, onError }) => {
      const [recording, setRecording] = useState<boolean>(false);
      const [errorMessage, setErrorMessage] = useState<string>('');
      const mediaRecorderRef = useRef<MediaRecorder | null>(null);
      const audioChunksRef = useRef<Blob[]>([]);

      const startRecording = async () => {
        try {
          const stream = await navigator.mediaDevices.getUserMedia({ audio: true });
          const mediaRecorder = new MediaRecorder(stream);
          mediaRecorderRef.current = mediaRecorder;
          audioChunksRef.current = [];
          mediaRecorder.ondataavailable = (event) => {
            if (event.data.size > 0) {
              audioChunksRef.current.push(event.data);
            }
          };
          mediaRecorder.onstop = () => {
            const audioBlob = new Blob(audioChunksRef.current, { type: 'audio/wav' });
            onRecordingComplete(audioBlob);
          };
          mediaRecorder.start();
          setRecording(true);
        } catch (error) {
          const msg = "Không thể truy cập microphone.";
          setErrorMessage(msg);
          onError(msg);
        }
      };

      const stopRecording = () => {
        mediaRecorderRef.current?.stop();
        setRecording(false);
      };

      return (
        <div style={{ textAlign: 'center' }}>
          {!recording ? (
            <button
              onClick={startRecording}
              style={{ padding: '1rem 2rem', fontSize: '1rem', margin: '1rem' }}
            >
              Bắt đầu ghi âm
            </button>
          ) : (
            <button
              onClick={stopRecording}
              style={{ padding: '1rem 2rem', fontSize: '1rem', margin: '1rem', backgroundColor: '#e53e3e', color: '#fff' }}
            >
              Dừng ghi âm
            </button>
          )}
          {errorMessage && <p style={{ color: 'red' }}>{errorMessage}</p>}
        </div>
      );
    };

    export default VoiceRecorder;
    ```
- **Error Handling:**
  - Catch exceptions from getUserMedia and update error messages.
  - Display error texts to the user and offer a retry.

---

### C. Analysis Result Component

#### File: src/components/AnalysisResult.tsx
- **Purpose:** Display analysis metrics (authentication rate, naturalness, stability) and the final verdict.
- **Changes:**
  - Accept props for analysis data and a reset callback.
  - Render each metric in a clean card layout.
  - Example implementation:
    ```tsx
    import React from 'react';

    export interface AnalysisResultData {
      authenticationRate: number;
      naturalness: number;
      stability: number;
      verdict: string; // "Thật" or "Giả"
    }

    interface AnalysisResultProps {
      data: AnalysisResultData;
      onReset: () => void;
    }

    const AnalysisResult: React.FC<AnalysisResultProps> = ({ data, onReset }) => {
      return (
        <div style={{ border: '1px solid #e2e8f0', borderRadius: '8px', padding: '2rem', marginTop: '2rem' }}>
          <h2 style={{ textAlign: 'center', marginBottom: '1rem' }}>Kết quả phân tích</h2>
          <div style={{ marginBottom: '1rem' }}>
            <strong>Tỷ lệ xác thực:</strong> {data.authenticationRate}%
          </div>
          <div style={{ marginBottom: '1rem' }}>
            <strong>Mức độ tự nhiên:</strong> {data.naturalness}%
          </div>
          <div style={{ marginBottom: '1rem' }}>
            <strong>Độ ổn định:</strong> {data.stability}%
          </div>
          <div style={{ marginTop: '1.5rem', textAlign: 'center', fontSize: '1.25rem', fontWeight: 'bold' }}>
            Phán đoán cuối cùng: {data.verdict}
          </div>
          <div style={{ textAlign: 'center', marginTop: '2rem' }}>
            <button onClick={onReset} style={{ padding: '0.75rem 1.5rem', fontSize: '1rem' }}>
              Ghi âm lại
            </button>
          </div>
        </div>
      );
    };

    export default AnalysisResult;
    ```
- **UI/UX Considerations:**
  - Use clear typography and a visually distinct card to highlight results.
  - Ensure responsive design for mobile devices.

---

### D. Voice Analysis Logic (Simulation)

#### File: src/lib/voiceAnalysis.ts
- **Purpose:** Simulate the deepfake detection analysis.
- **Changes:**
  - Create an async function `analyzeVoice` that accepts an audio Blob.
  - Simulate processing with a delay (e.g., using setTimeout or Promise delay).
  - Randomly generate metrics between 0 and 100. Define simple thresholds (e.g., if all metrics exceed 70, verdict is “Thật”; otherwise “Giả”).
  - Example implementation:
    ```typescript
    export interface AnalysisResult {
      authenticationRate: number;
      naturalness: number;
      stability: number;
      verdict: string;
    }

    export async function analyzeVoice(audioBlob: Blob): Promise<AnalysisResult> {
      // Simulate processing delay
      await new Promise((resolve) => setTimeout(resolve, 2000));

      // Generate simulated metrics
      const authenticationRate = Math.floor(Math.random() * 41) + 60; // 60-100%
      const naturalness = Math.floor(Math.random() * 41) + 60;
      const stability = Math.floor(Math.random() * 41) + 60;

      const verdict =
        authenticationRate >= 75 && naturalness >= 75 && stability >= 75 ? "Thật" : "Giả";

      return { authenticationRate, naturalness, stability, verdict };
    }
    ```
- **Error Handling:**
  - Include try/catch if additional processing is integrated later.
  - Validate that a non-empty Blob is provided.

---

## 3. Download Links for APK & IPA

### File: src/app/download/page.tsx
- **Purpose:** Provide a dedicated page with simulated download links for Android (APK) and iOS (IPA) packages.
- **Changes:**
  - Create a page that presents two prominent buttons/links.
  - Links can point to static files stored in `public/` (e.g., `public/downloads/app.apk` and `public/downloads/app.ipa`) or simulated placeholders.
  - Example implementation:
    ```tsx
    export default function DownloadPage() {
      return (
        <div style={{ maxWidth: '600px', margin: '0 auto', padding: '2rem', textAlign: 'center' }}>
          <h1 style={{ fontSize: '2rem', marginBottom: '1rem' }}>Tải xuống Ứng dụng</h1>
          <div style={{ margin: '1rem' }}>
            <a 
              href="/downloads/app.apk" 
              style={{ padding: '1rem 2rem', backgroundColor: '#3182ce', color: '#fff', textDecoration: 'none', borderRadius: '4px' }}
            >
              Tải xuống APK cho Android
            </a>
          </div>
          <div style={{ margin: '1rem' }}>
            <a 
              href="/downloads/app.ipa" 
              style={{ padding: '1rem 2rem', backgroundColor: '#38a169', color: '#fff', textDecoration: 'none', borderRadius: '4px' }}
            >
              Tải xuống IPA cho iOS
            </a>
          </div>
          <p style={{ marginTop: '2rem', fontSize: '0.875rem', color: '#4a5568' }}>
            (Lưu ý: Các liên kết tải xuống là mô phỏng và cần được đóng gói thực tế khi triển khai ứng dụng di động.)
          </p>
        </div>
      );
    }
    ```
- **UI/UX Considerations:**
  - Ensure buttons are large, easy-to-tap, and styled with consistent color themes.
  - Provide clear instructions and note that these links are for demonstration.

---

## 4. Overall Best Practices and Testing

- **Error Handling:**
  - Implement try/catch in asynchronous operations.
  - Provide user feedback on any errors (e.g., microphone access, processing failures).
- **Responsive Design:**
  - Use inline styles or CSS classes (in `globals.css`) for responsiveness and consistent spacing.
  - Ensure the UI adapts well to various mobile screen sizes.
- **Component Reusability:**
  - Keep components modular (e.g., separation of recorder, result display, analysis logic).
- **Simulated Analysis:**
  - Since no external API keys are used, the analysis logic uses simulated delays and random metric generation.
- **Testing:**
  - Test recording functionality using browser dev tools.
  - Validate the PWA installation process in mobile browsers.
  - Use curl commands to check any static file endpoints if necessary (for APK and IPA downloads).

---

## Summary

- Integrated next-pwa in next.config.ts and added a manifest file for PWA support.
- Created a new voice detection page that combines a recorder (using the Web Audio API) and an analysis component.
- Implemented a simulated voice analysis function generating metrics and a final verdict for deepfake detection.
- Developed a modern, responsive UI with clear typography and spacing, avoiding external icon libraries.
- Added a download page with simulated links for APK and IPA files.
- Emphasized proper error handling and best practices for a production-level PWA mobile experience.
