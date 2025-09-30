Insurance Real Time Voice AI Form Completion - 29/09/2025

# Catalogue des Nodes LangGraph

### Nodes de Gestion de Session

#### 1. initialize_session

- **Rôle** : Établir la connexion WebSocket Realtime et initialiser l'état conversationnel
- **Input** : Aucun (point d'entrée du graph)
- **Traitement** :
  - Génère un session_id unique (UUID)
  - Établit la connexion WebSocket persistante avec l'API Realtime d'OpenAI
  - Envoie l'événement `session.update` avec configuration (modèle, voix, paramètres audio)
  - Charge la définition des champs à collecter (schema Pydantic)
  - Initialise le State avec champs vides, compteurs de retry à zéro
  - Configure la détection de tour de parole (turn detection) pour gérer les pauses naturelles
- **Output** : State contenant websocket_session, session_id, fields_schema, extracted_fields={}, retry_counts={}, conversation_history=[]
- **Edges** : Connexion directe vers `open_question`
- **Note** : La connexion WebSocket reste ouverte durant tout l'entretien. Si elle se ferme, le graph doit gérer la reconnexion avec restauration du contexte.

### Nodes Conversationnels (Realtime API)

#### 2. open_question

- **Rôle** : Poser la question initiale ouverte pour capturer un maximum de contexte
- **Input** : State avec websocket_session active
- **Traitement** :
  - Construit le prompt de question ouverte : "Pour quelle raison nous appelez-vous aujourd'hui?"
  - Envoie l'événement `conversation.item.create` avec le message système
  - Envoie l'événement `response.create` pour déclencher la réponse TTS de Realtime
  - Attend la réponse complète de l'utilisateur (détection de fin de tour de parole)
- **Output** : State mis à jour avec current_question="open", waiting_for_response=True
- **Edges** : Connexion directe vers `realtime_conversation` (capture de la réponse)
- **Note** : Cette question est intentionnellement large pour encourager une réponse riche en informations

#### 3. realtime_conversation

- **Rôle** : Gérer l'échange audio bidirectionnel en temps réel via WebSocket
- **Input** : State avec websocket_session et contexte conversationnel
- **Traitement** :
  - Reçoit les événements audio streaming de l'utilisateur via WebSocket
  - L'API Realtime transcrit automatiquement (STT intégré) pendant que l'utilisateur parle
  - Détecte la fin du tour de parole (pause significative ou marqueur explicite)
  - Capture la transcription complète dans `conversation.item.input_audio_transcription.completed`
  - Stocke dans l'historique conversationnel
  - Gère les interruptions potentielles (utilisateur coupe le système)
- **Output** : State avec current_transcription (texte), audio_duration, conversation_history mis à jour
- **Edges** :
  - Après question ouverte → vers `extract_fields`
  - Après question ciblée → vers `validate_pydantic`
  - Après retry → vers `validate_pydantic`
- **Note** : Ce node est réutilisé à plusieurs endroits du flow - c'est le cœur de l'interaction temps réel

#### 4. ask_next_question

- **Rôle** : Formuler et poser soit une question ciblée standard, soit une question de confirmation pour un champ bonus
- **Input** : State avec fields_schema, extracted_fields (incluant champs avec needs_confirmation)
- **Traitement intelligent en 2 cas** :
  - **Cas A - Champ avec bonus tentative** : Si le prochain champ manquant a déjà une valeur stockée avec needs_confirmation=True (issu d'une extraction bonus précédente), génère une question de confirmation au lieu d'une question complète. Exemple : au lieu de "Quel est votre numéro de téléphone?", demande "Pour confirmer, je peux vous joindre au 514-555-1234?" Cette approche est plus naturelle et efficace - l'utilisateur répond juste "Oui" ou "C'est ça" au lieu de répéter l'information complète.
  - **Cas B - Champ vraiment manquant** : Si le champ n'a aucune valeur, génère une question ciblée contextuelle via un prompt template. Les questions sont formulées naturellement en tenant compte du contexte conversationnel (ce qui a déjà été dit, le ton de la conversation). Exemple pour "adresse" : "Parfait Marie, quelle est votre adresse actuelle?" plutôt que le robotique "Veuillez entrer votre adresse."
  - Marque le champ ciblé et le type de question (confirmation vs standard) dans le State pour que la validation sache comment interpréter la réponse.
- **Output** : State avec current_field_targeted, current_question (texte), question_type ("confirmation" ou "standard")
- **Edges** : Connexion directe vers `realtime_conversation` (capture réponse)
- **Note** : Cette intelligence au niveau de la formulation de question réduit dramatiquement les répétitions et améliore l'expérience utilisateur

#### 5. reask_question

- **Rôle** : Redemander une question de manière conversationnelle quand la validation a échoué
- **Input** : State avec validation_results (contient les raisons d'échec) et current_field_targeted
- **Traitement** :
  - Récupère les issues de validation depuis ValidationResult
  - Construit un message explicatif courtois basé sur le type d'erreur :
    - Format invalide : explique le format attendu avec exemple
    - Valeur manquante : guide sur ce qui est nécessaire
    - Incohérence : mentionne le problème détecté
  - Incrémente le retry_counter pour ce champ
  - Vérifie si retry_limit (3) est atteint
  - Si limite atteinte : marque le champ pour révision humaine et passe au suivant
  - Sinon : reformule la question avec l'explication
- **Output** : State avec retry_counts mis à jour, retry_message, needs_human_review (si limite)
- **Edges** :
  - Si retry possible → vers `realtime_conversation` (retry loop)
  - Si limite atteinte → vers `loop_start` (passe au prochain champ)
- **Note** : Maintient le ton conversationnel et courtois - jamais accusateur ou frustré

### Nodes de Processing et Validation

#### 6. quick_pydantic_check

- **Rôle** : Effectuer une vérification ultra-rapide du format pour éviter les silences conversationnels
- **Input** : State avec current_transcription et current_field_targeted
- **Traitement** :
  - Récupère le validator Pydantic pour le champ ciblé
  - Effectue un test de format basique et rapide (50ms maximum)
  - Ne vérifie QUE la structure : format, longueur, pattern regex
  - N'effectue AUCUNE validation sémantique (laissée à Claude)
  - Si échec évident (exemple: "abc" pour un champ numérique) → échec immédiat
  - Si format plausible → passe (même si potentiellement invalide sémantiquement)
- **Output** : State avec quick_check_result={passed: boolean, basic_format_ok: boolean}
- **Edges** :
  - Si passed=True → vers `send_acknowledgment` (feedback immédiat)
  - Si passed=False → vers `reask_question` (erreur de format évidente)
- **Note** : Ce node est la clé de la latence < 200ms. Il sacrifie la précision pour la vitesse - c'est un filtre grossier qui laisse passer les cas douteux pour que Claude décide. Mieux vaut un faux positif ici (corrigé après) qu'un silence gênant.

#### 7. send_acknowledgment

- **Rôle** : Fournir un feedback conversationnel immédiat pendant que le traitement lourd s'exécute en arrière-plan
- **Input** : State avec context (question ouverte ou question ciblée), current_field_targeted (optionnel)
- **Traitement** :
  - **Contexte A - Après question ouverte** : Génère un accusé long (3-4s) pour masquer extraction multi-champs. Exemples : "D'accord, laissez-moi noter tout ça...", "Parfait, je prends bien note de vos informations..."
  - **Contexte B - Après question ciblée** : Génère un accusé court (2-3s) pour masquer validation simple. Exemples : "D'accord, j'ai bien noté", "Parfait, je note ça", "Très bien"
  - Varie les formulations pour éviter la répétition robotique (essentiel pour naturalité)
  - Envoie immédiatement à l'API Realtime pour TTS
  - **Crucial** : Ne bloque PAS - retourne immédiatement après envoi sans attendre fin du TTS
  - Déclenche le traitement background en parallèle (non bloquant)
  - Enregistre le timestamp d'envoi et la durée estimée pour synchronisation
- **Output** : State avec acknowledgment_sent=True, ack_start_time, ack_expected_duration, ack_context
- **Edges** :
  - Depuis question ouverte → vers `extract_and_validate` (extraction parallèle)
  - Depuis question ciblée → vers `validate_in_background` (validation parallèle)
- **Note** : Ce node est utilisé DEUX fois dans le flow avec des durées différentes. Pour la question ouverte, l'accusé doit être plus long (1.2-1.5s d'extraction vs 0.8s de validation). C'est la clé de l'expérience MVP : transformer le traitement lourd invisible en maintenant une conversation fluide.

#### 8. validate_in_background

- **Rôle** : Effectuer la validation complète (Claude + Pydantic bonus) pendant que le système parle l'accusé de réception
- **Input** : State avec current_transcription, current_field_targeted, conversation_history, extracted_fields
- **Traitement** : Identique à l'ancien node `validate_with_context`, mais maintenant exécuté en parallèle avec le TTS
  - **Validation Claude du champ principal** : Sémantique, cohérence, completude (800ms)
  - **Extraction bonus contextuelle** : Cherche max 2 champs additionnels dans la réponse
  - **Validation Pydantic des bonus** : Teste format de chaque bonus (50ms par champ)
  - Attend que l'accusé de réception soit terminé avant de retourner (synchronisation)
  - Si l'accusé prend 2.5s et Claude prend 800ms → validation termine en avance, pas de latence perçue
  - Si Claude prend exceptionnellement 1.5s → ajoute seulement 1s après la fin de l'accusé (acceptable)
- **Output** : State avec primary_validation, bonus_fields_validated (identique structure précédente)
- **Edges** : Connexion directe vers `validation_check`
- **Note** : Ce node est le "travailleur lourd" qui s'exécute silencieusement pendant la parole du système. Architecture critique pour MVP : sacrifie un peu de temps CPU (parallelisation) pour éliminer la latence perçue par l'utilisateur.

#### 9. extract_and_validate

- **Rôle** : Extraire intelligemment tous les champs de la réponse initiale ET valider immédiatement chacun avec Pydantic
- **Input** : State avec current_transcription (réponse initiale longue de la question ouverte)
- **Traitement** :
  - **Phase A - Extraction par Claude** : Envoie la transcription complète à Claude Sonnet 4.5 avec un prompt structuré incluant le schema complet de tous les champs attendus. Claude analyse contextuellement et retourne un JSON avec tous les champs identifiables, leurs valeurs extraites, et un score de confiance pour chacun.
  - **Phase B - Validation Pydantic immédiate** : Pour chaque champ extrait par Claude, teste immédiatement contre le validator Pydantic correspondant. Si le format passe (exemple: "AB123456" match le pattern de numéro de police), le champ est accepté et stocké comme validé. Si Pydantic échoue (exemple: "AB12" trop court), le champ est rejeté silencieusement - il sera redemandé explicitement plus tard. Cette double couche (extraction Claude + validation Pydantic) garantit qu'on ne stocke que des données structurellement valides.
  - **Phase C - Agrégation** : Compile tous les champs qui ont passé les deux validations dans extracted_fields avec leurs métadonnées (confidence, timestamp, source).
- **Output** : State avec extracted_fields (dict contenant uniquement les champs validés par Pydantic), rejected_fields (pour logging/debug)
- **Edges** : Connexion directe vers `loop_start` (évaluation des champs manquants)
- **Note** : Cette extraction + validation unifiée peut remplir et valider 40-70% des champs en une passe, réduisant dramatiquement le nombre de questions nécessaires. La validation Pydantic immédiate évite de stocker des faux positifs que Claude pourrait extraire avec trop d'optimisme.

#### 10. validate_with_context

- **Rôle** : Valider le champ ciblé ET extraire opportunément des champs bonus, le tout avec double validation Pydantic + Claude
- **Input** : State avec current_transcription, current_field_targeted, conversation_history, extracted_fields
- **Traitement en 3 phases** :
  - **Phase 1 - Validation Pydantic du champ ciblé** : Récupère le validator Pydantic pour le champ demandé et teste la transcription. Si le format passe (exemple: "123 rue Principale, H3A 1B2" pour une adresse), extrait la valeur formatée. Si échoue (exemple: "123456" pour un numéro de police qui nécessite 2 lettres au début), marque comme invalide.
  - **Phase 2 - Validation et Extraction Contextuelle par Claude** : Envoie à Claude Sonnet 4.5 un prompt complet incluant la question posée, la réponse, l'historique conversationnel, et tous les champs déjà validés. Claude accomplit trois tâches simultanément : (A) Valide sémantiquement le champ ciblé (cohérence, completude, contradictions potentielles), (B) Cherche intelligemment d'autres informations dans la réponse (extraction bonus), et (C) Pour chaque bonus détecté, évalue la confiance d'extraction. Exemple : si la question était "Votre adresse?" et l'utilisateur dit "123 rue Principale, H3A 1B2, et mon téléphone est 514-555-1234", Claude valide l'adresse ET extrait le téléphone comme bonus avec confiance 0.92.
  - **Phase 3 - Validation Pydantic des Bonus** : Pour chaque champ bonus extrait par Claude (max 2 pour éviter surcharge), teste immédiatement avec le validator Pydantic correspondant. Si le bonus passe (exemple: "514-555-1234" match le format téléphone), il est accepté comme champ tentative avec needs_confirmation=True. Si Pydantic échoue, le bonus est rejeté silencieusement. Les bonus sont limités à confiance Claude > 0.85 pour éviter les faux positifs.
- **Output** : State avec primary_validation={is_valid, extracted_value, issues}, bonus_fields_validated={field_name: {value, confidence, needs_confirmation}}
- **Edges** : Connexion directe vers `validation_check` (décision sur champ primaire)
- **Note** : Ce node unifie la validation stricte (Pydantic) et l'intelligence contextuelle (Claude) tout en optimisant l'efficacité via l'extraction bonus. La latence reste acceptable car Claude traite validation + extraction en un seul appel API.

#### 11. validation_check

- **Rôle** : Décision finale sur la validité de la réponse en combinant Pydantic et Claude
- **Input** : State avec pydantic_validation et claude_validation
- **Traitement** :
  - Évalue les deux validations selon une logique ET avec pondération :
    - Pydantic DOIT passer (is_valid=True) - non négociable
    - Claude doit avoir confidence > 0.7 ET is_valid=True
  - Si les deux passent : retourne "valid"
  - Si l'une échoue :
    - Vérifie le retry_count pour ce champ
    - Si < 3 tentatives : retourne "invalid" (retry)
    - Si >= 3 tentatives : retourne "valid" (accepte malgré doute, flag pour révision)
  - Agrège les issues de validation pour le retry message
- **Output** : Décision de routing (string) : "valid" ou "invalid"
- **Edges** :
  - Si "valid" → vers `store_answer`
  - Si "invalid" → vers `reask_question`
- **Note** : Cette logique évite de bloquer l'utilisateur indéfiniment sur un champ problématique

### Nodes de Décision et Persistence

#### 12. loop_start

- **Rôle** : Déterminer si tous les champs obligatoires sont complètement validés ou s'il reste des questions/confirmations à faire
- **Input** : State avec fields_schema (définition des champs requis) et extracted_fields (champs remplis, incluant tentatives)
- **Traitement intelligent considérant 3 états possibles** :
  - Itère sur tous les champs marqués required=True dans le schema. Pour chaque champ requis, vérifie trois cas :
    - **Cas 1 - Champ absent** : N'existe pas du tout dans extracted_fields → ajoute à missing_fields
    - **Cas 2 - Champ tentative** : Existe mais avec validated=False et needs_confirmation=True (issu d'extraction bonus) → ajoute à missing_fields car nécessite confirmation
    - **Cas 3 - Champ validé** : Existe avec validated=True → considéré comme complet, ignore
  - Priorise les missing_fields selon importance (champs critiques d'abord) ET type (champs absents avant champs à confirmer, car poser une nouvelle question peut révéler d'autres bonus)
  - Retourne décision basée sur la liste finale :
    - Liste vide (tous champs existent ET tous validated=True) → "done"
    - Liste non vide (au moins un champ absent ou à confirmer) → "more"
- **Output** : Décision de routing (string) : "more" (questions/confirmations restantes) ou "done" (tout validé)
- **Edges** :
  - Si "more" → vers `ask_next_question` (continue loop avec question standard ou confirmation)
  - Si "done" → vers `export_to_sheets` (tous champs validés, termine entretien)
- **Note** : Ce node est le cœur du loop adaptatif - il détermine dynamiquement la fin ET distingue les champs vraiment manquants des champs à confirmer, permettant à ask_next_question de formuler la bonne question

#### 13. store_answer

- **Rôle** : Sauvegarder le champ primaire validé ET les champs bonus tentative dans le State
- **Input** : State avec current_field_targeted, primary_validation, bonus_fields_validated
- **Traitement en 2 parties** :
  - **Partie A - Stockage du champ primaire** : Récupère la valeur validée du champ ciblé et crée un objet FieldValue structuré contenant la donnée formatée, le score de confiance combiné Pydantic+Claude, l'horodatage de validation, le nombre de tentatives nécessaires, et un flag validated=True indiquant que ce champ est complètement validé. Ce champ est stocké dans extracted_fields[current_field_targeted] et compte comme officiellement complété.
  - **Partie B - Stockage des bonus comme tentatives** : Pour chaque champ bonus qui a passé la double validation Pydantic+Claude (max 2), crée un objet similaire mais avec validated=False et needs_confirmation=True. Ces champs bonus sont stockés dans extracted_fields mais ne comptent pas encore comme validés - ils nécessitent une confirmation explicite de l'utilisateur lors d'une prochaine question. Cette approche permet d'optimiser le flow tout en maintenant la rigueur de validation. Si un champ existe déjà dans extracted_fields (déjà validé précédemment), le bonus est ignoré pour éviter les écrasements.
  - Met à jour les statistiques de session et réinitialise les variables temporaires de validation.
- **Output** : State avec extracted_fields enrichi (champs validés + bonus tentative), statistics mises à jour
- **Edges** : Connexion directe vers `loop_start` (évalue champs manquants et champs à confirmer)
- **Note** : La distinction validated=True vs validated=False+needs_confirmation est cruciale pour que loop_start sache quels champs sont vraiment complets vs lesquels nécessitent confirmation

#### 14. export_to_sheets

- **Rôle** : Compiler et exporter les données validées vers Google Sheets
- **Input** : State avec extracted_fields (tous champs validés), session_id, timestamps
- **Traitement** :
  - Compile toutes les données en un dictionnaire structuré :
    - Métadonnées : session_id, start_time, end_time, total_duration
    - Champs : toutes les clé-valeurs de extracted_fields
    - Qualité : scores de confiance moyens, nombre de retries total, champs flaggés pour révision
  - Formate selon le schema Google Sheets (ordre des colonnes, types)
  - Appelle l'API Google Sheets pour append une nouvelle ligne
  - Attend la confirmation de succès (synchrone pour MVP)
  - En cas d'erreur d'export :
    - Log l'erreur avec toutes les données pour debugging
    - Sauvegarde locale de backup (JSON file temporaire)
    - Retourne error_state pour gestion d'erreur
- **Output** : State avec export_status="success" ou "failed", export_timestamp, sheets_row_id
- **Edges** : Connexion directe vers `END` (fin du graph)
- **Note** : L'export est bloquant pour le MVP - on ne termine pas tant que les données ne sont pas persistées