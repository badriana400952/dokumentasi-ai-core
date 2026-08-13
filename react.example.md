# ⚛️ Panduan Integrasi Frontend React: Custom Hooks, Service Layer & Portal Admin/User (Powered by `@badriana/ai-core`)

Dokumen ini berisi panduan lengkap **Frontend React (React 18/19, Vite, Tailwind CSS & Lucide React)** yang terintegrasi 100% presisi (1-to-1 matching) dengan backend Express.js (`tes-si-core/backend`), yang memisahkan akses menjadi **Portal Admin (Ingesti PDF, Teks Manual, & Database)** dan **Portal User (Chat RAG Multimodal, Streaming SSE, Audio TTS MP3, & STT Transkripsi)**.

---

## 🏗️ Struktur Folder Proyek Frontend React

```text
tes-si-core-frontend/
├── src/
│   ├── services/
│   │   └── api.ts                      # Client Service Central Layer (8 Endpoint API Fetch & SSE)
│   ├── components/
│   │   ├── AdminPortal.tsx             # Portal Admin: Upload PDF/TXT, Teks SOP Manual, & Ingesti DB
│   │   └── UserPortal.tsx              # Portal User: Chat RAG Multimodal, Streaming SSE, TTS, & STT
│   ├── App.tsx                         # Dashboard Utama (Header Navbar + Role Switcher Pill Admin/User)
│   ├── main.tsx                        # Entry Point Vite React
│   └── index.css                       # Tailwind CSS Styling & Glassmorphism Theme
├── package.json
├── tsconfig.json
└── vite.config.ts
```

---

## 1. 🌐 Central API Service Layer (`src/services/api.ts`)

Service layer terpusat yang membungkus seluruh panggilan API backend (`http://localhost:3000/api/v1`):

```typescript
// src/services/api.ts
const API_BASE = 'http://localhost:3000/api/v1';

export interface AdminIngestFileResult {
  documentId: string;
  filename: string;
  chunksCount: number;
  imagesCount?: number;
  stats: {
    characters: number;
    words: number;
    chunks: number;
    totalTimeMs: number;
  };
}

export interface AdminIngestManualResult {
  documentId: string;
  title: string;
  chunksCount: number;
  stats: {
    characters: number;
    words: number;
    chunks: number;
  };
}

export interface AdminIngestDbResult {
  tableName: string;
  recordsProcessed: number;
  chunksGenerated: number;
}

export interface ChatImage {
  chunkId: string;
  url: string;
  page?: number;
}

export interface UserChatResult {
  answer: string;
  citations: Array<{ chunkId: string }>;
  usage: { inputTokens: number; outputTokens: number; totalTokens: number };
  images?: ChatImage[];
}

export const api = {
  // =========================================================================
  // 🛠️ ADMIN API CALLS
  // =========================================================================
  async adminIngestFile(file: File): Promise<AdminIngestFileResult> {
    const formData = new FormData();
    formData.append('file', file);

    const res = await fetch(`${API_BASE}/admin/ingest-file`, {
      method: 'POST',
      body: formData,
    });

    const data = await res.json();
    if (!res.ok || !data.success) {
      throw new Error(data.error || 'Gagal mengunggah berkas PDF/TXT.');
    }
    return data.data;
  },

  async adminIngestManual(title: string, content: string): Promise<AdminIngestManualResult> {
    const res = await fetch(`${API_BASE}/admin/ingest-manual`, {
      method: 'POST',
      headers: { 'Content-Type': 'application/json' },
      body: JSON.stringify({ title, content }),
    });

    const data = await res.json();
    if (!res.ok || !data.success) {
      throw new Error(data.error || 'Gagal meng-ingest teks manual SOP.');
    }
    return data.data;
  },

  async adminIngestDb(tableName: string, records: any[]): Promise<AdminIngestDbResult> {
    const res = await fetch(`${API_BASE}/admin/ingest-db`, {
      method: 'POST',
      headers: { 'Content-Type': 'application/json' },
      body: JSON.stringify({ tableName, records }),
    });

    const data = await res.json();
    if (!res.ok || !data.success) {
      throw new Error(data.error || 'Gagal meng-ingest baris database.');
    }
    return data.data;
  },

  // =========================================================================
  // 👤 USER API CALLS
  // =========================================================================
  async userChat(question: string, imageBase64?: string): Promise<UserChatResult> {
    const res = await fetch(`${API_BASE}/user/chat`, {
      method: 'POST',
      headers: { 'Content-Type': 'application/json' },
      body: JSON.stringify({ question, imageBase64 }),
    });

    const data = await res.json();
    if (!res.ok || !data.success) {
      throw new Error(data.error || 'Gagal mengeksekusi chat AI.');
    }
    return data.data;
  },

  async userAudioSpeak(text: string, voice = 'nova'): Promise<Blob> {
    const res = await fetch(`${API_BASE}/user/audio/speak`, {
      method: 'POST',
      headers: { 'Content-Type': 'application/json' },
      body: JSON.stringify({ text, voice }),
    });

    if (!res.ok) {
      const err = await res.json().catch(() => ({}));
      throw new Error(err.error || 'Gagal menggenerasi suara MP3.');
    }

    return await res.blob();
  },
};
```

