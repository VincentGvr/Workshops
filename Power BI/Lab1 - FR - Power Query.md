Dans ce Lab nous allons importer les données des APIs Vélib. 

# Création des tables

Deux URLs seront utilisées pour accéder à ces données : 
1. https://velib-metropole-opendata.smovengo.cloud/opendata/Velib_Metropole/station_information.json
2. https://velib-metropole-opendata.smovengo.cloud/opendata/Velib_Metropole/station_status.json

Pour importer les données de ces APIs nous allons Récuperer les données via le Connecteur Web : 
1. _Get Data > More ... > Web > Connect_
2. Coller l'URL dans l'invite 
3. Répéter l'étape depuis Power Query pour obtenir les deux requêtes côte à côte 
4. Renommer les deux requêtes en utilisant la partie propriétés sur la droite OU en cliquant sur le nom de la Query et utilisant "F2", ou via _Click droit > Rename_ :
  - station_status > Etats
  - station_information > Stations

# Optionnel : Décortiquer le fichier JSON

## Stations
1. Supprimer les étapes générées par le moteur Power Query sur la droite en retirant les étapes les unes après les autres 
2. Convertir le document en tableau exploitable 
3. _Cliquer sur la première colonne > Onglet Transformer > Transposer les lignes en colonnes_
4. Utiliser la première ligne comme en-tête
5. Exposer les données de la colonne Data grâce au bouton _Expand_ situé à côté du nom de la colonne. Ne pas conserver le préfixe de la colonne. 
6. Etendre la liste "stations" en utilisant le même bouton. En consultant le lien des stations (voir partie 1, 1) ), essayez de déterminer si l'exposition doit être faite sur les valeurs ou sur de nouvelles lignes. 
7. Après avoir réfléchis, choisissez sur de nouvelles lignes. 
8. Exposer le contenu des Lignes : 
  - Retirez le préfixe
  - Lister l'ensemble des colonnes disponibles

## Etats
1. Supprimer les étapes générées par le moteur Power Query sur la droite en retirant les étapes les unes après les autres 
2. Convertir le document en tableau exploitable 
3. _Cliquer sur la première colonne > Onglet Transformer > Transposer les lignes en colonnes_
4. Utiliser la première ligne comme en-tête
5. Exposer les données de la colonne Data grâce au bouton _Expand_ situé à côté du nom de la colonne. Ne pas conserver le préfixe de la colonne. 
6. Etendre la liste _stations_ en utilisant le même bouton. En consultant le lien des stations (voir partie 1, 1) ), essayez de déterminer si l'exposition doit être faite sur les valeurs ou sur de nouvelles lignes. 
7. Après avoir réfléchis, choisissez sur de nouvelles lignes. 
8. Exposer le contenu des Lignes : 
  - Retirez le préfixe
  - Lister l'ensemble des colonnes disponibles
9. Traitement de la colonne _num_bikes_available_types_
  - Version 1 :
    - Exposer la colonne via le bouton Expand sur de nouvelles lignes.
    - On se retrouve avec deux lignes pour chaque station. Nous allons les exposer, puis réduire le résultat à une ligne.
    - Exposer les _Records_ sur les colonnes mechanical et ebikes
    - Ajouter une première colonne conditionnelle appelée Available Bike Types : Si colonne mechanical est != _null_ > "mechanical", Sinon "electric"
    - Ajouter une seconde colonne conditionnelle appelée Available Quantity :  Si colonne mechanical est != _null_ > valeur de la colonne mechanical, sinon valeur de la colonne ebikes. Pour y arriver, changer le logo "ABC/123" en "Select A column"
    - Supprimer les colonnes _mechanical_ et _ebikes_
    - Cliquer sur la colonne _Available Bikes Type_ > Onglet Transform > Pivot > Dans la colonne _Values Column_, choisir "Available Quantity"

