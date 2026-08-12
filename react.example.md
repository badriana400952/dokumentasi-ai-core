# ⚛️ Panduan Integrasi Frontend React: Custom Hooks & Komponen UI Multimodal (Powered by `@badriana/ai-core`)

Dokumen ini berisi panduan lengkap **Frontend React (React 18/19, Vite, Next.js Client Component, & Tailwind CSS)** yang terintegrasi 100% presisi (1-to-1 matching) dengan 7 API Endpoint Backend yang telah dibangun (Express, NestJS, Next.js, Fastify, Koa) untuk **Upload File PDF, Ingesti Database 40+ Tabel, Input Teks Knowledge Manual, Chat AI RAG Vision, Streaming Chat SSE (Ketik Instan), Text-to-Speech (TTS), & Speech-to-Text (STT)**.

---

## 🏗️ Struktur Folder Frontend React

```text
my-react-ai-app/
├── src/
│   ├── hooks/
│   │   └── useAICore.ts                       # Custom Hook Reusable (7 API Endpoints)
│   ├── components/
│   │   ├── DocumentIngestComponent.tsx        # UI Upload File PDF + Progress Bar (0% - 100%)
│   │   ├── ManualKnowledgeIngestComponent.tsx # UI Input Teks Knowledge Manual / SOP / Artikel
│   │   ├── DatabaseIngestComponent.tsx        # UI Ingesti Data Database Perusahaan (40+ Tabel SQL/JSON)
│   │   ├── MultimodalChatComponent.tsx        # UI Chat AI RAG + Upload Gambar + Citations
│   │   ├── StreamingChatComponent.tsx         # UI Streaming Chat SSE (Efek Ketik Instan)
│   │   ├── AudioTTSComponent.tsx              # UI Text-to-Speech (Pemutar Audio MP3 AI)
│   │   └── AudioSTTComponent.tsx              # UI Speech-to-Text (Perekam Suara Mikrofon)
│   ├── App.tsx                                # Dashboard Utama (7 Tab Navigasi Presisi)
│   └── main.tsx
├── package.json
└── tsconfig.json
```

---

## 1. 🎣 Custom React Hook Presisi 7 Endpoint (`src/hooks/useAICore.ts`)

Custom Hook TypeScript reusable yang membungkus 7 panggilan API `fetch` backend, mengelola status `isLoading`, `error`, pemutaran audio MP3, dan pembacaan stream Server-Sent Events (SSE).