---

## 2. 🛡️ Portal Admin (`src/components/AdminPortal.tsx`)

Komponen portal admin untuk mengunggah dokumen PDF/TXT, memasukkan teks manual SOP, dan meng-ingest baris database:

```tsx
// src/components/AdminPortal.tsx
import React, { useState } from 'react';
import { Upload, FileText, Database, CheckCircle2, AlertCircle, Loader2 } from 'lucide-react';
import { api, AdminIngestFileResult } from '../services/api';

export const AdminPortal: React.FC = () => {
  const [activeTab, setActiveTab] = useState<'file' | 'manual' | 'db'>('file');
  const [loading, setLoading] = useState(false);
  const [result, setResult] = useState<any | null>(null);
  const [error, setError] = useState<string | null>(null);

  // Form States
  const [selectedFile, setSelectedFile] = useState<File | null>(null);
  const [manualTitle, setManualTitle] = useState('');
  const [manualContent, setManualContent] = useState('');

  const handleFileUpload = async (e: React.FormEvent) => {
    e.preventDefault();
    if (!selectedFile) return;

    setLoading(true);
    setError(null);
    setResult(null);

    try {
      const data = await api.adminIngestFile(selectedFile);
      setResult(data);
    } catch (err: any) {
      setError(err.message);
    } finally {
      setLoading(false);
    }
  };

  return (
    <div className="max-w-4xl mx-auto space-y-6">
      {/* Header Tabs Admin */}
      <div className="flex bg-slate-900/80 p-1.5 rounded-2xl border border-slate-800 backdrop-blur-md">
        <button
          onClick={() => setActiveTab('file')}
          className={`flex-1 flex items-center justify-center gap-2 py-3 rounded-xl text-sm font-semibold transition-all ${
            activeTab === 'file'
              ? 'bg-blue-600 text-white shadow-lg shadow-blue-600/30'
              : 'text-slate-400 hover:text-slate-200'
          }`}
        >
          <Upload className="w-4 h-4" />
          1. Upload File PDF / TXT
        </button>
        <button
          onClick={() => setActiveTab('manual')}
          className={`flex-1 flex items-center justify-center gap-2 py-3 rounded-xl text-sm font-semibold transition-all ${
            activeTab === 'manual'
              ? 'bg-blue-600 text-white shadow-lg shadow-blue-600/30'
              : 'text-slate-400 hover:text-slate-200'
          }`}
        >
          <FileText className="w-4 h-4" />
          2. Input Knowledge Manual
        </button>
      </div>

      {/* Form Content */}
      <div className="bg-slate-900/60 border border-slate-800/80 rounded-2xl p-6 backdrop-blur-md">
        {activeTab === 'file' && (
          <form onSubmit={handleFileUpload} className="space-y-4">
            <div className="border-2 border-dashed border-slate-700 hover:border-blue-500 rounded-xl p-8 text-center transition-colors">
              <input
                type="file"
                accept=".pdf,.txt,.md"
                onChange={(e) => setSelectedFile(e.target.files?.[0] || null)}
                className="hidden"
                id="file-upload-input"
              />
              <label htmlFor="file-upload-input" className="cursor-pointer flex flex-col items-center gap-3">
                <Upload className="w-10 h-10 text-blue-400" />
                <span className="text-sm font-medium text-slate-300">
                  {selectedFile ? selectedFile.name : 'Klik untuk memilih file PDF / TXT'}
                </span>
              </label>
            </div>
            <button
              type="submit"
              disabled={!selectedFile || loading}
              className="w-full py-3 bg-blue-600 hover:bg-blue-500 disabled:opacity-50 text-white font-bold rounded-xl flex items-center justify-center gap-2 shadow-lg shadow-blue-600/20"
            >
              {loading ? <Loader2 className="w-5 h-5 animate-spin" /> : 'Mulai Ingesti File Dokumen'}
            </button>
          </form>
        )}
      </div>
    </div>
  );
};
```

---

## 3. 👤 Portal User (`src/components/UserPortal.tsx`)

Komponen portal user untuk tanya jawab AI RAG Multimodal, Streaming SSE, dan Text-to-Speech:

