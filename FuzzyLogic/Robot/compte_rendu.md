# Compte Rendu : Contrôleur Flou pour la Navigation de Robot

## 1. Architecture du Système

Le contrôleur flou permet de diriger le robot vers un objectif tout en évitant les obstacles en modulant deux sorties :
- **Slin** : Vitesse linéaire (Avancer).
- **Sang** : Vitesse angulaire (Tourner).

### Schéma des Entrées / Sorties

```mermaid
graph LR
    subgraph Capteurs / Entrées
        D_Goal[DistGoal]
        Dir_Goal[DirecGoal]
        O_Front[ObstFront]
        O_Left[ObstLeft]
        O_Right[ObstRight]
    end

    subgraph Moteurs / Sorties
        V_Lin[Slin <br/> Vitesse Linéaire]
        V_Ang[Sang <br/> Vitesse Angulaire]
    end

    Fuzzy((Contrôleur <br/> Logique Floue))

    D_Goal --> Fuzzy
    Dir_Goal --> Fuzzy
    O_Front --> Fuzzy
    O_Left --> Fuzzy
    O_Right --> Fuzzy
    
    Fuzzy --> V_Lin
    Fuzzy --> V_Ang
```

---

## 2. Variables Linguistiques et Ensembles Flous

Les variables d'environnement sont traduites en concepts qualitatifs (ensembles flous) :

*   **Distance au But (`DistGoal`)** : `Near` (Proche), `Far` (Loin).
*   **Distance des Obstacles (`Obst...`)** : `NearObst` (< 250), `FarObst` (> 120).
*   **Direction du But (`DirecGoal`)** : `onRight` (A droite), `inFront` (En face), `onLeft` (A gauche).
*   **Vitesse Linéaire (`Slin`)** : `Slow` (Lent), `Fast` (Rapide).
*   **Vitesse Angulaire (`Sang`)** : `turnRight` (Droite), `straightOn` (Tout droit), `turnLeft` (Gauche).

### Représentation des Fonctions d'Appartenance (Exemple d'Orientation)

```mermaid
graph TD
    A[Capteur d'Orientation: -90° à 90°] --> B{Fuzzification}
    B -->|-90° à 0°| C[onRight]
    B -->|-90° à 90°| D[inFront central]
    B -->|0° à 90°| E[onLeft]
```

---

## 3. Logique de Contrôle et Règles

Le comportement du robot est divisé en deux états majeurs qui sont pondérés en permanence par la logique floue : **Navigation vers le but** et **Évitement d'obstacles**.

```mermaid
stateDiagram-v2
    [*] --> Evaluation_Environnement
    Evaluation_Environnement --> Navigation : ObstFront = FarObst
    Evaluation_Environnement --> Evitement : ObstFront = NearObst

    state Navigation {
        Avancer_Rapide : Slin = Fast (si but lointain)
        Avancer_Lent : Slin = Slow (si but proche)
        Alignement : S'orienter vers DirGoal (R1, R2, R3)
    }

    state Evitement {
        Ralentissement : Slin = Slow (R8)
        Tourner_Droite : Si Mur à Gauche (R4)
        Tourner_Gauche : Si Mur à Droite (R5)
    }
```

### Synthèse des Règles (Base de Connaissance)

| Type | Règle N° | Condition (Si...) | Action (...Alors) |
| :--- | :--- | :--- | :--- |
| **Poursuite But** | R1 | But en face & Voie libre | Tout droit |
| | R2 | But à gauche & Voie gauche libre | Tourner à gauche |
| | R3 | But à droite & Voie droite libre | Tourner à droite |
| **Évitement** | R4 | Obstacle Face & Obstacle Gauche | Tourner à droite |
| | R5 | Obstacle Face & Obstacle Droite | Tourner à gauche |
| **Contrôle Vitesse**| R6 | But éloigné & Voie libre | Vitesse Rapide (`Fast`) |
| | R7 | But proche | Vitesse Lente (`Slow`) |
| | R8 | Obstacle Face proche | Vitesse Lente (`Slow`) |

---

## 4. Analyse des Limites et Conflits (Conflit R4 / R5)

Le jeu de règles actuel possède un point de défaillance structurel en cas d'encerclement symétrique (comme un angle droit fermé ou un couloir étroit où le robot arrive de face).

```mermaid
flowchart TD
    Situation[Robot face à un mur avec obstacles des deux côtés]
    Situation --> Cond1[ObstFront = NearObst]
    Situation --> Cond2[ObstLeft = NearObst]
    Situation --> Cond3[ObstRight = NearObst]
    
    Cond1 & Cond2 --> R4{Règle 4 déclenchée}
    Cond1 & Cond3 --> R5{Règle 5 déclenchée}
    
    R4 -->|Sang = turnRight| Conflit((Dé-fuzzification))
    R5 -->|Sang = turnLeft| Conflit
    
    Conflit -->|Annulation des forces opposées| Result[Sang = 0 <br> Le robot va tout droit et s'arrête / percute l'obstacle]
```

**Conclusion de l'analyse :**
Lorsque les règles `R4` et `R5` s'activent simultanément à des degrés d'appartenance similaires, le barycentre de la vitesse angulaire tombe à zéro. Pour corriger cela, il faut introduire une règle d'asymétrie stricte en cas de danger imminent (ex: "S'il y a un mur en face, forcer un virage à gauche sauf si un mur est spécifiquement à gauche").