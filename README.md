# Configuration de Jenkins pour le .Net

## Bonnes pratiques générales

* Remplacer MsTests par NUnit

    * Possibilité de créer des extensions au framework
    * GUI pour lancer les tests en standalone si nécessaire
    * NUnit peut être téléchargé séparément de Visual Studio

* Découpler Nuget de Visual Studio
    * Evite de modifier les .csproj de chaque projet pour rajouter les targets / imports

## Pré-requis

* Visual Studio 2015
* SQL Server Express (uniquement si des tests d'intégration portent sur la base de données)

## Composants externes

* Git for Windows v2.11.0
    * Téléchargement : [https://git-for-windows.github.io/](https://git-for-windows.github.io/)
    * A installer dans : C:\Git

* Nuget v3.5.0
    * Téléchargement : [https://dist.nuget.org/win-x86-commandline/v3.5.0/NuGet.exe](https://dist.nuget.org/win-x86-commandline/v3.5.0/NuGet.exe)
    * A installer dans : C:\Nuget

* NUnit v3.5.0
    * Téléchargement : [https://github.com/nunit/nunit-console/releases/tag/3.5](https://github.com/nunit/nunit-console/releases/tag/3.5)
    * A installer dans : C:\Nunit
    * Utiliser la même version dans Visual Studio [avec Nunit 3 TestAdapter for Visual Studio 3.6.0](https://www.nuget.org/packages/NUnit3TestAdapter/)

* MsBuild
    * Utiliser la version fournie lors de l'installation de Visual Studio dans : C:\Program Files (x86)\MSBuild\14.0\Bin\

* OpenCover v4.6.519
    * Téléchargement : [https://github.com/OpenCover/opencover/releases](https://github.com/OpenCover/opencover/releases)
    * A dézipper dans C:\OpenCover

* ReportGenerator v2.5.1.0
    * Téléchargement : [https://github.com/danielpalme/ReportGenerator/releases](https://github.com/danielpalme/ReportGenerator/releases)
    * A dézipper dans C:\ReportGenerator

* OpenCoverToCoberturaConverter
    * Télécharger : [https://github.com/danielpalme/OpenCoverToCoberturaConverter](https://github.com/danielpalme/OpenCoverToCoberturaConverter)
    * Builder en Release
    * Et copier l'exe dans C:\CoberturaConverter

* CatLight
    * Télécharger : [https://catlight.io/](https://catlight.io/)
    * A installer sur les postes des développeurs pour monitorer l'état des jobs


## Plugins Jenkins à installer

* MSBuild Plugin => build les solutions / projets
* NUnit plugin => exécution des tests unitaires
* Task Scanner Plug-in => parcours le code et liste les TODO
* thinBackup => sauvegarde / restauration de config
* Cobertura Plugin => affichage du rapport
* SonarQube Plugin => analyse de code
* HTML Publisher plugin => 


## Variables d’environnement à définir

* Prise en compte des différents chemins d'accès des composants externes ci-dessus 
* Nom de l'environnement de développement
* Nom du profile de déploiement

## Configuration de la notification par email

* Saisir l'adresse email de l’administrateur (personne à qui sera envoyé les mails d’échecs de builds)

## Supprimer les balises targets / imports dans tous les .csproj

* Téléchargement : [https://github.com/owen2/AutomaticPackageRestoreMigrationScript](https://github.com/owen2/AutomaticPackageRestoreMigrationScript)
* Ouvrir PowerShell en tant qu'Admin puis taper la ligne ci-dessous et répondre Oui ou Tout à la question qui s'affiche : 
```
Set-ExecutionPolicy RemoteSigned
```
* Se placer dans le dossier où se situe le script (là où se trouve la solution, ou un cran au-dessus - récursif) et exécuter le script : 

```
./migrateToAutomaticPackageRestore.ps1
```

## Pour chaque job Jenkins, dans la partie Build

* Propriétés avancées pour chaque job :
    * Utiliser un répertoire de travail spécifique (voir variables définition d'environnement plus haut => C:\Jenkins\dvlp\)

* En amont de la phase de build: restauration des packages Nuget via la ligne de commandes batch (fonctionne aussi avec des .csproj) :
```
%NUGET% restore "%WORKSPACE%\SolutionFile.sln"
```

## Build en configuration Release : 

* Sélectionner: MSBUILD
* La solution à builder :

```
${WORKSPACE}SolutionFile.sln
```

Les arguments de la ligne de commande :

```
/v:m /t:Clean,Build /p:Configuration=Release /p:Platform="Any CPU"
```

    * Revoir peut-être la verbosity => /v:m
    
* En aval de la phase de build : exécution des tests via la ligne de commande bacth :

```
%NUNIT% "%WORKSPACE%\project\bin\Release\test.dll" /xml=Results.xml
EXIT /B %%EXITCODE%%
```

## Etapes post build

* Affichage du rapport de test NUnit : « Results.xml » (pas de nécessité de spécifier le path)
* Affichage du rapport du nombre de TODO :
    * Files to scan : **/*.cs
    * Normal priority : TODO (ignore case checked)
* Notification par email pour chaque build instable : email du destinataire définit dans la configuration
* Execution de l'analyse de code par SonarQube


## Mise en place de SonarQube :

* Téléchargement : [https://www.sonarqube.org/](https://www.sonarqube.org/) - actuellement en version 6.2
* Dézipper l'archive dans C:\sonarqube

* Préparer la base de données SQL Server :
    * créer une base de données nommée 'sonarqube' par exemple
    * le classement doit être obligatoirement de type : **SQL_Latin1_General_CP1_CS_AS**
    * crée une nouvelle connexion avec une authentification SQL Server et lui donner tous les droits sur la base
    * jouer ensuite cette requête sur la Bdd pour éviter les problèmes de format de date : 
```
ALTER LOGIN <nom_connexion> WITH DEFAULT_LANGUAGE = ENGLISH;
```

* Modifier la configuration dans le fichier **sonar.properties** qui se trouve dans le dossier **conf** de SonarQube.
    * le login et le mot de passe de l'utilisateur définit précédemment :
```
sonar.jdbc.username=LOGIN
sonar.jdbc.password=PASSWORD
```

    * la chaîne de connexion en utilisant l'authentification SQL Server (ici une Bdd SQLEXPRESS est utilisée) :
```
sonar.jdbc.url=jdbc:sqlserver://localhost\\SQLEXPRESS;databaseName=sonarqube;language=english
```

* Exécuter SonarQube
    * se placer dans le répertoire en rapport avec votre plateforme, ici C:\sonarqube\bin\windows-x86-64
    * et lancer : StartSonar.bat
    * on peut ensuite se connecter avec le compte admin (login: admin/ pwd: admin) sur: http://localhost:9000

## Configurer Jenkins et SonarQube

* SonarQube :
    * créer un utilisateur : Administration > Security > Users  ->  Create User
    * une fois l'utilisateur créé, générer un token et conserver sa valeur (il sera utilisé dans Jenkins) 
    * crée ensuite une projet : Administration > Projects > Management  ->  Create Project
        * renseigner un nom et une clé (elle sera également utilisée dans Jenkins)

* Jenkins :
    * ajouter le plugin Sonar : Administrer Jenkins > Gestion des Plugins
        * télécharger SonarQube Plugin (actuellement en version 2.5)
    * ajouter un serveur SonarQube : Administrer Jenkins > Configurer le système
        * ajouter une installation SonarQube
        * définir un nom, l'url (http://localhost:9000 par défaut), version 5.3 ou + et enfin le token qu'on avait généré et sauvegardé auparavant
    * ajouter le scanner SonarQube : Administrer Jenkins > Configuration globale des outils
        * ajouter un Scanner SonarQube pour MSBuild 
        * définir un nom, cocher "Install automatically"
        * dans la liste déroulante "Install from GitHub" séléctionner : "SonarQube Scanner for MSBuild 2.2"

* Job Jenkins :
    * pour chaque job, entourer les étapes de build (MSBuild) d'une étape "Préparer une analyse SonarQube Scanner pour MSBuild" et terminer par une étape "Terminer une analyse SonarQube Scanner pour MSBuild"
    * dans l'étape de préparation, renseigner la clé du projet (voir l'étape de création du projet dans SonarQube) et son nom ainsi qu'un numéro version (1.0 fera l'affaire)
    * lancer enfin un build du Job Jenkins et les badges SonarQube apparaitront dans le rapport