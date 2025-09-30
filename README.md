# Insurance Real Time Voice AI Form Completion

## Overview

A conversational voice AI system for automating insurance phone interviews using real-time voice interaction. Combines OpenAI's Realtime API with Claude Sonnet 4.5 to create natural, efficient interviews that collect and validate insurance information.

## The Problem

Insurance companies face challenges with traditional phone interviews: high operational costs, variable quality, limited availability, complex compliance, and scaling difficulties.

## The Solution

An AI assistant that automates pre-qualification interviews with:
- **Real-time conversation** via OpenAI Realtime API (WebSocket, STT/TTS)
- **Intelligent extraction** - captures multiple fields from open responses
- **Dual validation** - Pydantic (format) + Claude (semantic)
- **Optimized latency** - perceived response time < 200ms
- **Natural experience** - no awkward silences, conversational error handling

## Key Features

- Open question approach reduces interview time by 40-70%
- Bonus field extraction from user responses
- Adaptive questioning - only asks for missing information
- Structured export to Google Sheets
- Complete audit trail with confidence scores

## Documentation

Complete project documentation is available in the following files:

- **[contexte.md](./contexte.md)** - Problem statement and solution overview
- **[MVP.md](./MVP.md)** - Complete MVP architecture, flow diagrams, node catalog, and technical implementation details
- **[state_and_schemas.md](./state_and_schemas.md)** - Data schemas, state management, and Pydantic models
- **[userflow.md](./userflow.md)** - User interaction flows and experience design
- **[project_checklist.md](./project_checklist.md)** - Implementation checklist and development milestones

## Technology Stack

- OpenAI Realtime API (voice interaction)
- Claude Sonnet 4.5 (extraction and validation)
- Pydantic (structural validation)
- LangGraph (conversation orchestration)
- Google Sheets API (export)

---

*For detailed architecture, implementation guides, and complete specifications, please refer to the documentation files above.*
