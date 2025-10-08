Insurance Real Time Voice AI Form Completion - 29/09/2025

# MVP Product Backlog

## EPIC 1: Infrastructure Conversationnelle en Temps Réel

Mettre en place les fondations techniques pour permettre une conversation vocale bidirectionnelle fluide entre le système et l'utilisateur.

### US-1.1: Connexion à l'API Realtime d'OpenAI

En tant que développeur, je veux établir une connexion WebSocket persistante avec l'API Realtime d'OpenAI pour pouvoir échanger de l'audio en temps réel avec l'utilisateur.

Nodes implémentés: 1. initialize_session

Tâches:
- Créer le client WebSocket avec gestion des événements Realtime
- Configurer la session (modèle voix, VAD, formats audio)
- Gérer les événements de base (connection, error, close)

Critères d'acceptation:
- La connexion WebSocket s'établit avec succès au démarrage
- Les événements audio sont reçus sans perte de paquets
- La voix configurée est celle demandée (ex: alloy, echo)

### US-1.2: Gestion de l'état conversationnel

En tant que développeur, je veux une structure de données centrale qui garde en mémoire toutes les informations de la conversation en cours pour pouvoir y accéder depuis n'importe quel node du système.

Nodes implémentés: Structure partagée par tous les nodes (State global)

Tâches:
- Définir le State Pydantic avec les champs de base
- Implémenter la persistence en mémoire pour la session active
- Créer les accesseurs et mutateurs pour mettre à jour le State

Critères d'acceptation:
- Exemple de State: user_id, session_id, extracted_fields, conversation_history, retry_counts, current_field_targeted, validation_results
- Le State persiste pendant toute la durée de la session
- Les modifications du State sont propagées entre les nodes
- Le State peut être sérialisé en JSON pour debugging

### US-1.3: Interface audio dans le navigateur

En tant qu'utilisateur, je veux pouvoir parler et entendre le système directement depuis mon navigateur pour avoir une conversation naturelle sans équipement spécial.

Nodes implémentés: 2. open_question, 3. realtime_conversation

Tâches:
- Intégrer le WebSocket audio dans agent-chat-ui
- Implémenter la capture audio (getUserMedia)
- Implémenter la lecture audio (AudioContext)
- Ajouter les indicateurs visuels (microphone actif, système parle)

Critères d'acceptation:
- L'utilisateur peut autoriser l'accès au microphone
- L'audio capturé est envoyé au WebSocket en temps réel
- L'audio reçu est joué sans délai perceptible
- Les indicateurs visuels reflètent l'état de la conversation
- L'audio fonctionne avec Google chrome, Brave, Safari, Firefox, Edge

### US-1.4: Reconnexion automatique

En tant qu'utilisateur, je veux que le système se reconnecte automatiquement si la connexion est perdue pour ne pas avoir à recommencer mon entretien depuis le début.

Nodes implémentés: Extension de 1. initialize_session

Tâches:
- Détecter les déconnexions WebSocket
- Implémenter la logique de reconnexion avec backoff
- Sauvegarder le contexte conversationnel avant déconnexion
- Restaurer le contexte après reconnexion

Critères d'acceptation:
- Le système détecte une déconnexion en moins de 5 secondes
- La reconnexion est tentée automatiquement (max 3 tentatives)
- Le contexte conversationnel est préservé (champs extraits, historique)
- L'utilisateur est informé de la reconnexion ("Un instant, reconnexion...")

### US-1.5: Architecture LangGraph et orchestration des nodes

En tant que développeur, je veux créer le StateGraph LangGraph qui orchestre tous les nodes pour que le flow conversationnel s'exécute correctement de bout en bout.

Nodes implémentés: Création du graph principal + définition de tous les edges

