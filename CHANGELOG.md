# Changelog

All notable changes to `@badrian/rag-core` will be documented in this file.

The format is based on [Keep a Changelog](https://keepachangelog.com/en/1.0.0/),
and this project adheres to [Semantic Versioning](https://semver.org/spec/v2.0.0.html).

## [0.2.0] - 2026-08-12

### Added & Enhanced
- 👁️ **Multimodal Vision Chat**: Full support for image URL & Base64 inputs in `ai.chat.generate()` and `ai.chat.stream()`.
- 🔍 **Image Intent Recognition (`imageAnalysis`)**: Automatically classifies vision intent into 8 categories (`person_face`, `object_detection`, `landmark_map`, `document_text`, `exam_question`, `whiteboard`, `diagram`, `general_photo`).
- 🗄️ **Company Database RAG Ingestion**: Added `formatDataRecordsForIngestion()` utility to format 40+ database tables with automatic sensitive key redaction (`password`, `secret`, `token`).
- 🔊 **Audio Speech Synthesis & Transcription**: Added `ai.audio.speak()` (Text-to-Speech MP3) and `ai.audio.transcribe()` (Speech-to-Text Whisper AI).
- 🏛️ **5 Framework Architecture Tutorials**: Created comprehensive 1-to-1 matching backend guides for Express.js (`express.example.md`), NestJS (`nestjs.example.md`), Next.js App Router (`nextjs.example.md`), Fastify (`fastify.example.md`), and Koa.js (`koa.example.md`).
- 🎨 **React Frontend Integration Guide**: Created `react.example.md` featuring `useAICore.ts` custom hook and 7 matching UI components.
- 🛡️ **TypeScript Improvements**: Added node buffer type imports and `performance` from `node:perf_hooks` for 100% zero-error compilation across all environments.

## [0.1.0] - 2026-08-06

### Initial Release

- 🚀 Initial release of `@badrian/rag-core` RAG Ingestion Engine.
- 📦 Public API: Implemented `parse()`, `chunk()`, `createDocument()`, `createQuery()`.
- 📄 Document Parsers: Support for TXT, PDF text streams, and Base64 Data URIs.
- ✂️ Chunk Engine: Recursive splitting by paragraph (`\n\n`), sentence (`. `, `! `, `? `, `\n`), and character fallback.
- 🧩 Embedding Engine: Dependency Injection wrapper for user-supplied embedding functions with strict vector validation.
- 🛑 Custom Error System: Strongly-typed custom errors extending `Error`.
- 🧹 Utility Layer: Text normalization (CRLF to LF, tab/space collapsing, paragraph preservation) and RFC 4122 v4 UUID generator.
- 🧪 Testing: Full Vitest unit test suite covering all modules.