# Nettoyage des données :
## Stations
1. Nous allons nettoyer les données et préparer l'analyse dans notre rapport. N'hésitez pas à compléter ces étapes avec ce qui vous semble pertinent. 
2. Retirez les colonnes lastUpdatedOther et ttl, en utilisant _Ctrl + Click_ sur les deux colonnes, puis _click droit > Retirer les colonnes_. 
3. En consultant l'URL des stations_status (voir partie 1, 2) ), déterminez si l'usage des colonnes stationCode et station_Id est pertinent. Gardez la colonne la plus performante. 
4. Rennomez les colonnes (en déduisant la colonne que vous avez retiré à l'étape 3) : 
  - station_id > Station Id
  - stationCode > Station Code
  - name > Name
  - lat > Latitude
  - lon > Longitude
  - capacity > Capacity
5. Déduisez le typage des colonnes pertinent selon chaque information. Utilisez si bon vous semble la fonctionnalité _Detect data types_

## Etats
1. Nous allons nettoyer les données et préparer l'analyse dans notre rapport. N'hésitez pas à compléter ces étapes avec ce qui vous semble pertinent. 
2. Retirez les colonnnes lastUpdatedOther et ttl, en utilisant _Ctrl + Click_ sur les deux colonnes, puis _Click droit > Retirer les colonnes_. 
3. Gardez la colonne la plus performante en suivant votre logique du nettoyage des information des stations. 
4. Retirez les colonnes en double dans la table. 
5. Renommez les colonnes :
  - num_bikes_available ou numBikesAvailable > Available Bikes 
  - num_docks_available ou numDocksAvailable > Available Docks
  - is_Installed > Installed Flag
  - is_returning > Returning Flag 
  - is_renting > Renting Flag
  - last_reported > Last Reported Date

6. Utilisez la fonctionnalité de transformation/ajout de colonnes par la méthode de votre choix pour transformer les colonnes "... Flag" en colonnes _true/false_. 
7. Changez ensuite le type de la colonne en type true/false 
8. Trouvez un moyen de transformer la colonnes last_reported en valeur au format Datetime. 

## General 
Il est important de prendre du recul une fois les étapes appliquées. L'ordre des étapes est important : 
- Plus les étapes appliquées sont nombreuses, et plus l'ordre est important
- Les étapes les plus discriminantes doivent être faites en premier : les filtres, suppression de colonnes ...
- Les étapes qui se répètent peuvent être unifiées et mises ensemble (renommer toutes les colonnes d'un coup)

Le but de cet exercice est maintenant d'identifier les étapes qui se répètent, ou celles qui pouvaient être faites au plus tôt, et de les réorganiser. Il est possible de réorganiser les étapes appliquées en Cliquant/Glissant les étapes. Attention : si une étape dépend d'une autre, il faut l'anticiper. 
- Indice : Les colonnes lastUpdateOrder et ttl auraient pu être retirées dès le départ ... Chercher un moyen de les enlever le plus tôt possible. 

- Il est possible d'organiser les requêtes dans des dossiers : dans la partie _query > New Group > Glisser les requêtes dans les bons groupes_ 

# Paramétrage des requêtes 

Nous allons découvrir l'usage des paramètres pour simplifier l'évolution du tableau en remplaçant l'URL fixe par un paramètre.   
1. Créer un nouveau paramètre appelé _URL_ via l'une des deux méthodes suivantes : 
  - Click Droit sous _Query > New Parameter_ ... 
  - _Onglet Home > Manage Parameter > New Parameter_
2. Définir le nom sur _URL_
3. Définir le type sur _Text_ 
4. Remplir la valeur dans _Current Value_ avec ce qui est en commun dans les deux URL proposées plus haut. 
5. Pour chaque requête Stations et Etat, répéter les étapes suivantes : 
  - Cliquer sur la _Query_ choisie 
  - Dans les étapes appliquées, retrouver _Source_ 
  - Dans la barre de Formule, remplacer la partie fixe par le paramètre URL, et concaténer avec le reste via l'usage de _URL & "string"_ (remplacer string par la bonne valeur)
  - Cliquer sur la dernière étape et valider le développement. 

# Appliquer les développements dans la partie Power BI 

Une fois les développement effectués, appliquer les développements dans la partie Power Query : 
1. Dans l'onglet Home, cliquer sur _Appliquer les changements_. 

# Table Date

L'usage d'une table de date est importante dans un modèle de données Power BI est primordial pour arriver à utiliser l'intelligence Calendaire et les atouts de Power BI. Pour y arriver, il faut créer un référentiel de date. En général, cette requête est à mettre de côté et à réutiliser dans tous les modèles Power BI. Il existe plusieurs façons différentes de la générer, en voici une via Power Query. 
1. Créer une nouvelle requête vide : _Get Data > New Blank Query_
2. Dans l'onglet _Home > Advanced Editor_
3. Coller le contenu du code suivant et cliquer sur appliquer :
   ```
   let
     StartDate = #date(2020,1,1),
     EndDate = #date(2025,12,31),
     Culture = "fr-fr",
     DayCount = Duration.Days(Duration.From(EndDate - StartDate)) + 1,
     Source = List.Dates(StartDate,DayCount,#duration(1,0,0,0)),
     TableFromList = Table.FromList(Source, Splitter.SplitByNothing()),    
     ChangedType = Table.TransformColumnTypes(TableFromList,{{"Column1", type date}}),
     RenamedColumns = Table.RenameColumns(ChangedType,{{"Column1", "Date"}}),
     InsertYear = Table.AddColumn(RenamedColumns, "Year", each Date.Year([Date]), type number),
     InsertQuarterKey = Table.AddColumn(InsertYear, "QuarterKey", each (([Year] * 10) + Date.QuarterOfYear([Date]))),
     InsertQuarter = Table.AddColumn(InsertQuarterKey, "Quarter", each ("FY" & Number.ToText([Year]) & "-Q" & Number.ToText(Date.QuarterOfYear([Date]))), type text),
     InsertMonthKey = Table.AddColumn(InsertQuarter, "MonthKey", each (([Year] * 100) + Date.Month([Date]))),
     InsertMonth = Table.AddColumn(InsertMonthKey, "Month", each (Number.ToText([Year]) & " - " & Date.ToText([Date], "MMM", Culture)), type text),
     InsertDateKey = Table.AddColumn(InsertMonth, "DateKey", each (([Year] * 10000) + (Date.Month([Date]) * 100) + Date.Day([Date]))),
     InsertDay = Table.AddColumn(InsertDateKey, "Day", each Date.ToText([Date], "yyyy-MM-dd", Culture), type text),
     DateTable = Table.TransformColumnTypes(InsertDay, {{"DateKey", Int64.Type}, {"MonthKey", Int64.Type}, {"QuarterKey", Int64.Type}})
   in
     DateTable
   ```
5. En consultant les étapes et/ou le code, ajoutez les colonnes manquantes de la manière que vous préférez (Colonnes à partir d'exemple, Fonctions, Aide via AI/Stack Overflow ...)
 