Tâches:
- Créer le StateGraph avec le State Pydantic
- Ajouter tous les 14 nodes au graph
- Définir les edges fixes: 1→2, 2→3, 3→7, 7→9, 9→12, 4→3, 3→6, 6→7 ou 6→5, 7→8, 8→11, 11→13 ou 11→5, 13→12, 12→14
- Définir les conditional edges avec fonctions de routing: loop_start (12→4 ou 12→14), validation_check (11→13 ou 11→5), quick_check (6→7 ou 6→5)
- Implémenter les fonctions de routing pour chaque décision (retour de clés string)
- Compiler le graph et créer l'API d'invocation

Critères d'acceptation:
- Le graph contient tous les 14 nodes
- Les conditional edges retournent les bonnes clés selon le State ("more"/"done", "valid"/"invalid", "pass"/"fail")
- Le graph peut être compilé sans erreur
- Une invocation complète exécute le bon flow selon les données (traçabilité des transitions dans les logs)
- Le graph peut être visualisé dans Langsmith Studio

---

## EPIC 2: Extraction et Validation Intelligente

Permettre au système d'extraire plusieurs informations d'une seule réponse et de valider leur exactitude avec rigueur.

### US-2.1: Modèles de données avec validation

En tant que développeur, je veux des modèles Pydantic qui définissent la structure exacte des champs à collecter pour garantir que seules des données valides sont stockées.

Nodes implémentés: Utilisé par 6. quick_pydantic_check, 9. extract_and_validate, 10. validate_with_context

Tâches:
- Créer FieldSchema pour les champs (nom, police, adresse, téléphone, date, raison)
- Définir les validators Pydantic (formats, patterns regex, longueurs)
- Créer ExtractedField avec états: validé, tentative, absent
- Créer ValidationResult combinant résultats Pydantic et Claude

Critères d'acceptation:
- Chaque champ a un validator qui rejette les formats invalides
- Les numéros de police doivent commencer par 2 lettres + 6-10 chiffres
- Les codes postaux québécois sont validés (format H3A 1B2)
- Un champ peut être marqué comme tentative (needs_confirmation)

### US-2.2: Intégration avec Claude pour extraction

En tant que système, je veux utiliser Claude Sonnet 4.5 pour extraire intelligemment plusieurs champs d'une seule réponse utilisateur afin de réduire le nombre de questions à poser.

Nodes implémentés: Utilisé par 9. extract_and_validate, 10. validate_with_context

Tâches:
- Créer le client API Claude
- Créer les prompts d'extraction structurés (inclure schema des champs)
- Parser les réponses JSON de Claude
- Valider chaque champ extrait avec Pydantic

Critères d'acceptation:
- Claude reçoit la transcription complète et le schema des champs
- Claude retourne un JSON avec champs identifiés et scores de confiance
- Seuls les champs qui passent Pydantic sont acceptés
- Les champs rejetés sont ignorés silencieusement (redemandés plus tard)

### US-2.3: Extraction multi-champs de la réponse initiale

En tant qu'utilisateur, je veux pouvoir mentionner plusieurs informations dans ma première réponse pour ne pas avoir à répondre à une longue série de questions.

Nodes implémentés: 9. extract_and_validate

Tâches:
- Implémenter le node extract_and_validate
- Extraire tous les champs identifiables (Claude)
- Valider immédiatement chaque champ (Pydantic)
- Stocker les champs validés dans extracted_fields

Critères d'acceptation:
- Une réponse comme "Je m'appelle Marie, police AB123456, pour ajouter mon auto" extrait 3-4 champs
- Chaque champ extrait passe obligatoirement la validation Pydantic
- Le taux d'extraction multi-champs atteint 40-70%
- Les champs extraits sont marqués comme validés (validated)

### US-2.4: Pipeline de validation rapide et approfondie

En tant que système, je veux valider rapidement le format des réponses puis vérifier la cohérence sémantique en arrière-plan pour donner un feedback immédiat sans ralentir la conversation.

Nodes implémentés: 6. quick_pydantic_check, 8. validate_in_background, 11. validation_check, 5. reask_question

Tâches:
- Implémenter quick_pydantic_check (validation format 50ms)
- Implémenter validate_in_background (validation Claude pendant TTS)
- Combiner les deux validations dans validation_check
- Implémenter retry conversationnel avec messages pédagogiques

