---
lab:
  title: 'Übung 03: Automatisieren von Azure-Auslastungsprüfungen mithilfe von GitHub-Aktionen '
  module: 'Module 3: Implement Azure Load Testing'
---

# Übersicht

In dieser Übung erfahren Sie, wie Sie GitHub-Aktionen konfigurieren, um eine Beispiel-Web-App bereitzustellen und mithilfe von Azure Load Testing einen Auslastungstest zu starten.

Dieses Lab deckt Folgendes ab:

* Erstellen Sie App Service- und Load Testing-Ressourcen in Azure.
* Erstellen und konfigurieren Sie einen Dienstprinzipal, um GitHub Actions-Workflows zum Ausführen von Aktionen in Ihrem Azure-Konto zu ermöglichen.
* Stellen Sie eine .NET 8-Anwendung mit einem GitHub Actions-Workflow in Azure App Service bereit.
* Aktualisieren Sie einen GitHub Actions-Workflow, um einen URL-basierten Load Test aufzurufen.

**Geschätzte Dauer des Abschlusses: 40 Minuten**

## Voraussetzungen

* Ein **Azure-Konto** mit einem aktiven Abonnement. Wenn Sie noch keines haben, können Sie sich unter [https://azure.com/free](https://azure.com/free) für eine kostenlose Testversion registrieren.
    * Ein vom Azure-Webportal unterstützter [Browser](https://learn.microsoft.com/azure/azure-portal/azure-portal-supported-browsers-devices).
    * Ein Microsoft-Konto oder ein Microsoft Entra-Konto mit der Rolle „Mitwirkender“ oder „Besitzer“ im Azure-Abonnement. Ausführliche Informationen finden Sie in den Artikeln zum [Auflisten von Azure-Rollenzuweisungen mithilfe des Azure-Portals](https://docs.microsoft.com/azure/role-based-access-control/role-assignments-list-portal) und [Anzeigen und Zuweisen von Administratorrollen in Azure Active Directory](https://docs.microsoft.com/azure/active-directory/roles/manage-roles-portal).
* Ein GitHub-Konto. Wenn Sie nicht über ein GitHub-Konto verfügen, das Sie für dieses Lab verwenden können, folgen Sie den Anweisungen unter [Für ein neues GitHub-Konto registrieren](https://github.com/join), um ein Konto zu erstellen.


## Anweisungen

## Übung 1: Importieren Sie die Beispiel-App in Ihr GitHub-Repository

In dieser Übung importieren Sie das Repository der [Beispiel-App für den Azure-Auslastungstest](https://github.com/MicrosoftLearning/azure-load-test-sample-app) in Ihr eigenes GitHub-Konto.

### Aufgabe 1: Importieren des eShopOnWeb-Repository

1. Navigieren Sie in Ihrem Webbrowser zu GitHub [http://github.com](http://github.com) und melden Sie sich mit Ihrem Kontos an.
1. Starten Sie den Importvorgang [https://github.com/new/import](https://github.com/new/import).
1. Geben Sie die folgenden Informationen auf der Seite **Importieren Sie Ihr Projekt in GitHub** ein.

    | Einstellung | Aktion |
    |--|--|
    | **Die URL für Ihr Quell-Repository** | Geben Sie `https://github.com/MicrosoftLearning/azure-load-test-sample-app` ein. |
    | **Besitzer** | Wählen Sie Ihren GitHub-Alias |
    | **Repositoryname** | bennen Sie ihr Repository |
    | **Datenschutz** | Nach Auswahl des **Besitzenden** erscheinen die Datenschutzoptionen. Wählen Sie **Öffentlich** aus. |

1. Wählen Sie **Import starten** und warten Sie, bis der Importvorgang abgeschlossen ist.
1. Wählen Sie auf der neuen Repository-Seite **Einstellungen** und dann **Aktionen > Allgemein** im linken Navigationsbereich.
1. Wählen Sie im Abschnitt **Aktionsberechtigungen** der Seite die Option **Alle Aktionen und wiederverwendbare Workflows zulassen** und wählen Sie dann **Speichern**.

## Übung 2: Erstellen von Ressourcen in Azure

In dieser Übung erstellen Sie die Ressourcen in Azure, die Sie benötigen, um die App bereitzustellen und den Test auszuführen. 

### Aufgabe 1: Erstellen von Ressourcen mit der Azure CLI.

In diesem Szenario erstellen Sie die folgenden Azure-Ressourcen:

* Ressourcengruppe
* App Service-Plan
* App Service-Instanz
* Laden Sie die Testinstanz

1. Navigieren Sie in Ihrem Browser zum Azure-Portal [https://portal.azure.com](https://portal.azure.com).
1. Öffnen Sie die **Cloud-Shell** und wählen Sie den **Bash**-Modus. **Hinweis:** Möglicherweise müssen Sie den dauerhaften Speicher konfigurieren, wenn Sie die Cloud Shell zum ersten Mal starten.

1. Führen Sie die folgenden Befehle nacheinander aus, um Variablen zu erstellen, die in den Befehlen der übrigen Schritte verwendet werden. Ersetzen Sie `<mylocation>` durch Ihren bevorzugten Speicherort.

    ```
    myLocation=<mylocation>
    myAppName=az2006app$RANDOM
    ```
1. Führen Sie den folgenden Befehl aus, um die Ressourcengruppe zu erstellen, die die anderen Ressourcen enthält.

    ```
    az group create -n az2006-rg -l $myLocation
    ```

1. Führen Sie den folgenden Befehl aus, um den Ressourcenanbieter für den **Azure App Service** zu registrieren.

    ```bash
    az provider register --namespace Microsoft.Web
    ```

1. Führen Sie den folgenden Befehl aus, um den Plan für den App-Dienst zu erstellen. **Hinweis:** Der im App-Service Plan verwendete B1-Plan kann Kosten verursachen. 

    ```
    az appservice plan create -g az2006-rg -n az2006webapp-plan --sku B1
    ```

1. Führen Sie den folgenden Befehl aus, um die Dienstinstanz für die App zu erstellen.

    ```
    az webapp create -g az2006-rg -p az2006webapp-plan -n $myAppName --runtime "dotnet:8"
    ```

1. Führen Sie den folgenden Befehl aus, um eine Auslastungstestressource zu erstellen. Wenn Sie einen Prompt erhalten, um die **Auslastungs**-Erweiterung zu installieren, wählen Sie Ja.

    ```
    az load create -n az2006loadtest -g az2006-rg --location $myLocation
    ```

1. Führen Sie die folgenden Befehle aus, um Ihre Abonnement-ID zu erhalten. **Hinweis:** Achten Sie darauf, die Ausgabe der Befehle zu kopieren und zu speichern, der Wert der Abonnement-ID wird später in dieser Übung verwendet.

    ```
    subId=$(az account list --query "[?isDefault].id" --output tsv)
    
    echo $subId
    ```

### Aufgabe 2: Erstellen Sie das Dienstprinzipal und konfigurieren Sie die Berechtigung

In dieser Aufgabe erstellen Sie ein Dienstprinzipal für die App und konfigurieren es für die föderierte OpenID Connect-Authentifizierung.

1. Suchen Sie im Azure-Portal nach **Microsoft Entra ID** und navigieren Sie zu dem Dienst.

1. Wählen Sie im linken Navigationsbereich **App-Registrierungen** in der Gruppe **Verwalten**. 

1. Wählen Sie **+ Neue Registrierung** im Hauptbereich und geben Sie `GH-Action-webapp` als Namen ein, und wählen Sie dann **Registrieren**.

    >**WICHTIG:** Kopieren und speichern Sie die beiden Werte **Anwendungs-(Client-)ID** und **Verzeichnis-(Mandanten-)ID** für später in dieser Übung.


1. Wählen Sie im linken Navigationsbereich **Zertifikate und Geheimnisse** in der Gruppe **Verwalten** und dann im Hauptfenster **Verbundanmeldeinformationen** aus. 

1. Wählen Sie **Anmeldeinformationen hinzufügen** aus und wählen Sie dann **GitHub-Aktionen, die Azure-Ressourcen bereitstellen** in der Auswahl-Dropdownliste aus.

1. Geben Sie die folgenden Informationen im Abschnitt **Verbinden Sie Ihr GitHub-Konto** ein. **Hinweis:** Bei diesen Feldern wird die Groß-/Kleinschreibung beachtet. 

    | Feld | Aktion |
    |--|--|
    | Organisation | Geben Sie den Namen des Benutzers oder der Organisation ein. |
    | Repository | Geben Sie den Namen des Repositorys ein, das Sie zuvor in der Übung angelegt haben. |
    | Entitätstyp | Wählen Sie **Branch** aus. |
    | Name des GitHub-Branch | Geben Sie **main** ein. |

1. Im Abschnitt **Details der Anmeldeinformationen** geben Sie Ihren Anmeldeinformationen einen Namen und wählen dann **Hinzufügen**.

### Aufgabe 3: Zuweisen von Rollen an das Dienstprinzipal

In dieser Aufgabe weisen Sie dem Dienstprinzipal die erforderlichen Rollen für den Zugriff auf Ihre Ressourcen zu.

1. Führen Sie die folgenden Befehle aus, um die Rolle „Auslastungstest (Testmitwirkender)“ zuzuweisen, damit der GitHub-Workflow die Ressourcentests zur Ausführung senden kann. 

    ```
    spAppId=$(az ad sp list --display-name GH-Action-webapp --query "[].{spID:appId}" --output tsv)

    loadTestId=$(az resource show -g az2006-rg -n az2006loadtest --resource-type "Microsoft.LoadTestService/loadtests" --query "id" -o tsv)

    az role assignment create --assignee $spAppId --role "Load Test Contributor"  --scope $loadTestId
    ```

1. Führen Sie den folgenden Befehl aus, um die Rolle „Mitwirkender“ zuzuweisen, damit der GitHub-Workflow die App im App-Dienst bereitstellen kann. 

    ```
    rgId=$(az group show -n az2006-rg --query "id" -o tsv)
    
    az role assignment create --assignee $spAppId --role contributor --scope $rgId
    ```

## Übung 3: Bereitstellen und Testen der Web App mithilfe von GitHub-Aktionen

In dieser Übung konfigurieren Sie Ihr Repository, um die enthaltenen Workflows auszuführen.

* Die Workflows befinden sich im Ordner *.github/workflows* des Repos.
* Beide Workflows, *deploy.yml* und *loadtest.yml*, sind so konfiguriert, dass sie manuell ausgeführt werden.

Bei dieser Übung bearbeiten Sie Repository-Dateien im Browser. Nachdem Sie eine Datei zur Bearbeitung ausgewählt haben, haben sie folgende Möglichkeiten:
* Wählen Sie **Bearbeiten an Ort und Stelle** und bestätigen Sie die Änderungen, wenn Sie die Bearbeitung beendet haben. 
* Öffnen Sie die Datei mit **github.dev**, um sie mit Visual Studio Code im Browser zu bearbeiten. Wenn Sie diese Option wählen, können Sie zum Standard-Repository zurückkehren, indem Sie **Zurück zum Repository** im oberen Menü wählen.

    ![Screenshot der Bearbeitungsoptionen.](./media/github-edit-options.png)

### Aufgabe 1: Konfigurieren von Geheimnissen

In dieser Aufgabe fügen Sie Ihrem Repo Geheimnisse hinzu, um die Workflows zu aktivieren, die sich in Ihrem Namen bei Azure anmelden und Aktionen durchführen.

1. Navigieren Sie in Ihrem Webbrowser zu [GitHub](https://github.com) und wählen Sie das Repository aus, das Sie für diese Übung erstellt haben. 
1. Wählen Sie **Einstellungen** am oberen Rand des Repo.
1. Wählen Sie im linken Navigationsbereich **Geheimnisse und Variablen** und dann **Aktionen**.
1. Im Abschnitt **Repository secrets** fügen Sie die folgenden drei geheimen Schlüssel hinzu. Sie fügen einen geheimen Schlüssel hinzu, indem Sie **Neues Repository-Geheimnis** wählen.

    | Name | Geheimnis |
    |--|--|
    | AZURE_CLIENT_ID | Geben Sie die **Anwendungs-(Client-)ID** ein, die Sie zuvor in der Übung gespeichert haben. |
    | AZURE_TENANT_ID | Geben Sie die **Verzeichnis (Mandant) ID** ein, die Sie zuvor in der Übung gespeichert haben. |
    | AZURE_SUBSCRIPTION_ID | Geben Sie den Wert der Abonnement-ID ein, den Sie zuvor in der Übung gespeichert haben. |

### Aufgabe 2: Bereitstellen der Web App

1. Wählen Sie die Datei *deploy.yml* im Ordner *.github/workflows*.

1. Bearbeiten Sie die Datei und ändern Sie im Abschnitt **env:** den Wert der Variablen `AZURE_WEB_APP`. Ersetzen Sie `<your web app name>**` durch den Namen der zuvor in dieser Übung erstellten Web App. Committen Sie die Änderung.

1. Nehmen Sie sich etwas Zeit, um den Inhalt des Workflows zu überprüfen.

1. Wählen Sie **Aktionen** in der oberen Navigationsleiste Ihres Repositorys aus. 

1. Wählen Sie **Erstellen und Veröffentlichen** im linken Navigationsbereich.

1. Wählen Sie das Dropdown-Menü **Workflow ausführen** und dann **Workflow ausführen** aus, wobei Sie die Standardeinstellung **Branch: Main** beibehalten. Es kann einige Zeit dauern, bis der Workflow startet.

Wenn es Probleme mit dem erfolgreichen Abschluss des Workflows gibt, wählen Sie den Workflow **Erstellen und Veröffentlichen** und wählen Sie dann **Erstellen** auf dem nächsten Bildschirm. Dies liefert detaillierte Informationen über den Workflow und kann helfen, das Problem zu diagnostizieren, das den erfolgreichen Abschluss des Workflows verhindert hat.

### Aufgabe 3: Ausführen eines Auslastungstests

1. Wählen Sie die Datei *loadtest.yml* im Ordner *.github/workflows*.

1. Bearbeiten Sie die Datei und ändern Sie im Abschnitt **env:** den Wert der Variablen `AZURE_WEB_APP`. Ersetzen Sie `<your web app name>**` durch den Namen der zuvor in dieser Übung erstellten Web App. Committen Sie die Änderung.

1. Nehmen Sie sich etwas Zeit, um den Inhalt des Workflows zu überprüfen.

1. Wählen Sie **Aktionen** in der oberen Navigationsleiste Ihres Repositorys aus. 

1. Wählen Sie **Auslastungstest** im linken Navigationsbereich.

1. Wählen Sie das Dropdown-Menü **Workflow ausführen** und dann **Workflow ausführen** aus, wobei Sie die Standardeinstellung **Branch: Main** beibehalten. Es kann einige Zeit dauern, bis der Workflow startet.

    >**Hinweis:** Es kann 5-10 Minuten dauern, bis der Workflow abgeschlossen ist. Der Test läuft zwei Minuten lang, und es kann einige Minuten dauern, bis der Auslastungstest in die Warteschlange gestellt und in Azure gestartet wird. 

Wenn es Probleme mit dem erfolgreichen Abschluss des Workflows gibt, wählen Sie den Workflow **Erstellen und Veröffentlichen** und wählen Sie dann **Erstellen** auf dem nächsten Bildschirm. Dies liefert detaillierte Informationen über den Workflow und kann helfen, das Problem zu diagnostizieren, das den erfolgreichen Abschluss des Workflows verhindert hat.

#### Optional

Die Datei *config.yaml* im Stammverzeichnis des Repositorys legt die Fehlerkriterien für den Auslastungstest fest. Wenn Sie erzwingen wollen, dass der Auslastungstest fehlschlägt, führen Sie die folgenden Schritte aus.

1. Bearbeiten Sie die *Datei config.yaml* im Stammverzeichnis des Repositorys.
1. Ändern Sie den Wert im Feld `- p90(response_time_ms) > 4000` auf einen niedrigen Wert. Die Änderung in `- p90(response_time_ms) > 50` wird höchstwahrscheinlich dazu führen, dass der Test fehlschlägt. Das bedeutet, dass die App in 90 % der Fälle innerhalb von 50 ms antworten wird. 

### Aufgabe 4: Ansicht der Ergebnisse des Auslastungstests

Wenn Sie einen Auslastungstest über Ihre CI/CD-Pipeline ausführen, können Sie die Zusammenfassungsergebnisse direkt im CI/CD-Ausgabeprotokoll anzeigen. Da die Testergebnisse als Pipeline-Artefakt gespeichert wurden, können Sie auch eine CSV-Datei zur weiteren Berichterstellung herunterladen.

![Screenshot: Protokollierungsinformationen des Workflows](./media/github-actions-workflow-completed.png)

## Übung 4: Bereinigen von Ressourcen

In dieser Übung löschen Sie die zuvor in der Übung erstellten Ressourcen.

1. Navigieren Sie zum Azure-Portal [https://portal.azure.com](https://portal.azure.com) und starten Sie die Cloud Shell. Wählen Sie die **Bash**-Shell-Sitzung.

1. Führen Sie den folgenden Befehl aus, um die Ressourcengruppe `az2006-rg` zu löschen. Außerdem werden der App Service-Plan und diese Instanz entfernt.

    ```
    az group delete -n az2006-rg --no-wait --yes
    ```

    >**Hinweis**: Der Befehl wird asynchron ausgeführt (festgelegt mit dem Parameter `--no-wait`), so dass Sie zwar unmittelbar danach innerhalb derselben Bash-Sitzung einen anderen Azure CLI-Befehl ausführen können, es aber einige Minuten dauert, bis die Ressourcengruppen tatsächlich entfernt werden.

## Überprüfung

In dieser Übung haben Sie GitHub Action-Workflows implementiert, die eine Azure Web App bereitstellen und deren Auslastung testen.
