# Créer une table de Mesures

1. Depuis le portail ajouter une table : _Home > Enter Data_
2. Ajouter une colonne avec une ligne (sans valeur particulière), Appeler la table _Calculs_, et valider
3. Dans cette table, créez votre première mesure : Clic droit sur la table _Calculs_ > Nouvelle mesure > Dans la barre de formule > _Today = TODAY()_ > Entrée
4. Supprimer la colonne de la table Calculs. La table devrait apparaître en-tête de la liste des tables, et son logo devrait changer pour une calculatrice.

# Créer ses premières mesures 

1. Nombre de vélos disponibles
2. Nombre de vélos mécaniques disponibles
3. Moyenne du nombre de vélos par station
4. Nombre de stations
5. Booléen qui indique s'il y a des vélibs mécaniques (True) ou non (False) dans la borne.
6. Nombre de stations vides
7. Nombre de stations dont la capacité est supérieure à 35
8. Nombre de stations pleines
9. Pourcentage de stations pleines
  - Pour effectuer une division, favoriser l'usage de la fonction DIVIDE( A, B), plutôt que "/" qui ne gère pas la division par zéro et qui peut générer des latences (CallBack)
  - Pour afficher le taux sous forme de %, cliquer sur la mesure et dans l'onglet Measure Tools, cliquer sur "%"
  - ![image](https://github.com/user-attachments/assets/c2d915fb-0721-4446-86c3-42540c9fe64c)
10. Taux de remplissage des stations (nombre de vélos disponibles comparé à la capacité)
11. Pourcentage de capacité de la station comparé à la capacité totale
12. Rang de station en terme de capacité

# Réponses
1. ```Nombre de vélos disponibles = SUM(Etats[Total Available Bikes])```
2. ```Nombre de vélos mécaniques disponibles = SUM(Etats[Mechanical Bikes Available])```
3. ```Moyenne du nombre de vélos par stations = AVERAGE(Etats[Total Available Bikes])```
4. ```Nombre de stations = DISTINCTCOUNT(Etats[Station Code])```
5. ```Vélos Mécaniques disponibles = IF(SUM([Nombre de vélos mécaniques disponibles]) > 0, true, false)```
6. ```
    Nombre de stations vides = CALCULATE(
       [Nombre de stations]
       , 'Etats'[Total Available Bikes] = 0
   )
  ---
7. ```
    Nombre de Stations Capacité Elevée = CALCULATE(
        [Nombre de stations]
        , Stations[Capacity] > 35
    )
  ---
8. ```
    Nombre de stations pleines = CALCULATE(
        [Nombre de stations]
        , 'Etats'[Total Available Docks] = 0 
        , 'Etats'[Total Available Bikes] <> 0 
    )
  ---
9. ```
    Pourcentage de stations pleines = DIVIDE(
        [Nombre de stations pleines]
        , [Nombre de stations]
    )
  ---
10. ```
    Taux de remplissage des stations = DIVIDE(
        [Nombre de vélos disponibles]
        , MAX(Stations[Capacity])
    )
   ---
11. ```
    Pourcentage vs Total = DIVIDE(
        SUM(Stations[Capacity])
        , CALCULATE( SUM(Stations[Capacity]), ALL(Stations) )
    )
  ---
12. ```
    Capacité Station = MAX(Stations[Capacity])
    Rang Capacité Station = RANKX( ALL( Stations ), [Capacite Station] ,,,DENSE)
   ---

# Scenario What-If

1. Créer une table avec une liste de valeurs de 0 à 10
2. Cette table sans relation est dite "déconnectée". 
3. Créer une mesure ScenarioDisponibilite qui récupère la valeur selectionnée par l'utilisateur
    1. Utiliser la fonction MIN() ou MAX() ou SELECTEDVALUE()
4. Créer une mesure qui soustrait au nombre de vélo disponible la mesure ScenarioDisponibilite
5. Créer une visel filtre avec comme choix la colonne de la table déconnectée 
6. Dans une matrice, afficher le nom des stations, le nombre de vélo disponible et cette nouvelle mesure testée à l'étape 4. Jouer avec le filtre pour faire varier la mesure.