```typescript
// src/hooks/useAICore.ts
import { useState, useCallback } from "react";

const BACKEND_BASE_URL = process.env.VITE_API_BASE_URL || "http://localhost:3000/api/v1/ai";

export function useAICore() {
  const [isLoading, setIsLoading] = useState(false);
  const [error, setError] = useState<string | null>(null);

  /**
   * 1. Ingesti Berkas PDF/Teks (POST /api/v1/ai/ingest-file)
   */
  const ingestFile = useCallback(async (file: File) => {
    setIsLoading(true);
    setError(null);
    try {
      const formData = new FormData();
      formData.append("file", file);

      const res = await fetch(`${BACKEND_BASE_URL}/ingest-file`, {
        method: "POST",
        body: formData,
      });

      const json = await res.json();
      if (!res.ok || !json.success) {
        throw new Error(json.error || "Gagal meng-ingest berkas PDF");
      }
      return json.data;
    } catch (err: any) {
      setError(err.message);
      throw err;
    } finally {
      setIsLoading(false);
    }
  }, []);

  /**
   * 2. Ingesti Teks Knowledge Manual / SOP (POST /api/v1/ai/ingest-manual)
   */
  const ingestManualText = useCallback(async (title: string, content: string) => {
    setIsLoading(true);
    setError(null);
    try {
      const res = await fetch(`${BACKEND_BASE_URL}/ingest-manual`, {
        method: "POST",
        headers: { "Content-Type": "application/json" },
        body: JSON.stringify({ title, content }),
      });

      const json = await res.json();
      if (!res.ok || !json.success) {
        throw new Error(json.error || "Gagal meng-ingest teks manual");
      }
      return json.data;
    } catch (err: any) {
      setError(err.message);
      throw err;
    } finally {
      setIsLoading(false);
    }
  }, []);

  /**
   * 3. Ingesti Records Tabel Database (POST /api/v1/ai/ingest-db)
   */
  const ingestDatabase = useCallback(async (tableName: string, records: Record<string, any>[]) => {
    setIsLoading(true);
    setError(null);
    try {
      const res = await fetch(`${BACKEND_BASE_URL}/ingest-db`, {
        method: "POST",
        headers: { "Content-Type": "application/json" },
        body: JSON.stringify({ tableName, records }),
      });

      const json = await res.json();
      if (!res.ok || !json.success) {
        throw new Error(json.error || "Gagal meng-ingest database");
      }
      return json.data;
    } catch (err: any) {
      setError(err.message);
      throw err;
    } finally {
      setIsLoading(false);
    }
  }, []);

  /**
   * 4. Tanya Jawab Multimodal Vision Chat (POST /api/v1/ai/chat)
   */
  const sendChat = useCallback(async (question: string, imageBase64?: string) => {
    setIsLoading(true);
    setError(null);
    try {
      const res = await fetch(`${BACKEND_BASE_URL}/chat`, {
        method: "POST",
        headers: { "Content-Type": "application/json" },
        body: JSON.stringify({ question, imageBase64 }),
      });

      const json = await res.json();
      if (!res.ok || !json.success) {
        throw new Error(json.error || "Gagal memproses chat AI");
      }
      return json.data;
    } catch (err: any) {
      setError(err.message);
      throw err;
    } finally {
      setIsLoading(false);
    }
  }, []);

  /**
   * 5. Streaming Chat SSE / Ketik Instan (POST /api/v1/ai/chat-stream)
   */
  const sendChatStream = useCallback(async (question: string, onChunk: (text: string) => void) => {
    setIsLoading(true);
    setError(null);
    try {
      const res = await fetch(`${BACKEND_BASE_URL}/chat-stream`, {
        method: "POST",
        headers: { "Content-Type": "application/json" },
        body: JSON.stringify({ question }),
      });

      if (!res.ok || !res.body) {
        throw new Error("Gagal menginisialisasi stream chat AI");
      }

      const reader = res.body.getReader();
      const decoder = new TextDecoder();
      let done = false;

      while (!done) {
        const { value, done: doneReading } = await reader.read();
        done = doneReading;
        if (value) {
          const chunkStr = decoder.decode(value);
          const lines = chunkStr.split("\n\n");
          for (const line of lines) {
            if (line.startsWith("data: ")) {
              const dataStr = line.replace("data: ", "").trim();
              if (dataStr === "[DONE]") break;
              try {
                const parsed = JSON.parse(dataStr);
                if (parsed.text) onChunk(parsed.text);
              } catch (e) {
                // Abaikan potongan JSON parsial
              }
            }
          }
        }
      }
    } catch (err: any) {
      setError(err.message);
      throw err;
    } finally {
      setIsLoading(false);
    }
  }, []);

  /**
   * 6. Text-to-Speech MP3 Stream (POST /api/v1/ai/audio/speak)
   */
  const speakText = useCallback(async (text: string, voice: string = "nova") => {
    setIsLoading(true);
    setError(null);
    try {
      const res = await fetch(`${BACKEND_BASE_URL}/audio/speak`, {
        method: "POST",
        headers: { "Content-Type": "application/json" },
        body: JSON.stringify({ text, voice }),
      });

      if (!res.ok) throw new Error("Gagal mensintesis suara AI");

      const blob = await res.blob();
      const audioUrl = URL.createObjectURL(blob);
      const audio = new Audio(audioUrl);
      await audio.play();

      return audioUrl;
    } catch (err: any) {
      setError(err.message);
      throw err;
    } finally {
      setIsLoading(false);
    }
  }, []);

  /**
   * 7. Speech-to-Text Transkripsi Suara (POST /api/v1/ai/audio/transcribe)
   */
  const transcribeAudio = useCallback(async (audioBase64: string, language: string = "id") => {
    setIsLoading(true);
    setError(null);
    try {
      const res = await fetch(`${BACKEND_BASE_URL}/audio/transcribe`, {
        method: "POST",
        headers: { "Content-Type": "application/json" },
        body: JSON.stringify({ audioBase64, language }),
      });

      const json = await res.json();
      if (!res.ok || !json.success) throw new Error(json.error || "Gagal mentranskripsi suara");
      return json.data;
    } catch (err: any) {
      setError(err.message);
      throw err;
    } finally {
      setIsLoading(false);
    }
  }, []);

  return {
    isLoading,
    error,
    ingestFile,
    ingestManualText,
    ingestDatabase,
    sendChat,
    sendChatStream,
    speakText,
    transcribeAudio,
  };
}
```