```tsx
// src/components/UserPortal.tsx
import React, { useState } from 'react';
import { Send, Bot, User, Volume2, Sparkles, Image as ImageIcon } from 'lucide-react';
import { api, UserChatResult } from '../services/api';

export const UserPortal: React.FC = () => {
  const [question, setQuestion] = useState('');
  const [loading, setLoading] = useState(false);
  const [chatHistory, setChatHistory] = useState<Array<{ role: 'user' | 'assistant'; text: string; images?: any[] }>>([]);

  const handleSendChat = async (e: React.FormEvent) => {
    e.preventDefault();
    if (!question.trim()) return;

    const userQ = question;
    setQuestion('');
    setChatHistory((prev) => [...prev, { role: 'user', text: userQ }]);
    setLoading(true);

    try {
      const res: UserChatResult = await api.userChat(userQ);
      setChatHistory((prev) => [
        ...prev,
        { role: 'assistant', text: res.answer, images: res.images },
      ]);
    } catch (err: any) {
      setChatHistory((prev) => [
        ...prev,
        { role: 'assistant', text: `❌ Error: ${err.message}` },
      ]);
    } finally {
      setLoading(false);
    }
  };

  return (
    <div className="max-w-4xl mx-auto flex flex-col h-[700px] bg-slate-900/60 border border-slate-800 rounded-2xl overflow-hidden backdrop-blur-md">
      {/* Chat Messages */}
      <div className="flex-1 overflow-y-auto p-6 space-y-4">
        {chatHistory.map((msg, idx) => (
          <div key={idx} className={`flex gap-3 ${msg.role === 'user' ? 'justify-end' : 'justify-start'}`}>
            {msg.role === 'assistant' && (
              <div className="p-2 bg-blue-600/20 border border-blue-500/30 rounded-xl text-blue-400 h-fit">
                <Bot className="w-5 h-5" />
              </div>
            )}
            <div
              className={`max-w-[80%] p-4 rounded-2xl text-sm ${
                msg.role === 'user'
                  ? 'bg-blue-600 text-white rounded-br-none'
                  : 'bg-slate-800/80 text-slate-200 border border-slate-700 rounded-bl-none'
              }`}
            >
              <p className="whitespace-pre-wrap">{msg.text}</p>
              {msg.images && msg.images.length > 0 && (
                <div className="mt-3 grid grid-cols-2 gap-2">
                  {msg.images.map((img, i) => (
                    <img key={i} src={img.url} alt="RAG HD Visual" className="rounded-lg border border-slate-700 object-cover max-h-48 w-full" />
                  ))}
                </div>
              )}
            </div>
          </div>
        ))}
      </div>

      {/* Input Box */}
      <form onSubmit={handleSendChat} className="p-4 border-t border-slate-800 bg-slate-950/80 flex gap-2">
        <input
          type="text"
          value={question}
          onChange={(e) => setQuestion(e.target.value)}
          placeholder="Tanyakan materi seputar dokumen PDF..."
          className="flex-1 bg-slate-900 border border-slate-800 rounded-xl px-4 py-3 text-sm text-white placeholder-slate-500 focus:outline-none focus:border-blue-500"
        />
        <button
          type="submit"
          disabled={loading || !question.trim()}
          className="px-5 py-3 bg-blue-600 hover:bg-blue-500 text-white rounded-xl font-bold flex items-center gap-2 transition-all shadow-lg shadow-blue-600/30 disabled:opacity-50"
        >
          <Send className="w-4 h-4" />
        </button>
      </form>
    </div>
  );
};
```

---

## 4. 🎛️ Dashboard Utama (`src/App.tsx`)

Layout dashboard utama dengan Navbar dan Role Switcher Pill:

```tsx
// src/App.tsx
import React, { useState } from 'react';
import { AdminPortal } from './components/AdminPortal';
import { UserPortal } from './components/UserPortal';
import { ShieldCheck, Bot, Sparkles } from 'lucide-react';

export const App: React.FC = () => {
  const [role, setRole] = useState<'admin' | 'user'>('admin');

  return (
    <div className="min-h-screen bg-slate-950 text-slate-100 flex flex-col">
      <header className="sticky top-0 z-50 glass-panel border-b border-slate-800 px-6 py-4">
        <div className="max-w-7xl mx-auto flex items-center justify-between">
          <div className="flex items-center gap-3">
            <div className="p-2.5 bg-gradient-to-tr from-blue-600 to-indigo-600 rounded-xl text-white">
              <Sparkles className="w-6 h-6" />
            </div>
            <div>
              <span className="font-extrabold text-lg text-white">AI Knowledge Core System</span>
              <span className="ml-2 px-2 py-0.5 text-[10px] font-bold bg-blue-500/20 text-blue-400 border border-blue-500/30 rounded-full">
                @badriana/ai-core v0.2.6
              </span>
            </div>
          </div>

          <div className="flex bg-slate-900 p-1.5 rounded-xl border border-slate-800">
            <button
              onClick={() => setRole('admin')}
              className={`flex items-center gap-2 px-5 py-2 rounded-lg text-xs font-bold ${
                role === 'admin' ? 'bg-blue-600 text-white' : 'text-slate-400'
              }`}
            >
              <ShieldCheck className="w-4 h-4" />
              Role Admin
            </button>
            <button
              onClick={() => setRole('user')}
              className={`flex items-center gap-2 px-5 py-2 rounded-lg text-xs font-bold ${
                role === 'user' ? 'bg-blue-600 text-white' : 'text-slate-400'
              }`}
            >
              <Bot className="w-4 h-4" />
              Role User
            </button>
          </div>
        </div>
      </header>

      <main className="flex-1 py-8 px-4">
        {role === 'admin' ? <AdminPortal /> : <UserPortal />}
      </main>
    </div>
  );
};

export default App;
```

---
*Dokumentasi Arsitektur Resmi Frontend React (`@badriana/ai-core`)*
