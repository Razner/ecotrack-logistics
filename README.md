# EcoTrack Logistics - Projet fil rouge

Lancer le docker compose : docker compose up -d
Stoper les processus : docker compose down

# Questions TP1 : 

1/ Les données SQL sont-elles toujours présentes ? : Oui 

# Questions TP2 :

2. Analyse
Quels sont les problèmes potentiels ?
- Redondance d’informations
- Difficulté de mise à jour
- Risque d’incohérence

3. Normalisation
Quel Problème ?
- Si on change un conducteur on aura des modifications multiples
- Donnée dupliquée

5. Le Challenge de l’Ingénieur
Que se passe-t-il si on supprime un entrepôt utilisé ?
On a une erreur de contrainte
Est-ce toujours une bonne idée ?
Non ce n'est pas une bonne idée ça peut créer des problèmes de cohérences des données.

![Schéma sans route](./schema_sans_route.png)
![Schéma avec route](./schema_avec_route.png)

# Questions TP3 : 

> Partie 4 :

Quel résultat avez-vous obtenu ?
SQL Error [42501]: ERROR: must be owner of table entrepots

Pourquoi cette opération est-elle interdite ?
Parce que l’utilisateur readonly n’a que des droits de lecture. Il ne peut ni modifier ni supprimer des tables. La commande DROP TABLE est donc refusée.

Quel principe de sécurité est appliqué ?
Le principe du moindre privilège : chaque rôle ne reçoit que les droits nécessaires, rien de plus. readonly = lecture uniquement.

# Questions TP4 :

> Partie 4 :

Comment se comporte l’insertion ?
Une fois l'insertion faite il me renvoi une erreur puisque au préalable, j'ai fait une fonction de validation où on peut
pas entrée une valeur inférieur ou égale à 0 pour le champ distance_km du coup la fonction check_distance() se déclenche
et dans DBeaver ça me renvoie ça : SQL Error [P0001]: ERROR: Distance invalide
                                   Where: PL/pgSQL function check_distance() line 4 at RAISE

> Partie 5 :

Créer un trigger qui :
● empêche un trajet > 2000 km
● log l’erreur

>Les fonctions :

```
CREATE OR REPLACE FUNCTION distance_inf_2000()
RETURNS TRIGGER AS $$
BEGIN
IF NEW.distance_km > 2000 THEN
INSERT INTO log_erreurs_distance (trajet_id, message)
VALUES (NEW.id, 'Distance invalide : ' || NEW.distance_km);
RAISE EXCEPTION 'Distance invalide : % km', NEW.distance_km;
END IF;
RETURN NEW;
END;
$$ LANGUAGE plpgsql;
```

```
CREATE OR REPLACE FUNCTION log_erreurs_distance()
RETURNS TRIGGER AS $$
BEGIN
INSERT INTO log_erreurs_distance (trajet_id, message)
VALUES (NEW.id, 'Distance invalide : ' || NEW.distance_km || ' km');
RETURN NEW;
END;
$$ LANGUAGE plpgsql;
```

> Les triggers : 

```
CREATE TRIGGER trg_distance_inf_2000
BEFORE INSERT OR UPDATE ON trajets
FOR EACH ROW
EXECUTE FUNCTION distance_inf_2000();
```

```
CREATE TRIGGER trg_log_erreurs_distance
AFTER INSERT OR UPDATE ON trajets
FOR EACH ROW
WHEN (NEW.distance_km > 2000)
EXECUTE FUNCTION log_erreurs_distance();
```

> Partie 6 :

● Ajouter UPDATE trigger

```
CREATE OR REPLACE FUNCTION log_update_trajet()
RETURNS TRIGGER AS $$
BEGIN
INSERT INTO log_updates_trajets (trajet_id, ancienne_distance_km, nouvelle_distance_km)
VALUES (OLD.id, OLD.distance_km, NEW.distance_km);
RETURN NEW;
END;
$$ LANGUAGE plpgsql;
```

```
CREATE TABLE IF NOT EXISTS log_updates_trajets (
    id SERIAL PRIMARY KEY,
    trajet_id INT,
    ancienne_distance_km INT,
    nouvelle_distance_km INT,
    date_modif TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);
```

```
CREATE TRIGGER trg_log_update_trajet
AFTER UPDATE ON trajets
FOR EACH ROW
EXECUTE FUNCTION log_update_trajet();
```

Voila ce que ça donne après un update d'un trajet