---

## 2. 📄 Komponen Upload Dokumen + Progress Bar (`src/components/DocumentIngestComponent.tsx`)

```tsx
// src/components/DocumentIngestComponent.tsx
import React, { useState } from "react";
import { useAICore } from "../hooks/useAICore";

export function DocumentIngestComponent() {
  const { ingestFile, isLoading, error } = useAICore();
  const [selectedFile, setSelectedFile] = useState<File | null>(null);
  const [progress, setProgress] = useState(0);
  const [stageMessage, setStageMessage] = useState("");
  const [resultStats, setResultStats] = useState<any>(null);

  const handleFileChange = (e: React.ChangeEvent<HTMLInputElement>) => {
    if (e.target.files && e.target.files[0]) {
      setSelectedFile(e.target.files[0]);
      setResultStats(null);
    }
  };

  const handleUpload = async (e: React.FormEvent) => {
    e.preventDefault();
    if (!selectedFile) return;

    setProgress(15);
    setStageMessage("Memparsing struktur dokumen PDF...");

    try {
      setTimeout(() => { setProgress(45); setStageMessage("Mengekstrak gambar & Vision OCR..."); }, 400);
      setTimeout(() => { setProgress(75); setStageMessage("Menghasilkan Vektor Embedding..."); }, 900);

      const data = await ingestFile(selectedFile);

      setProgress(100);
      setStageMessage("Dokumen sukses di-ingest ke Vector Database!");
      setResultStats(data.stats);
    } catch (err) {
      setProgress(0);
      setStageMessage("Gagal meng-ingest dokumen!");
    }
  };

  return (
    <div className="bg-slate-900 border border-slate-800 rounded-xl p-6 shadow-xl">
      <h2 className="text-lg font-bold text-blue-400 mb-4">📄 Endpoint 1: POST /api/v1/ai/ingest-file</h2>

      <form onSubmit={handleUpload} className="flex flex-col gap-4">
        <input
          type="file"
          accept=".pdf,.txt,.md"
          onChange={handleFileChange}
          className="block w-full text-xs text-slate-400 file:mr-4 file:py-2.5 file:px-4 file:rounded-lg file:border-0 file:text-xs file:font-semibold file:bg-blue-600 file:text-white hover:file:bg-blue-500 cursor-pointer"
        />

        {isLoading || progress > 0 ? (
          <div className="flex flex-col gap-1.5 mt-2">
            <div className="flex justify-between text-xs font-medium text-slate-300">
              <span>{stageMessage}</span>
              <span className="text-blue-400 font-bold">{progress}%</span>
            </div>
            <div className="w-full bg-slate-800 rounded-full h-2.5 overflow-hidden">
              <div
                className="bg-gradient-to-r from-blue-600 to-cyan-400 h-2.5 rounded-full transition-all duration-300 ease-out"
                style={{ width: `${progress}%` }}
              />
            </div>
          </div>
        ) : null}

        <button
          type="submit"
          disabled={!selectedFile || isLoading}
          className="w-full bg-blue-600 hover:bg-blue-500 disabled:bg-slate-800 text-white font-medium py-2.5 rounded-lg text-xs transition"
        >
          {isLoading ? "Memproses Ingesti..." : "Upload & Mulai Ingesti RAG"}
        </button>
      </form>

      {error && <p className="text-xs text-rose-400 mt-3">⚠️ {error}</p>}

      {resultStats && (
        <div className="mt-6 bg-slate-950 border border-slate-800 rounded-lg p-4 text-xs font-mono text-slate-300">
          <p className="font-bold text-emerald-400 mb-2">✅ Statistik Ingesti Berhasil:</p>
          <ul className="space-y-1 text-slate-400">
            <li>• Total Karakter: {resultStats.characters}</li>
            <li>• Total Kata: {resultStats.words}</li>
            <li>• Jumlah Chunks Vektor: {resultStats.chunks}</li>
            <li>• Total Waktu: {resultStats.totalTimeMs} ms</li>
          </ul>
        </div>
      )}
    </div>
  );
}
```

