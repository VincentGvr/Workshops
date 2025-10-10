# Introduction & Architecture Générale

L'objectif de ce pas à pas est de proposer un environnement de travail au sein de Microsoft Fabric avec les leviers de sécurité d'accès les plus restrictifs. Nous allons découvrir comment fournir l'accès aux ressources uniquement depuis le réseau interne d'une entreprise ou d'une organisation, pour être capable de développer un Notebook Spark au sein de Microsoft Fabric, qui aurait besoin d'atteindre des données contenues dans un Data Lake au sein d'Azure. L'échange entre l'utilisateur et la plateforme se fait ne manière privée, et l'échange entre Microsoft Azure et Microsoft Fabric se fait lui aussi de manière privée.

Nous allons pas à pas déployer un environnement qui respecte les contraintes suivantes : 
  - **Utilisation du Private Link** : Les échanges avec Power BI et/ou Microsoft Fabric sont fait via le réseau Azure, et non via Internet. Cela permet de contrôler le point d'entrée des utilisateur, ainsi que les intéreactions avec les données qui transitent vers et en dehors de Microsoft Fabric. 
  - **Désactivation de l'accès depuis Internet public** : Non seulement l'accès se fait via les services interne Azure, mais en plus l'accès au portail ne peut plus se faire via l'IP publique du service.
  - **Désactivation des accès public aux ressources dans Azure** : En désactivant les accès au Key Vault (container des clés d'identification) ou à ADLS Gen2 (container des données) via un réseau public, on interdit aussi les accès depuis n'importe quel endroit
  - **Utilisation de private endpoints pour les services Azure** : On définit un point d'entrée approuvé pour accéder aux ressources Azure, ce point d'entrée est donc strictement réservé à un service ou un usage.
  - **Consommations des services Azure via Microsoft Fabric de manière privée** : Une fois l'accès au portail restreint, les échanges entre le service et les ressources maîtrisé, les utilisateurs et développeurs évolueront dans un environnement cloisoné.  

**Notes & Limitations :**
  - Managed Private Network : Lorsqu'un Administrateur active le private Link, le lancement d'un notebook Spark générera automatiquement l'usage de Managed Virtual Network pour instancier les ressources de calcul Spark. Ce cluster n'est pas accessible depuis le réseau public. L'instanciation des ressources au sein de ce Managed Virtual Network est plus long que lorsqu'on l'utilise de manière publique, le démarrage d'une session passe de 30 secondes environ à 4 minutes.
     
  - Starter Pools : Il est possible de lancer un Notebook Spark sans se soucier de la configuration des ressources de calculs utilisées. Dans ce cas, ce pool de ressource appelé _Starter Pool_ sera instancié et possède une taille standard mais non optimale pour exécuter les calculs. Lors de la privatisation des ressources, ce Starter Pool n'est plus disponible, et il est donc nécessaire d'en créer un avec une taille définie directement au sein du Workspace par exemple pour être capable d'exécuter notre Notebook.  

Une fois ces différents pré-requis validés, nous allons voir comment nous pouvons utiliser Microsoft Fabric pour transformer nos données et les faire évoluer dans notre environnement. 

**Architecture générale :** 

<img src="https://github.com/user-attachments/assets/ab850c9f-8417-478f-bf26-a90a1baad66a">

L'usage d'une VM dans Azure n'est pas nécessaire. 

# Pré requis

Pour ce pas à pas, nous aurons besoin : 
  - **Les rôles** : 
    - Au sein d'Azure, nous avons besoin des droits de création dans une souscription de test ou d'un groupe de ressources 
    - Au sein de Microsoft Fabric, des droits d'administrateurs sur l'environnement. 
  - **Les ressources** : 
    - D'un environnement **Azure**, avec la possibilité de créer :
      - Une capacité Microsoft Fabric 
      - Un Virtual Network ainsi qu'un Bastion
      - Une Virtual Machine 
      - Un private Link Services
      - Un Key Vault
      - Un Azure Data Lake Storage Gen 2
    - D'un environnement **Microsoft Fabric** avec le rôle Administrateur :
      - Pour activer/desactiver les features au sein du portail
      - La possibilité de créer un workspace, qui va contenir un Notebook

# Pas à pas : Partie 1 : Mettre en place l'accès au portail de manière privée. 

Il existe une documentation Microsoft permettant de créer un Private Link pour Power BI et Microsoft Fabric. Je reprends ces éléments en détail pour le début du lab : https://learn.microsoft.com/en-us/fabric/security/security-private-links-use 

## 1. Activation des features dans le portail Power BI :

_Dans cette partie, nous allons activer la feature Private Link qui nous permet d'accéder aux ressources via le Backbone Azure._

Dans le portail d'administration Power BI, nous allons activer la feature permettant d'utiliser les Private Links : Accéder au portail d'administartion, rechercher les paramètres réseaux avancés, et activer la feature : 

<img src="https://github.com/user-attachments/assets/638cbb9f-7586-41c0-a6fb-2809fddb171d" width="300">
<img src="https://github.com/user-attachments/assets/4fcba74a-4e9e-4aff-b505-18f8c48cb04f" width="400">

Nous allons aussi récolter l'ID de notre tenant Microsoft Fabric : Cliquer sur "?" dans l'en-tête > About Power BI > et copier le GUID après _ctid=_. **Nous allons noter cet ID pour plus tard.** 

<img src="https://github.com/user-attachments/assets/eda6e8f3-7528-4452-acbd-5e5cd6cccd8c" width="300">
<img src="https://github.com/user-attachments/assets/02293db2-0d6b-4358-afbc-13b49ba1d705" width="300">

## 2. Création du groupe de ressources dans Azure :

_Dans cette partie, nous allons créer le container de nos ressources ainsi que le service private Link côté Azure._

Depuis le portail Azure : ```https://portal.azure.com``` : 
Nous allons créer un groupe de ressources pour contenir l'ensemble de nos services Azure. Dans le volet latéral > créer une ressource > rechercher ```resource group``` (il est préférable de ne filtrer que les services Azure) > Créer > Choisir la souscription retenue, et renseigner le nom du groupe de ressources. **Noter ce nom de resource group pour plus tard.**

<img src="https://github.com/user-attachments/assets/3a2f0f70-98b0-4bee-889b-f590af92e954" width="500">
<img src="https://github.com/user-attachments/assets/ef9ced03-cb39-47ce-9876-d7cadd99b1b5" width="500">

## 3. Création du private Link Service pour Power BI (même pour Microsoft Fabric, cela s'appelle Power BI) :

Une fois le resource group créé > Dans le centre de notifcation, accéder au groupe de ressources > Choisir Créer dans l'en-tête > Rechercher ```Template deployment``` > Créer. Ce template nous permet de créer une ressource via du Code sans passer par les différents menus. 

<img src="https://github.com/user-attachments/assets/796576ae-db3a-4b57-b5df-62234418f900" width="300">
<img src="https://github.com/user-attachments/assets/9c076b8e-e2d5-4eb5-b71f-43469ba176d0" width="500">
<img src="https://github.com/user-attachments/assets/3dd14a61-365e-4f5b-85d9-c15de5bec98b" width="500">

Une fois le bloc de code apparu à l'écran : copier le code ci-dessous, en remplaçant : 
- La valeur de l'attribut name pour le nom du private link service
- La valeur de l'attribut tenantId pour l'ID de l'environnement récupéré sur Power BI.
```{
  "$schema": "http://schema.management.azure.com/schemas/2015-01-01/deploymentTemplate.json#",
  "contentVersion": "1.0.0.0",
  "parameters": {},
  "resources": [
      {
          "type":"Microsoft.PowerBI/privateLinkServicesForPowerBI",
          "apiVersion": "2020-06-01",
          "name" : "<resource-name>",
          "location": "global",
          "properties" : 
          {
               "tenantId": "<tenant-object-id>"
          }
      }
  ]
}
```

Une fois renseigné, passer les onglets de création en renseignant les informations liées au resource group, puis Valider la création : 

<img src="https://github.com/user-attachments/assets/4d72d13c-183e-436c-8328-205af0a96310" width="500">
<img src="https://github.com/user-attachments/assets/550dd9a7-6823-45a1-bb5a-98af24b5096d" width="400">
<img src="https://github.com/user-attachments/assets/a4777ba1-5507-4316-b382-6f416a26fb23" width="500">

## 4. Création des services réseau

L'architecture réseau ne respecte pas aujourd'hui la topologie Hub N Spoke. Pour être plus précis, il serait préférable de favoriser cette architecture en scindant les éléments au sein de différents réseaux. 

Aussi, dans mon cas je n'ai pas de réseau d'Entreprise à relier à notren infrastructure. Je vais donc créer une machine virtuelle directement dans le réseau de test. Dans un monde idéal, il serait préférable d'utiliser une machine locale, reliée au réseau d'enterprise, lui même relié au service Azure via une Express Route.

_Dans cette partie, nous allons créer les ressources réseaux nécessaire au regroupement de nos services._

**Création du VNET**

Depuis le groupe de ressource : Créer > rechercher _Virtual Network_ > Créer une ressource. 

Dans l'onglet **Basics**, choisir le groupe de ressources créé précédemment, définir le nom et la région du réseau virtuel.

<img src="https://github.com/user-attachments/assets/2355d410-83c6-4711-9c69-5134efa75b6a" width="500">

Dans l'onglet **Security**, activer la feature ```Azure Bastion```, qui nous permettra d'accéder à la machine virtuelle depuis un onglet web, évitant ainsi d'ouvrir un port nécessaire au contrôle à distance. 

<img src="https://github.com/user-attachments/assets/e7170bdb-dd44-40ed-b6fe-3738a641e106" width="500">

Dans l'onglet **IP Adresses**, définir les plages réseaux nécessaires à notre infrastructure. Dans mon cas, le nombre d'adresse est très petit, il est possible de limiter le nombre d'adresses réservées à notre cas de test. 

<img src="https://github.com/user-attachments/assets/48b68d16-2f6b-4607-908d-bf8c6e09c2f8" width="500">

**Création du Private Endpoint pour Power BI & Microsoft Fabric** 

C'est le service qui va nous permettre de communiquer avec Microsoft Fabric de manière privée. 

Depuis le groupe de ressource : Créer > rechercher _Private Endpoint_ > Créer une ressource. 

Dans l'onglet **Basics**, renseigner le de resource group ainsi que le nom du private endpoint :  

<img src="https://github.com/user-attachments/assets/9ed19018-0238-40d5-9ce9-e89f62d761fe" width="500">

Dans l'onglet Resource, choisir Microsoft.PowerBI/privateLinkServicesForPowerBI pour le type de resources, puis dans le menu déroulant, choisir le service créé à l'aidu du template ARM plus tôt dans ce lab. Pour la sous ressource, choisir _tenant_. 

<img src="https://github.com/user-attachments/assets/63ee729e-056e-47b3-8043-3bcac9c7780d" width="500">

Dans l'onglet **Virtual Network**, choisir le VNET créé plus tôt, et laisser le sous-réseau par défaut : 

<img src="https://github.com/user-attachments/assets/ba171ea8-73a1-4a3f-8754-4e12e684ca1a" width="500">

Dans l'onglet **DNS**, définir les 3 DNS Private Zone pour notre service (ils doivent être renseignés par défaut) : 
  - _(New)privatelink.analysis.windows.net_
  - _(New)privatelink.pbidedicated.windows.net_
  - _(New)privatelink.prod.powerquery.microsoft.com_

<img src="https://github.com/user-attachments/assets/569ead0d-1dc0-4604-820b-39b043f03384" width="500">

## 5. Creation de la machine virtuelle

Une fois le réseau virtuel créé, nous allons créer une machine virtuelle permettant de simuler un poste utilisateur intégré à un réseau privé. C'est notre point d'entrée pour accéder à Microsoft Fabric. 

Depuis le groupe de ressource : Créer > rechercher _Virtual Machine_ > Créer une ressource. 

Dans l'onglet **Basics**, renseigner le groupe de ressources, le nom de la machine virtuelle. 
Pour notre test, Choisir ```No Infrastrcuture redundancy required``` pour le niveau de disponibilité, > ```Standard``` pour le Security Type. Pour la taille de la VM, choisri quelque chose de simple : ```Standard_D2s_V6``` par exemple, avec une image ```Windows Server 2022```. 

<img src="https://github.com/user-attachments/assets/d04620bb-1cce-4764-8185-85198e3439f1" width="500">

Plus bas, renseigner le compte Administrateur ainsi que le mot de passe pour s'authentifier. 
Désactiver l'ensemble des ports entrants en selectionnant "None". 

<img src="https://github.com/user-attachments/assets/1a7a8bbe-6c90-47eb-9f5d-00fd91b7a506" width="500">

Passer l'onglet **Disks**, et dans l'onglet **Networking**, renseigner le VNET créé auparavant. 

<img src="https://github.com/user-attachments/assets/954cd52f-28b7-457e-9005-839e20162fba" width="500">

Dans l'onglet **Advanced**, préférez l'activation de l'Auto Shutdown pour éviter de consommer des ressources inutilement :

<img src="https://github.com/user-attachments/assets/8a30b3af-460f-4ac1-bb69-63d73cf4c345" width="500">

**Connect via Bastion** 

Une fois la machine créée, nous allons nous y connecter via Azure Bastion : Retrouver la ressource créée dans le portail, dans les menus latéraux se rendre dans Connect > Bastion > renseigner le username & password > Cliquer sur connect. 

<img src="https://github.com/user-attachments/assets/5535616a-3938-4f2d-9f67-d863170f839d" width="500">

Une fois connecté à la VM, démarrer une invite de commande : Menu Windows > tapper _cmd_. 

Retourner sur le portail Azure, trouver la ressource _private endpoint_ créée précédement, et se rendre dans Settings > DNS Configuration : copier la valeur dans la colonne FQDN :

<img src="https://github.com/user-attachments/assets/a68193ae-8274-4697-9874-5c8957cbcd9a" width="800">

Depuis l'onglet Bastion pour la VM, nous allons tester le FQDN du DNS :  
Remplacer la valeur du FQDN par celle récupérée sur le portail : ```nslookup e171a59a21664df89256ed18abb8ee91-api.analysis.windows.net```

<img src="https://github.com/user-attachments/assets/528c56c5-54a0-43e6-8998-eb5dad5cefb3" width="500">

L'idée est d'observer que l'adresse récupérée est bien une adresse ausein de notre réseau (en 10.X.X.X) et non une IP publique. 

Une fois qu'il est possible d'accéder au portail via une IP privée, nous allons désactiver l'accès au portail via Internet. Depuis la VM, se connecter au portail Power BI, et se rendre à nouveau dans les paramètres du tenant > Activer le paramètre Block Public Internet Access :  

<img src="https://github.com/user-attachments/assets/aef0c1ea-e111-4771-b57a-9420f2bac3ec" width="500">

**Ce paramètre ne peut pas être désactivé depuis une IP publique par sécurité.** 

<img src="https://github.com/user-attachments/assets/8c3ae0d1-2626-4f4f-be85-97356105645d" width="800">

S'il on essaye d'accéder au portail directement depuis un navigateur, en dehors de la VM : 

<img src="https://github.com/user-attachments/assets/aad07028-6c18-41ad-80cd-cfbe7529ab2a" width="600">

# Partie 2 : Accéder aux services et données dans Azure depuis Microsoft Fabric 

# 1. Création d'un compte de stockage 

Pour continuer notre cas d'usage nous avons besoin de stocker des données au sein d'Azure Data Lake Storage Gen 2. 
Depuis le portail, Créer une ressource > Rechercher _Storage Account_ > Créer. 

Dans l'onglet **Basics**, renseigner le resource group, la region ainsi que le nom de la resource (sans majuscule ni caractères spéciaux). 

<img src="https://github.com/user-attachments/assets/967bc46a-c23c-48f0-a285-430ec8f39bc8" width="600">

Dans l'onglet **Advanced**, activer _Enable Storage Account Key Access_ pour utiliser la clé du compte plus tard, puis Enable Hierarchical Namespace qui permet de transformer le compte de stockage classique en data lake : 

<img src="https://github.com/user-attachments/assets/99c078b6-8009-482a-a211-cf278d342f99" width="500">

Dans l'onglet networking, désactiver l'accès publique au compte de stockage. 

<img src="https://github.com/user-attachments/assets/63a69146-5ef2-434c-aae4-9ae169d8364f" width="500">

Terminer la création du compte de stockage. Une fois créé, depuis les menus latéraux, Data Storage > Containers > et créer un container. Pour aller plus loin dans la démo, il est possible d'alimenter ce compte de stockage avec des données. (Pour cela, il faut soit l'alimenter depuis l'intérieur du réseau, soit le charger manuellement en ré-activant la partie réseau publique temporairement). 

<img src="https://github.com/user-attachments/assets/7abcac83-4384-4aa9-8764-102ba547e8cf" width="500">

# 2. Creation du workspace au sein de Microsoft Fabric : 

Depuis le portail Fabric, Créer un workspace, l'attribuer à une capacité Fabric depuis le menu **Advanced**

<img src="https://github.com/user-attachments/assets/c24cd867-8b60-432b-a6d6-98e0f1979bf9" width="500">

# 3. Création du Private Endpoint vers ADLS Gen 2 : 

Une fois le workspace créé, naviguer dans le workspace et cliquer sur Settings. Dans le menu latéral, choisir Network Security > puis dans Managed Private Endpoint > Créér. Nous allons créer un private endpoint depuis notre workspace vers l'ADLS Gen 2 créé auparavant. 

<img src="https://github.com/user-attachments/assets/6db69e94-632f-4422-8d5f-ef9358f10dd0" width="500">

Définir le nom du private endpoint, puis renseigner les éléments du compte de stockage en remplaçant les valeurs : 

```/subscriptions/{subscription-id}/resourceGroups/{resource-group-name}/providers/Microsoft.Storage/storageAccounts/{storage-account-name}```

Par exemple, dans notre cas : (ces valeurs peuvent être retrouvées dans

```/subscriptions/ecaa47e5-79d1-47e4-ac93-80b1484c3dbf/resourceGroups/rg-private-fabric-all-in/providers/Microsoft.Storage/storageAccounts/safabricprivate```

<img src="https://github.com/user-attachments/assets/afdf748a-13a8-4bf9-8567-6e3173109ce4" width="500">

Une fois créé, on le retrouve dans la liste des endpoints, au statut _Activating_. 

<img src="https://github.com/user-attachments/assets/4d1f3787-4515-49b0-ab56-367d308bc397" width="500">
<img src="https://github.com/user-attachments/assets/ced69224-96ba-422a-a683-e99af82ed63a" width="500">

Dans Azure, retrouver le compte de stockage auquel on veut se connecter > Networking > Onglet Private Endpoint Connections et Approuver le private endpoint 

<img src="https://github.com/user-attachments/assets/25616761-58eb-4374-b926-027510233a8c" width="800">

Une fois Activé, il est au statut _Approved_ et _Succeeded_ sur Fabric : 

<img src="https://github.com/user-attachments/assets/893f2e18-2156-4fd3-a90a-b30f7a54a546" width="500">

Attention, une fois le Private Endpoint déployé et utilisé, il sera impossible d'utiliser le Pool Spark par défaut : rien de grave, simplement il faut en créer un dédié. Pas de surcout additionnel ici, mais cela demande simplement plus de paramétrage. Dans l'absolu, c'est une bonne chose, il faut adapter son pool spark à ses besoins et ne pas utiliser un pool par défaut. 

<img src="https://github.com/user-attachments/assets/c6557c00-72a7-4118-887b-118fc68f1034" width="600">

Pour le créer, ans **Worksapce Settings**, déployer le menu **Data Engineering**, créer et définir le pool Spark par défaut :  

<img src="https://github.com/user-attachments/assets/3c4b5b94-e748-4231-8996-836927eaec84" width="400">
<img src="https://github.com/user-attachments/assets/97cca32a-8cb0-40de-bf04-d41621b245db" width="400">

# Workspace Identities

Une fois le lien réalisé entre Azure et notre compte de stockage, nous allons définir la méthode permettant de se connecter à notre Key Vault. Pour y accéder, il est possible d'utiliser l'identité managée du workspace Fabric : ce n'est pas l'utilisateur qui développe le notebook qui accède au Key Vault mais bien le workspace lui même.  

Pour y arriver, toujours dans **Worksapce Settings**, déployer le menu **Workspace Identity** et cliquer sur **+ Workspace Identity** 

<img src="https://github.com/user-attachments/assets/c1359d7e-b59a-4308-9df3-38780dcb11d9" width="400">
<img src="https://github.com/user-attachments/assets/b765fe16-f328-4934-9a71-de26299bd597" width="300">

# Key Vault

Création du Key Vault dans Azure et désactivation de l'accès public : 

<img src="https://github.com/user-attachments/assets/d6136e47-4c03-4fa3-a011-9fadad19517d" width="300">
<img src="https://github.com/user-attachments/assets/efa8ce05-e461-4602-a57d-62597a8baa72" width="500">
<img src="https://github.com/user-attachments/assets/ada19118-87ae-45dc-a7db-e6ffac0ce9dc" width="300">


Create key 
<img src="https://github.com/user-attachments/assets/a509871c-1e50-4a61-bd40-50d3f7cd1584" width="300">

Toto 

<img src="https://github.com/user-attachments/assets/cf7cc421-399c-40f3-80b7-398a07f8c848" width="300">

<img src="https://github.com/user-attachments/assets/166f30b7-879c-413c-b71c-7c869b7da732" width="300">
<img src="https://github.com/user-attachments/assets/a2682b80-86b0-45fc-8fd6-ee1c3648467f" width="300">

Toto

<img src="https://github.com/user-attachments/assets/44141c0c-f1e7-488b-acb5-26806ee7fb69" width="300">
<img src="https://github.com/user-attachments/assets/d9fa770d-a26c-4dfb-8305-58a0d60f6777" width="300">

Toto 

<img src="https://github.com/user-attachments/assets/30ea44ae-4430-4a93-85dd-c173af572cd2" width="300">
<img src="https://github.com/user-attachments/assets/04da8163-6f38-4182-bbeb-39e3a3f5ca41" width="300">


Nous allons répéter l'opération pour créer un private endpoint pour notre Key Vault, qui va contenir les identifiants permettant d'accéder à nos données dans notre compte de Stockage. 


```/subscriptions/{subscription-id}/resourceGroups/{resource-group-name}/providers/Microsoft.KeyVault/vaults/{vault-name}```
```/subscriptions/ecaa47e5-79d1-47e4-ac93-80b1484c3dbf/resourceGroups/rg-private-fabric-all-in/providers/Microsoft.KeyVault/vaults/kv-private-gennaker```

<img src="https://github.com/user-attachments/assets/81b4726b-42e7-45cf-b26c-ee9ff2cfe0fd" width="300">
<img src="https://github.com/user-attachments/assets/0c252fd4-f053-490c-87c2-9a496f807e1f" width="300">

# Use Key Vault & Notebook 

<img src="https://github.com/user-attachments/assets/f3063673-09e1-4511-9a58-78be4d3c1385" width="300">
<img src="https://github.com/user-attachments/assets/595294b8-4783-41d1-aee8-e4faad48622d" width="300">
