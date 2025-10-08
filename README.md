# Insurance Real Time Voice AI Form Completion

## Overview

A conversational voice AI system for automating insurance phone interviews using real-time voice interaction. Combines OpenAI's Realtime API with Claude Sonnet 4.5 to create natural, efficient interviews that collect and validate insurance information.

## Documentation

Complete project documentation is available in the **PRD** folder:

**User:**

- **[contexte.md](./PRD/User/contexte.md)** - Problem statement and solution overview
- **[userflow.md](./PRD/User/userflow.md)** - User interaction flows and experience design

**Dev:**

- **[MVP_architecture.md](./PRD/Dev/MVP_architecture.md)** - Complete MVP architecture, flow diagrams
- **[product_backlog.md](./PRD/Dev/product_backlog.md)** - Product backlog with EPICs and User Stories
- **[nodes.md](./PRD/Dev/nodes.md)** - Detailed catalog of all LangGraph nodes
- **[state_and_schemas.md](./PRD/Dev/state_and_schemas.md)** - Data schemas, state management, and Pydantic models

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
- Parallelisation of processing
- Complete audit trail with confidence scores

## Technology Stack

- OpenAI Realtime API (voice interaction)
- Claude Sonnet 4.5 (extraction and validation)
- Pydantic (structural validation)
- LangGraph (conversation orchestration)
- Google Sheets API (export)