---

## 3. ✍️ Komponen Input Knowledge Teks Manual (`src/components/ManualKnowledgeIngestComponent.tsx`)

```tsx
// src/components/ManualKnowledgeIngestComponent.tsx
import React, { useState } from "react";
import { useAICore } from "../hooks/useAICore";

export function ManualKnowledgeIngestComponent() {
  const { ingestManualText, isLoading, error } = useAICore();
  const [title, setTitle] = useState("SOP_Tata_Tertib_Sekolah");
  const [content, setContent] = useState("Siswa wajib hadir pukul 07.00 WIB. Pakaian seragam lengkap sesuai hari.");
  const [isSuccess, setIsSuccess] = useState(false);

  const handleSubmit = async (e: React.FormEvent) => {
    e.preventDefault();
    if (!title || !content) return;
    setIsSuccess(false);

    try {
      await ingestManualText(title, content);
      setIsSuccess(true);
    } catch (err) {
      console.error(err);
    }
  };

  return (
    <div className="bg-slate-900 border border-slate-800 rounded-xl p-6 shadow-xl">
      <h2 className="text-lg font-bold text-blue-400 mb-4">✍️ Endpoint 2: POST /api/v1/ai/ingest-manual</h2>

      <form onSubmit={handleSubmit} className="flex flex-col gap-4">
        <div>
          <label className="text-xs font-semibold text-slate-400">Judul Pengetahuan / SOP:</label>
          <input
            type="text"
            value={title}
            onChange={(e) => setTitle(e.target.value)}
            className="w-full bg-slate-800 border border-slate-700 rounded-lg p-2.5 text-xs text-white mt-1 focus:outline-none focus:border-blue-500"
          />
        </div>

        <div>
          <label className="text-xs font-semibold text-slate-400">Isi Teks Pengetahuan:</label>
          <textarea
            value={content}
            onChange={(e) => setContent(e.target.value)}
            className="w-full bg-slate-800 border border-slate-700 rounded-lg p-3 text-xs text-white mt-1 focus:outline-none focus:border-blue-500"
            rows={5}
          />
        </div>

        <button
          type="submit"
          disabled={isLoading}
          className="w-full bg-blue-600 hover:bg-blue-500 text-white font-medium py-2.5 rounded-lg text-xs transition"
        >
          {isLoading ? "Menyimpan Knowledge..." : "Simpan Pengetahuan Manual"}
        </button>
      </form>

      {error && <p className="text-xs text-rose-400 mt-3">⚠️ {error}</p>}
      {isSuccess && <p className="text-xs text-emerald-400 mt-3">✅ Teks Knowledge berhasil disimpan!</p>}
    </div>
  );
}
```

---

## 4. 🗄️ Komponen Ingesti Database Perusahaan (`src/components/DatabaseIngestComponent.tsx`)

