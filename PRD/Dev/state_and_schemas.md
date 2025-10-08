Insurance Real Time Voice AI Form Completion - 29/09/2025 	

# State and Schemas

### (BESOIN DE MODIFICATION DU SHÉMA LORSQUE LES CHAMPS SERONT RECONFIRMÉS)

## Gestion du State

Le State de ce système conversationnel contient plus d'informations que le MVP upload-audio car il doit gérer une conversation temps réel continue avec contexte.

### Identification et Session

- `session_id` : UUID unique pour cette session d'entretien
- `websocket_session` : Objet de connexion WebSocket Realtime persistante
- `start_timestamp` : Horodatage de début de session
- `end_timestamp` : Horodatage de fin (rempli à l'export)
- `is_complete` : Booléen indiquant entretien terminé

### Contexte Conversationnel

- `conversation_history` : Liste chronologique de tous les échanges (system messages, user responses, assistant responses)
- `current_question` : Texte de la dernière question posée
- `current_field_targeted` : Nom du champ actuellement en cours de collection
- `current_transcription` : Dernière transcription reçue de Realtime API

### Structure des Champs

- `fields_schema` : Définition Pydantic de tous les champs à collecter (types, formats, required flags)

- `extracted_fields` : Dictionnaire des champs avec **3 états possibles** :

  ```python
  {
    # État 1: Champ complètement validé
    "first_name": {
      "value": "Marie",
      "confidence": 0.95,
      "validated": True,
      "needs_confirmation": False,
      "retry_count": 1,
      "source": "targeted_question",
      "timestamp": "2025-01-15T10:23:45Z"
    },
  
    # État 2: Champ bonus tentative (extrait mais nécessite confirmation)
    "phone": {
      "value": "514-555-1234",
      "confidence": 0.88,
      "validated": False,
      "needs_confirmation": True,
      "source": "bonus_extraction",
      "timestamp": "2025-01-15T10:24:12Z"
    },
  
    # État 3: Champ absent (non présent dans le dict)
    # "address": n'existe pas encore
  }
  ```

- `missing_fields` : Liste des champs obligatoires non encore complètement validés (calculée par loop_start, inclut les champs absents ET les champs à confirmer)

### Validation

- `primary_validation` : Résultat de validation du champ ciblé

  - is_valid : booléen (AND logique Pydantic + Claude)
  - extracted_value : valeur formatée ou None
  - pydantic_passed : booléen spécifique Pydantic
  - claude_passed : booléen spécifique Claude
  - issues : liste unifiée des problèmes
  - confidence : float 0.0-1.0

- `bonus_fields_validated` : Dictionnaire des champs bonus extraits et validés lors de la dernière réponse

  ```python
  {
    "last_name": {
      "value": "Tremblay",
      "confidence": 0.90,
      "pydantic_passed": True,
      "needs_confirmation": True
    }
  }
  ```

  (Maximum 2 champs bonus par réponse)

### Gestion des Retries

- `retry_counts` : Dictionnaire comptant les tentatives par champ

  ```python
  {
    "phone_number": 2,
    "address": 1,
    ...
  }
  ```

- `fields_needing_review` : Liste des champs acceptés malgré échecs de validation (après max_retries)

### Export

- `export_status` : "pending", "in_progress", "success", "failed"
- `export_timestamp` : Horodatage de l'export Google Sheets
- `sheets_row_id` : ID de la ligne créée dans Sheets

Tous ces champs sont gérés dans une classe State Pydantic qui assure la cohérence des types tout au long du graph.





---



## Structures de Données

### RealtimeSessionConfig

Configuration de la session WebSocket Realtime API :

```python
{
  "model": "gpt-4o-realtime-preview",  # ou "gpt-realtime" en GA
  "voice": "alloy",  # voix pour le TTS
  "input_audio_format": "pcm16",
  "output_audio_format": "pcm16",
  "turn_detection": {
    "type": "server_vad",  # Voice Activity Detection côté serveur
    "threshold": 0.5,
    "prefix_padding_ms": 300,
    "silence_duration_ms": 500
  },
  "instructions": "Tu es un assistant d'assurance courtois et efficace..."
}
```

### FieldSchema (Pydantic)

Définition d'un champ à collecter :

- `field_name` : Identifiant unique du champ (string)
- `field_type` : Type de données (TEXT, EMAIL, PHONE, POLICY_NUMBER, ADDRESS, BOOLEAN)
- `required` : Booléen indiquant si obligatoire
- `pydantic_validator` : Regex ou validator Pydantic pour le format
- `claude_prompt` : Instructions spécifiques pour Claude sur comment valider ce champ
- `question_template` : Template pour générer la question ciblée
- `max_retries` : Nombre maximum de tentatives (défaut 3)

Exemple pour numéro de police :

```python
FieldSchema(
  field_name="policy_number",
  field_type="POLICY_NUMBER",
  required=True,
  pydantic_validator=r"^[A-Z]{2}\d{6,10}$",
  claude_prompt="Un numéro de police d'assurance canadien valide. Vérifier cohérence.",
  question_template="Quel est votre numéro de police?",
  max_retries=3
)
```

### ExtractedField

Structure d'un champ extrait et validé :

- `value` : La valeur proprement formatée (any type selon field_type)
- `confidence` : Score de confiance combiné Pydantic + Claude (0.0-1.0)
- `validated` : Booléen indiquant validation complète réussie
- `validation_timestamp` : Horodatage de la validation
- `retry_count` : Nombre de tentatives nécessaires pour valider
- `needs_review` : Flag si accepté après max_retries (pour révision humaine ultérieure)
- `extraction_source` : "open_question" ou "targeted_question" (traçabilité)

### ValidationResult

Résultat combiné de validation Pydantic + Claude :

- `is_valid` : Booléen final (ET logique des deux validations)
- `pydantic_passed` : Booléen spécifique à Pydantic
- `claude_passed` : Booléen spécifique à Claude
- `combined_confidence` : Score combiné (moyenne pondérée)
- `issues` : Liste unifiée de tous les problèmes détectés
- `suggested_retry_message` : Message pré-généré pour reask si validation échoue