Critères d'acceptation:
- Le quick check retourne en moins de 50ms
- Les erreurs de format évidentes sont détectées immédiatement
- La validation Claude s'exécute pendant que le système parle
- Les messages de retry expliquent clairement le problème

### US-2.5: Extraction bonus opportuniste

En tant que système, je veux capturer des informations supplémentaires mentionnées par l'utilisateur même si ce n'était pas la question posée pour optimiser l'efficacité de l'entretien.

Nodes implémentés: 10. validate_with_context, 13. store_answer

Tâches:
- Détecter jusqu'à 2 champs bonus dans chaque réponse (Claude)
- Valider les bonus avec Pydantic
- Stocker les bonus comme tentative (needs_confirmation)
- Générer des questions de confirmation au lieu de questions complètes

Critères d'acceptation:
- Maximum 2 bonus par réponse pour éviter surcharge
- Bonus acceptés seulement si confiance Claude > 85%
- Bonus doivent passer validation Pydantic (sinon rejetés)
- Confirmation demandée: "Je peux vous joindre au 514-555-1234?" au lieu de "Quel est votre téléphone?"

---

## EPIC 3: Optimisation de la Latence Perçue

Éliminer les silences gênants pour créer une conversation fluide et réactive qui donne l'impression d'interagir avec un humain.

### US-3.1: Architecture de réponse rapide

En tant qu'utilisateur, je veux que le système me réponde en moins de 200ms après que j'ai fini de parler pour avoir l'impression d'une vraie conversation.

Nodes implémentés: 7. send_acknowledgment

Tâches:
- Implémenter send_acknowledgment avec réponse immédiate
- Générer des accusés longs (3-4s) après question ouverte
- Générer des accusés courts (2-3s) après question ciblée
- Varier les formulations pour éviter répétition robotique

Critères d'acceptation:
- Feedback envoyé en moins de 200ms après fin de parole utilisateur
- Les accusés durent assez longtemps pour masquer le processing Claude
- Pas de répétition des mêmes phrases ("D'accord", "Parfait", "Très bien" alternent)
- Aucun silence gênant perçu par l'utilisateur

### US-3.2: Processing en parallèle masqué par la parole

En tant que développeur, je veux exécuter les validations Claude pendant que le système parle pour que le temps de processing soit invisible à l'utilisateur.

Nodes implémentés: 8. validate_in_background (orchestration avec 7. send_acknowledgment)

Tâches:
- Lancer validate_in_background en parallèle du TTS
- Synchroniser la fin du TTS avec la fin de la validation
- Logger les latences (temps Claude, temps TTS, latence perçue)
- Optimiser pour que validation termine avant ou juste après le TTS

Critères d'acceptation:
- La validation Claude démarre dès l'envoi de l'accusé de réception
- Si validation termine avant le TTS: aucune latence ajoutée
- Si validation prend plus longtemps: attente maximale de 1s
- Les logs montrent latence perçue < 200ms sur 90% des interactions

### US-3.3: Génération dynamique de questions

En tant que système, je veux poser des questions complètes pour les champs vraiment manquants et des confirmations rapides pour les champs bonus afin de minimiser le temps d'entretien.

Nodes implémentés: 12. loop_start, 4. ask_next_question

Tâches:
- Implémenter la priorisation des champs (critiques first)
- Distinguer champs absents vs champs tentative (bonus)
- Générer questions complètes pour absents
- Générer confirmations rapides pour tentatives

Critères d'acceptation:
- Les champs critiques (nom, police, raison) sont demandés en premier
- Les champs absents déclenchent: "Quelle est votre adresse?"
- Les champs tentative déclenchent: "Pour confirmer, 123 rue Principale?"
- Le nombre moyen de questions complètes est de 5-7 maximum

---

## EPIC 4: Export et Observabilité

Sauvegarder les données collectées de manière structurée et permettre le debugging efficace du système.