```tsx
// src/components/DatabaseIngestComponent.tsx
import React, { useState } from "react";
import { useAICore } from "../hooks/useAICore";

export function DatabaseIngestComponent() {
  const { ingestDatabase, isLoading, error } = useAICore();
  const [tableName, setTableName] = useState("student_scores");
  const [jsonInput, setJsonInput] = useState(`[
  { "id": 1, "student_name": "Budi", "subject": "Fisika", "score": 95, "password_hash": "123secret" },
  { "id": 2, "student_name": "Siti", "subject": "Fisika", "score": 88, "password_hash": "456secret" }
]`);
  const [resultData, setResultData] = useState<any>(null);

  const handleIngestDb = async (e: React.FormEvent) => {
    e.preventDefault();
    try {
      const records = JSON.parse(jsonInput);
      const data = await ingestDatabase(tableName, records);
      setResultData(data);
    } catch (err: any) {
      alert("Format JSON tidak valid: " + err.message);
    }
  };

  return (
    <div className="bg-slate-900 border border-slate-800 rounded-xl p-6 shadow-xl">
      <h2 className="text-lg font-bold text-blue-400 mb-4">🗄️ Endpoint 3: POST /api/v1/ai/ingest-db</h2>

      <form onSubmit={handleIngestDb} className="flex flex-col gap-4">
        <div>
          <label className="text-xs font-semibold text-slate-400">Nama Tabel Database:</label>
          <input
            type="text"
            value={tableName}
            onChange={(e) => setTableName(e.target.value)}
            className="w-full bg-slate-800 border border-slate-700 rounded-lg p-2.5 text-xs text-white mt-1 focus:outline-none focus:border-blue-500"
          />
        </div>

        <div>
          <label className="text-xs font-semibold text-slate-400">Data Baris Database (JSON Array):</label>
          <textarea
            value={jsonInput}
            onChange={(e) => setJsonInput(e.target.value)}
            className="w-full bg-slate-800 border border-slate-700 rounded-lg p-3 text-xs font-mono text-emerald-400 mt-1 focus:outline-none focus:border-blue-500"
            rows={6}
          />
        </div>

        <button
          type="submit"
          disabled={isLoading}
          className="w-full bg-blue-600 hover:bg-blue-500 text-white font-medium py-2.5 rounded-lg text-xs transition"
        >
          {isLoading ? "Memproses Ingesti Database..." : "Upload Records ke Vector DB"}
        </button>
      </form>

      {error && <p className="text-xs text-rose-400 mt-3">⚠️ {error}</p>}
      {resultData && (
        <div className="mt-4 bg-slate-950 border border-slate-800 rounded-lg p-4 text-xs font-mono text-emerald-400">
          ✅ Sukses memproses {resultData.recordsProcessed} record dari tabel '{resultData.tableName}'!
        </div>
      )}
    </div>
  );
}
```

---

## 5. ⚡ Komponen Streaming Chat SSE (`src/components/StreamingChatComponent.tsx`)

Komponen Chat RAG yang menampilkan **Efek Ketik Instan Per-Kata (Streaming SSE)**.

```tsx
// src/components/StreamingChatComponent.tsx
import React, { useState } from "react";
import { useAICore } from "../hooks/useAICore";

export function StreamingChatComponent() {
  const { sendChatStream, isLoading, error } = useAICore();
  const [question, setQuestion] = useState("");
  const [streamedAnswer, setStreamedAnswer] = useState("");

  const handleStreamSubmit = async (e: React.FormEvent) => {
    e.preventDefault();
    if (!question.trim()) return;

    setStreamedAnswer("");

    await sendChatStream(question, (chunkText) => {
      setStreamedAnswer((prev) => prev + chunkText);
    });
  };

  return (
    <div className="bg-slate-900 border border-slate-800 rounded-xl p-6">
      <h2 className="text-lg font-bold text-blue-400 mb-4">⚡ Endpoint 5: POST /api/v1/ai/chat-stream (SSE)</h2>

      <form onSubmit={handleStreamSubmit} className="flex flex-col gap-3 mb-4">
        <input
          type="text"
          value={question}
          onChange={(e) => setQuestion(e.target.value)}
          placeholder="Tanyakan materi fisika untuk melihat efek ketik instan..."
          className="w-full bg-slate-800 border border-slate-700 rounded-lg p-3 text-xs text-white focus:outline-none focus:border-blue-500"
        />

        <button
          type="submit"
          disabled={isLoading}
          className="bg-blue-600 hover:bg-blue-500 text-white font-medium py-2.5 rounded-lg text-xs transition"
        >
          {isLoading ? "Mengetik..." : "Kirim Pertanyaan Streaming"}
        </button>
      </form>

      {error && <p className="text-xs text-rose-400 mb-3">⚠️ {error}</p>}

      {streamedAnswer && (
        <div className="bg-slate-950 border border-slate-800 rounded-lg p-4 text-xs leading-relaxed text-slate-100 font-mono whitespace-pre-wrap">
          <span className="text-blue-400 font-bold block mb-1">🤖 AI Real-Time Stream Response:</span>
          {streamedAnswer}
          <span className="inline-block w-2 h-4 bg-blue-500 animate-pulse ml-1"></span>
        </div>
      )}
    </div>
  );
}
```

