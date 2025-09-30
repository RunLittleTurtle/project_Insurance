Insurance Real Time Voice AI Form Completion - 29/09/2025

# Checklist MVP - High Level

Cette checklist guide l'implémentation du MVP avec les éléments essentiels pour démontrer la valeur.

## Phase 1 : Infrastructure Conversationnelle

- [ ] **Connexion OpenAI Realtime API**
  - WebSocket persistant avec gestion événements
  - Configuration session (voix, VAD, formats audio)

- [ ] **State Management**
  - Structure State Pydantic (6 champs de base)
  - Persistence en mémoire pour session active

- [ ] **Interface Audio**
  - Intégration agent-chat-ui avec WebSocket audio
  - Capture/lecture audio navigateur (getUserMedia)

- [ ] **Reconnexion Basique**
  - Détection déconnexion et reconnexion auto
  - Sauvegarde contexte conversation

## Phase 2 : Extraction et Validation

- [ ] **Modèles Pydantic**
  - FieldSchema avec validators (nom, police, adresse, tel, date, raison)
  - ExtractedField avec 3 états: validé / tentative (needs_confirmation) / absent
  - ValidationResult combinant Pydantic + Claude

- [ ] **Claude Sonnet 4.5 Integration**
  - Client API avec prompts extraction structurés
  - Parsing JSON et validation Pydantic

- [ ] **Extraction Multi-Champs**
  - Double validation (Claude → Pydantic)
  - Identification champs manquants
  - Extraction bonus opportuniste

- [ ] **Validation Pipeline**
  - Quick Pydantic check (< 50ms) pour fast response
  - Validation sémantique Claude (parallèle, masquée par TTS)
  - Retry conversationnel avec messages pédagogiques

## Phase 3 : Optimisation Latence

- [ ] **Fast Response Architecture**
  - Réponses immédiates < 200ms (quick check + accusé)
  - Processing Claude pendant que TTS parle
  - Calcul et logging latences perçues

- [ ] **Question Generation Dynamique**
  - Priorisation champs (critiques first)
  - Questions complètes pour champs absents
  - Confirmations rapides pour champs tentative (bonus)

## Phase 4 : Export et Logging

- [ ] **Google Sheets Export**
  - Auth service account
  - Export structuré fin de session
  - Backup local si échec

- [ ] **Observabilité Basique**
  - Logging State transitions
  - Métriques durée/questions/latence
  - Traces conversations pour debugging

## Tests Critiques MVP

- [ ] **Scénario Happy Path**
  - Question ouverte → extraction 4 champs → 2 questions ciblées → export

- [ ] **Scénario Extraction Bonus**
  - Détection téléphone dans réponse adresse → confirmation

- [ ] **Scénario Retry**
  - Date incomplète → retry pédagogique → validation

- [ ] **Performance**
  - Latence perçue < 200ms sur toutes interactions
  - Claude processing 100% masqué par TTS 
