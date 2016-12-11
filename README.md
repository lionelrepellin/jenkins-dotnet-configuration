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
* SQL Server Express (uniquement si des tests *unitaires* portent sur la base de données)

## Composants externes

* Git for Windows 
    * Téléchargement : [https://git-scm.com/download/win](https://git-scm.com/download/win)
    * A installer dans : C:\Git

* Nuget v3.5.0
    * Téléchargement : [https://dist.nuget.org/win-x86-commandline/v3.5.0/NuGet.exe](https://dist.nuget.org/win-x86-commandline/v3.5.0/NuGet.exe)
    * A installer dans : C:\Nuget

* NUnit v2.6.4
    * Téléchargement : [https://github.com/nunit/nunitv2/releases/tag/2.6.4](https://github.com/nunit/nunitv2/releases/tag/2.6.4)
    * A installer dans : C:\Nunit (n'accepte pas les dossiers du type « C:\Program Files (x86) »)
    * Utiliser la même version que dans Visual Studio (avec Nunit TestAdapter 2.0)

* MsBuild
    * Utiliser la version fournie avec VS dans : C:\Program Files (x86)\MSBuild\14.0\Bin\
    * Comportement différent avec C:\Windows\Microsoft.NET\Framework64\v4.0.30319

## Plugins Jenkins à installer

* MSBuild
* Nunit
* Task Scanner (TODO)
* thinBackup

## Variables d’environnement à définir

* Prise en compte des différents chemins d'accès des composants externes ci-dessus 
* Nom de l'environnement de développement
* Nom du profile de déploiement

## Configuration de la notification par email

* Saisir l'adresse email de l’administrateur (personne à qui sera envoyé les mails d’échecs de builds)

## Supprimer les balises targets / imports dans tous les .csproj via script PowerShell

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
    *   Utiliser un répertoire de travail spécifique (voir variables définition d'environnement plus haut => C:\Jenkins\dvlp\)

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