---

## 6. 🎙️ Komponen Speech-to-Text Perekam Suara (`src/components/AudioSTTComponent.tsx`)

```tsx
// src/components/AudioSTTComponent.tsx
import React, { useState, useRef } from "react";
import { useAICore } from "../hooks/useAICore";

export function AudioSTTComponent() {
  const { transcribeAudio, isLoading } = useAICore();
  const [isRecording, setIsRecording] = useState(false);
  const [transcript, setTranscript] = useState("");
  const mediaRecorderRef = useRef<MediaRecorder | null>(null);
  const audioChunksRef = useRef<Blob[]>([]);

  const startRecording = async () => {
    try {
      const stream = await navigator.mediaDevices.getUserMedia({ audio: true });
      const mediaRecorder = new MediaRecorder(stream);
      mediaRecorderRef.current = mediaRecorder;
      audioChunksRef.current = [];

      mediaRecorder.ondataavailable = (event) => {
        if (event.data.size > 0) audioChunksRef.current.push(event.data);
      };

      mediaRecorder.onstop = async () => {
        const audioBlob = new Blob(audioChunksRef.current, { type: "audio/mp3" });
        const reader = new FileReader();
        reader.readAsDataURL(audioBlob);
        reader.onloadend = async () => {
          const base64Audio = reader.result as string;
          const res = await transcribeAudio(base64Audio);
          setTranscript(res.text);
        };
      };

      mediaRecorder.start();
      setIsRecording(true);
    } catch (err) {
      alert("Akses mikrofon ditolak!");
    }
  };

  const stopRecording = () => {
    if (mediaRecorderRef.current) {
      mediaRecorderRef.current.stop();
      setIsRecording(false);
    }
  };

  return (
    <div className="bg-slate-900 border border-slate-800 rounded-xl p-6">
      <h2 className="text-lg font-bold text-blue-400 mb-4">🎙️ Endpoint 7: POST /api/v1/ai/audio/transcribe</h2>

      <div className="flex flex-col gap-4 items-center">
        <button
          onClick={isRecording ? stopRecording : startRecording}
          disabled={isLoading}
          className={`w-20 h-20 rounded-full font-bold text-white transition flex items-center justify-center text-2xl ${
            isRecording ? "bg-rose-600 animate-pulse" : "bg-blue-600 hover:bg-blue-500"
          }`}
        >
          {isRecording ? "⏹️" : "🎙️"}
        </button>

        <p className="text-xs text-slate-400">
          {isRecording ? "Sedang Merekam Suara..." : "Klik tombol untuk merekam suara mikrofon"}
        </p>

        {transcript && (
          <div className="w-full bg-slate-800 border border-slate-700 rounded-lg p-4 text-xs text-white">
            <p className="font-semibold text-emerald-400 mb-1">Hasil Transkripsi Suara:</p>
            <p className="italic">"{transcript}"</p>
          </div>
        )}
      </div>
    </div>
  );
}
```

---

## 7. 🎛️ Dashboard Utama (`src/App.tsx`)

Dashboard Tabbed Interface yang menggabungkan presisi 7 API Endpoint Backend.

