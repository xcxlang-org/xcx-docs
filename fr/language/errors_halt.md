# Gestion des Erreurs (Halt) XCX 3.1

XCX utilise un système `halt` structuré pour gérer les conditions d'exécution et les erreurs.

## Niveaux de Halt

| Niveau        | Comportement                                                 | Cas d'Usage                      |
|---------------|--------------------------------------------------------------|----------------------------------|
| `halt.alert`  | Affiche un message ; l'exécution continue.                   | Journalisation, avertissements non critiques |
| `halt.error`  | Affiche sur stderr ; abandonne le frame actuel.              | Erreurs logiques récupérables    |
| `halt.fatal`  | Affiche un message ; termine la VM immédiatement.            | Défaillance critique, violation de sécurité |

## Exemples

```xcx
halt.alert >! "Cache missed, fetching from DB...";

if (divisor == 0) then;
    halt.error >! "Division by zero!";
    return 0; --- Execution returns to caller from the recovery point
end;

if (NOT db.is_healthy()) then;
    halt.fatal >! "Database corrupted. Emergency shutdown.";
end;
```

## Paniques Sémantiques et à l'Exécution

Certaines opérations invalides résultent en une **panique** automatique (équivalente à `halt.fatal` ou `halt.error` selon le contexte) :
- **Division par zéro** (arithmétique)
- **Échec d'Analyse JSON** : Une chaîne invalide dans `json.parse()` déclenche une sortie immédiate de la VM.
- **Traversée de Chemin** : Utiliser `..` dans les méthodes `store` ou des chemins absolus hors de la racine du projet déclenche `halt.fatal`.
- **Profondeur de Récursion** : Dépasser 800 frames déclenche `halt.error`.