### US-4.1: Export vers Google Sheets

En tant qu'assureur, je veux que les informations collectées soient automatiquement exportées dans Google Sheets pour pouvoir les consulter et les traiter facilement.

Nodes implémentés: 14. export_to_sheets

Tâches:
- Configurer l'authentification service account Google
- Créer le client Google Sheets API
- Formatter les données selon le schema (colonnes définies)
- Implémenter backup local JSON si export échoue

Critères d'acceptation:
- Chaque session complète ajoute une ligne dans le Sheet
- Les colonnes incluent: session_id, timestamp, tous les champs, scores de confiance
- L'export est synchrone (système attend confirmation)
- En cas d'échec, un fichier JSON backup est créé localement

### US-4.2: Logging et métriques

En tant que développeur, je veux des logs détaillés et des métriques pour comprendre comment le système performe et identifier les problèmes rapidement.

Nodes implémentés: Instrumentation ajoutée à tous les nodes

Tâches:
- Logger toutes les transitions de State
- Calculer et logger: durée totale, nombre de questions, latences
- Logger les transcriptions complètes pour chaque conversation
- Créer un format de log structuré (JSON)

Critères d'acceptation:
- Chaque transition de node est loggée avec timestamp et Langsmith tracing
- Les métriques incluent: latence perçue moyenne, taux d'extraction multi-champs, taux de retry
- Les conversations complètes sont sauvegardées pour replay/debugging
- Les logs sont faciles à parser et analyser

---

## EPIC 5: Validation du MVP Q&A

Vérifier que le système fonctionne de bout en bout et respecte les critères de qualité définis.

### US-5.1: Scénario happy path

En tant que testeur, je veux exécuter un entretien complet typique pour vérifier que le flow de base fonctionne sans erreur.

Nodes testés: Flow complet (1→2→3→7→9→12→4→3→6→7→8→11→13→12→14)

Critères d'acceptation:
- Question ouverte: "Bonjour, je m'appelle Marie Tremblay, police AB123456, pour mon auto"
- Extraction: 3 champs validés (nom, police, type assurance)
- 2-3 questions ciblées pour champs manquants (adresse, téléphone, date)
- Export réussi vers Google Sheets avec toutes les données

### US-5.2: Scénario extraction bonus

En tant que testeur, je veux vérifier que le système capture et confirme correctement les informations bonus mentionnées spontanément.

Nodes testés: Focus sur 10. validate_with_context, 13. store_answer, 4. ask_next_question

Critères d'acceptation:
- Question: "Quelle est votre adresse?"
- Réponse: "123 rue Principale, H3A 1B2, et mon téléphone est 514-555-1234"
- Le système stocke l'adresse comme validée
- Le système stocke le téléphone comme tentative (needs_confirmation=True)
- Prochaine question: "Pour confirmer, je peux vous joindre au 514-555-1234?"

### US-5.3: Scénario retry pédagogique

En tant que testeur, je veux vérifier que le système gère gracieusement les erreurs de format et guide l'utilisateur vers une réponse valide.

Nodes testés: Focus sur 6. quick_pydantic_check, 11. validation_check, 5. reask_question

Critères d'acceptation:
- Question: "Quel est votre numéro de police?"
- Réponse invalide: "123456" (manque les lettres)
- Message retry: "Nos numéros commencent par 2 lettres suivies de chiffres, comme AB123456"
- Réponse corrigée: "AB123456"
- Validation réussie après retry

### US-5.4: Performance de latence

En tant que testeur, je veux mesurer les latences réelles pour confirmer que l'expérience est fluide.

Nodes testés: Focus sur 7. send_acknowledgment, 8. validate_in_background (timing)

Critères d'acceptation:
- Latence perçue < 200ms sur toutes les interactions (90% du temps)
- Le processing Claude est masqué à 100% par le TTS
- Aucun silence de plus de 500ms pendant la conversation
- L'entretien complet dure moins de 5 minutes pour 6 champs 