```tsx
// src/App.tsx
import { useState } from "react";
import { DocumentIngestComponent } from "./components/DocumentIngestComponent";
import { ManualKnowledgeIngestComponent } from "./components/ManualKnowledgeIngestComponent";
import { DatabaseIngestComponent } from "./components/DatabaseIngestComponent";
import { MultimodalChatComponent } from "./components/MultimodalChatComponent";
import { StreamingChatComponent } from "./components/StreamingChatComponent";
import { AudioTTSComponent } from "./components/AudioTTSComponent";
import { AudioSTTComponent } from "./components/AudioSTTComponent";

export default function App() {
  const [activeTab, setActiveTab] = useState<"chat" | "stream" | "file" | "manual" | "db" | "tts" | "stt">("chat");

  return (
    <div className="min-h-screen bg-slate-950 text-slate-100 font-sans p-8">
      <div className="max-w-5xl mx-auto flex flex-col gap-6">
        
        <header className="flex items-center justify-between border-b border-slate-800 pb-4">
          <div>
            <h1 className="text-2xl font-extrabold text-blue-400">🚀 Frontend AI Dashboard (7 Presisi Endpoints)</h1>
            <p className="text-xs text-slate-400">100% Matching dengan Express / NestJS / Next.js / Fastify / Koa Backend</p>
          </div>

          <nav className="flex bg-slate-900 border border-slate-800 rounded-lg p-1 gap-1 flex-wrap">
            <button
              onClick={() => setActiveTab("chat")}
              className={`px-3 py-1.5 rounded-md text-xs font-medium transition ${
                activeTab === "chat" ? "bg-blue-600 text-white" : "text-slate-400 hover:text-white"
              }`}
            >
              💬 Chat RAG
            </button>
            <button
              onClick={() => setActiveTab("stream")}
              className={`px-3 py-1.5 rounded-md text-xs font-medium transition ${
                activeTab === "stream" ? "bg-blue-600 text-white" : "text-slate-400 hover:text-white"
              }`}
            >
              ⚡ Chat Stream
            </button>
            <button
              onClick={() => setActiveTab("file")}
              className={`px-3 py-1.5 rounded-md text-xs font-medium transition ${
                activeTab === "file" ? "bg-blue-600 text-white" : "text-slate-400 hover:text-white"
              }`}
            >
              📄 File PDF
            </button>
            <button
              onClick={() => setActiveTab("manual")}
              className={`px-3 py-1.5 rounded-md text-xs font-medium transition ${
                activeTab === "manual" ? "bg-blue-600 text-white" : "text-slate-400 hover:text-white"
              }`}
            >
              ✍️ Teks Manual
            </button>
            <button
              onClick={() => setActiveTab("db")}
              className={`px-3 py-1.5 rounded-md text-xs font-medium transition ${
                activeTab === "db" ? "bg-blue-600 text-white" : "text-slate-400 hover:text-white"
              }`}
            >
              🗄️ Ingest DB
            </button>
            <button
              onClick={() => setActiveTab("tts")}
              className={`px-3 py-1.5 rounded-md text-xs font-medium transition ${
                activeTab === "tts" ? "bg-blue-600 text-white" : "text-slate-400 hover:text-white"
              }`}
            >
              🔊 Audio TTS
            </button>
            <button
              onClick={() => setActiveTab("stt")}
              className={`px-3 py-1.5 rounded-md text-xs font-medium transition ${
                activeTab === "stt" ? "bg-blue-600 text-white" : "text-slate-400 hover:text-white"
              }`}
            >
              🎙️ Audio STT
            </button>
          </nav>
        </header>

        <main>
          {activeTab === "chat" && <MultimodalChatComponent />}
          {activeTab === "stream" && <StreamingChatComponent />}
          {activeTab === "file" && <DocumentIngestComponent />}
          {activeTab === "manual" && <ManualKnowledgeIngestComponent />}
          {activeTab === "db" && <DatabaseIngestComponent />}
          {activeTab === "tts" && <AudioTTSComponent />}
          {activeTab === "stt" && <AudioSTTComponent />}
        </main>
      </div>
    </div>
  );
}
```