![Update d'un trajet](./update_trajet.png)

● Gérer DELETE

```
CREATE TABLE IF NOT EXISTS log_delete_trajets (
    id SERIAL PRIMARY KEY,
    trajet_id INT,
    distance_km INT,
    date_suppression TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);
```

```
CREATE OR REPLACE FUNCTION log_delete_trajet()
RETURNS TRIGGER AS $$
BEGIN
    INSERT INTO log_delete_trajets (trajet_id, distance_km)
    VALUES (OLD.id, OLD.distance_km);

    RETURN OLD;
END;
$$ LANGUAGE plpgsql;
```

```
CREATE TRIGGER trg_log_delete_trajet
BEFORE DELETE ON trajets
FOR EACH ROW
EXECUTE FUNCTION log_delete_trajet();
```

Voila ce que donne les logs après le delete d'un trajets

![Delete d'un trajet](./delete_trajets.png)

● Ajouter gestion d’erreurs avancée

Pour cette partie la, on la fait dans la partie 5 où on a créer une fonction qui interdit d'avoir une distance d'un trajet supérieur à 2000 km sinon ça renvoie une erreur 

> Questions de réfléxion

● Pourquoi mettre la logique dans la base ?

Pour éviter les erreurs et être sûr que les règles sont toujours respectées, peu importe l’application qui utilise la base. Ça permet aussi d’automatiser certaines actions directement côté base.

● Quel impact sur la performance ?

Ça peut ralentir les INSERT / UPDATE / DELETE, car la base doit exécuter du code en plus à chaque fois. Plus il y a de triggers, plus ça peut devenir lourd.

● Quand éviter les triggers ?

Quand la logique est trop complexe ou quand elle peut être faite plus simplement dans le backend. Aussi quand ça rend la base difficile à comprendre ou à maintenir.

# Questions TP5 :

> Pourquoi imbriquer ici ?

- les données sont fortement liées
- les camions et leurs entretiens sont souvent lu ensemble
- cela évite les jointures SQL
- la lecture est plus rapide

> Quand utiliser MongoDB plutôt que PostgreSQL ?

- les données changent souvent 
- les structures sont flexibles
- les données sont imbriquées
- on veut éviter les jointures
- les performances de lecture sont importantes

> Quand faut-il imbriquer ?

- les données sont fortement liées
- elles sont souvent consultées ensemble
- cela évite des jointures
- la taille du document reste raisonnable

> Quels sont les risques d’un document trop gros ?

- ralentissement des performances
- documents plus lourds à charger
- limite MongoDB de 16 MB
- mises à jour plus coûteuses
- duplication excessive des données

> Requêtes TP5 : 

# Partie 1 : 

insertOne :

```
{
  "nom": "GreenLogistics",
  "ville": "Lille",
  "capacite_totale": 120,
  "camions": [
    {
      "immatriculation": "FR-001-EC",
      "energie": "diesel",
      "options": ["GPS", "Frigo"]
    }
  ]
}
```

insertMany :

```
[
  {
    "nom": "EcoMove",
    "ville": "Marseille",
    "capacite_totale": 200,
    "camions": [
      {
        "immatriculation": "FR-002-EC",
        "energie": "electrique",
        "options": ["GPS"]
      }
    ]
  },
  {
    "nom": "HydroTrans",
    "ville": "Nantes",
    "camions": [
      {
        "immatriculation": "FR-003-EC",
        "energie": "hydrogene",
        "options": ["GPS", "Hayon"]
      }
    ]
  }
]
```

# Partie 2 : 

Trouver les capacités > 150 :

```
{ "capacite_totale": { "$gt": 150 } }
```

Trouver les camions électriques ou hydrogène :

```
{
  "camions.energie": {
    "$in": ["electrique", "hydrogene"]
  }
}
```

Trouver les camions avec option Frigo :

```
{
  "camions.options": "Frigo"
}
```

# Partie 3 :

Ajouter un $group :

```
{
  "_id": "$ville",
  "moyenne_capacite": {
    "$avg": "$capacite_totale"
  }
}
```

Ajouter $sort :

```
{
  "moyenne_capacite": -1
}
```

# Partie 4 :

Ajouter des entretiens :

```
"entretiens": [
  {
    "date": "2025-01-10",
    "type": "vidange",
    "cout": 300
  },
  {
    "date": "2025-03-15",
    "type": "freins",
    "cout": 700
  }
]
```

# Partie 5 :

Ajouter plusieurs entretiens :

```
{
  "date": "2025-05-01",
  "type": "pneus",
  "cout": 450
},
{
  "date": "2025-06-10",
  "type": "batterie",
  "cout": 900
}
```

# Questions TP7 :

> Requêtes :

```
SELECT COUNT(*) FROM trajets_perf;
```

![select_trajet](./trajets_select.png)

```
SELECT *
FROM trajets_perf
WHERE ville_depart = 'Paris';
```

![paris_trajet](./paris_select.png)

Avant Index
```
EXPLAIN ANALYZE
SELECT *
FROM trajets_perf
WHERE ville_depart = 'Paris';
```

![explain_analyse](./explain_analyse.png)

Après Index
```
EXPLAIN ANALYZE
SELECT *
FROM trajets_perf
WHERE ville_depart = 'Paris'
```

![explain_analyse](./explain_analyse_after.png)

Index Composite
```
EXPLAIN ANALYZE
SELECT *
FROM trajets_perf
WHERE ville_depart = 'Paris'
AND date_trajet > CURRENT_DATE - 30;
```

![explain_analyse_composite](./explain_analyse_composite.png)

> Pourquoi PostgreSQL lit-il toute la table ?

Parce qu’il n’y a pas d’index sur la colonne.
Donc il doit vérifier chaque ligne une par une.

> Quelles sont vos observations par rapport à Postgres ?

Avec l’index, la requête est plus rapide.
PostgreSQL lit moins de données.
Il utilise un Index Scan au lieu d’un Seq Scan.

> Est-ce plus rapide que l’index simple ?

Oui, souvent c’est plus rapide.
Parce que l’index composite filtre mieux les données.

> Pourquoi les index accélèrent-ils les lectures ?

Parce qu’ils évitent de lire toute la table.
PostgreSQL trouve directement les bonnes lignes.

> Pourquoi ralentissent-ils parfois les INSERT ?

Parce que l’index doit aussi être mis à jour à chaque insertion.

> Pourquoi ne pas indexer toutes les colonnes ?

Parce que ça prend de la place.
Et ça ralentit les écritures (INSERT, UPDATE).

# Question TP7bis :

# Partie 1 :
>1/ Index Composite
![tp7bispart1](./tp7bispart1.png)

> PostgreSQL utilise-t-il l’index ?

Oui, PostgreSQL utilise l’index composite si la requête filtre sur ville_depart et date_trajet.

> Le gain est-il visible ?

Oui, le gain est visible, réduction du temps d’exécution et moins de scans séquentiels.

>2/ Index Partiel
![tp7bispart2](./tp7bispart1_2.png)

> Pourquoi c’est puissant ?

Parce qu’il indexe uniquement les données utiles aux requêtes fréquentes, ce qui le rend plus petit, plus rapide et moins gourmand en mémoire.

> Pourquoi la requête distance_km < 100 n’utilisera pas forcément l’index ?

Car l’index partiel a été créé uniquement pour les lignes où distance_km > 500.

>3/ Sans Index
![tp7bispart3](./tp7bispart1_3.png)

Sans index Temps : 0.016s     
Index simple Temps : 1.140s
Index partiel Temps : 4.591

# Partie 2 :

> Que remarquez-vous après la suppression de 50 % des données ?

La taille de la table ne diminue pas immédiatement, car l’espace libéré n’est pas automatiquement récupéré.

> Pourquoi le bloat est-il dangereux ?
Parce qu’il augmente la taille des tables et des index, ce qui ralentit les requêtes et gaspille de l’espace disque.

# Partie 4 :

> Générer requêtes lentes

```
SELECT *
FROM trajets_perf
ORDER BY random()
```

![tp7bispart4](./tp7bispart4.png)

> Lire les logs

```
docker logs ecotrack-postgres
```

![logs](./logs.png)

# Questions de réflexion :

> Pourquoi tous les index ne sont-ils pas utiles ?

Parce que si tu mets un index partout, ça sert à rien. Il y en a qui ne seront jamais utilisés par les requêtes, donc ça prend juste de la place pour rien et ça peut même ralentir la base.

> Pourquoi les index ralentissent les INSERT ?

Parce qu’à chaque fois que tu ajoutes une ligne, la base doit aussi mettre à jour tous les index liés. Donc plus tu en as, plus chaque insertion prend du temps.

> Pourquoi MongoDB a besoin d’index spécifiques ?

Parce que les données ne sont pas comme en SQL : c’est du JSON avec des tableaux, du texte, des coordonnées… Du coup il faut des index adaptés à chaque type pour que les recherches soient rapides.

> Pourquoi le bloat est dangereux ?

Parce qu’avec les suppressions et les modifications, la base garde de l’espace “vide” inutile. Du coup ça grossit, ça prend plus de place et ça finit par ralentir les requêtes.