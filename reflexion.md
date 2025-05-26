# Reflexion: Externer Zugriff auf Services mit Kubernetes Ingress

## 1. Warum ist es notwendig, die lokale Hosts-Datei anzupassen, um das Hostnamen-basierte Routing mit Ingress in deinem lokalen Cluster zu testen?

Die lokale Hosts-Datei muss angepasst werden, weil das Domain Name System (DNS) im Internet den Hostnamen `myapp.local` nicht kennt. Es handelt sich um einen für lokale Testzwecke gewählten, fiktiven Hostnamen. Damit dein Browser (oder ein anderes Client-Tool auf deinem lokalen Rechner) weiß, an welche IP-Adresse die Anfrage für `myapp.local` gesendet werden soll, muss diese Zuordnung manuell in der Hosts-Datei deines Betriebssystems hinterlegt werden. Die Hosts-Datei dient als lokaler "DNS-Override". Ohne diesen Eintrag würde dein System versuchen, `myapp.local` über öffentliche DNS-Server aufzulösen, was fehlschlagen würde, oder es würde als Suchanfrage interpretiert. Durch den Eintrag `<Cluster-IP> myapp.local` wird dem System mitgeteilt, dass Anfragen an `myapp.local` direkt an die IP-Adresse des Ingress Controllers im lokalen Kubernetes-Cluster gesendet werden sollen.

## 2. Wie hat die Ingress Resource (my-ingress.yaml) gesteuert, welcher Service (Nginx oder HTTPBin) eine Anfrage erhält, wenn der Browser http://myapp.local/ oder http://myapp.local/api/anything aufruft? (Nenne die relevanten Felder in der YAML und wie sie zusammenarbeiten).

Die Ingress Resource `my-ingress.yaml` steuert das Routing über folgende Felder:

*   **`spec.ingressClassName`**: Dieses Feld (z.B. `nginx`) gibt an, welcher Ingress Controller für die Verarbeitung dieser Ingress-Regel zuständig ist.
*   **`spec.rules`**: Dies ist eine Liste von Routing-Regeln.
    *   **`host: myapp.local`**: Definiert, dass die nachfolgenden Pfadregeln nur für Anfragen gelten, die an den Hostnamen `myapp.local` gerichtet sind. Der Ingress Controller prüft den `Host`-Header der HTTP-Anfrage.
    *   **`http.paths`**: Innerhalb der Host-Regel definiert dies eine Liste von Pfad-basierten Regeln.
        *   Für `http://myapp.local/`:
            *   **`path: /`**: Diese Regel matcht Anfragen, deren Pfad mit `/` beginnt.
            *   **`pathType: Prefix`**: Gibt an, dass der Pfad als Präfix gematcht wird.
            *   **`backend.service.name: nginx-service`**: Leitet den Traffic an den Service namens `nginx-service`.
            *   **`backend.service.port.number: 80`**: Gibt den Port des `nginx-service` an, an den der Traffic weitergeleitet wird.
        *   Für `http://myapp.local/api/anything`:
            *   **`path: /api`**: Diese Regel matcht Anfragen, deren Pfad mit `/api` beginnt (also auch `/api/anything`).
            *   **`pathType: Prefix`**: Gibt an, dass der Pfad als Präfix gematcht wird.
            *   **`backend.service.name: httpbin-service`**: Leitet den Traffic an den Service namens `httpbin-service`.
            *   **`backend.service.port.number: 80`**: Gibt den Port des `httpbin-service` an.

Zusammenfassend: Der Ingress Controller empfängt die Anfrage. Er prüft zuerst den Hostnamen (`myapp.local`). Wenn dieser übereinstimmt, prüft er die Pfade in der Reihenfolge ihrer Definition (oder gemäß der Spezifität, abhängig vom Controller). Die erste übereinstimmende `path`-Regel bestimmt dann den `backend.service`, an den die Anfrage weitergeleitet wird.

## 3. Was passiert mit einer Anfrage an http://andererhost.local/ oder an http://<Cluster-IP>/some/path/, wenn diese nicht in deinen Ingress-Regeln für myapp.local definiert sind (und kein Default Backend existiert)?

*   **Anfrage an `http://andererhost.local/`**:
    Wenn der Hostname `andererhost.local` in keiner `spec.rules[].host`-Definition in irgendeiner Ingress-Ressource im Cluster vorkommt (oder zumindest nicht von diesem Ingress Controller verwaltet wird), und kein `defaultBackend` auf Ingress-Ebene definiert ist, wird der Ingress Controller die Anfrage typischerweise mit einem HTTP-Statuscode `404 Not Found` beantworten. Der Controller findet keine Regel, die für diesen Hostnamen zuständig ist.

*   **Anfrage an `http://<Cluster-IP>/some/path/`**:
    Diese Anfrage wird ohne einen spezifischen `Host`-Header (oder mit der IP-Adresse als Host-Header) an den Ingress Controller gesendet.
    *   Wenn es eine Ingress-Regel ohne `host`-Feld gibt (eine sogenannte "catch-all" Regel für Anfragen ohne spezifischen Hostnamen-Match), würde diese greifen.
    *   Wenn in `my-ingress.yaml` (und anderen Ingress-Ressourcen) alle Regeln ein `host`-Feld (wie `myapp.local`) spezifizieren und kein `defaultBackend` im Ingress Controller oder in einer Ingress-Ressource definiert ist, wird der Ingress Controller ebenfalls typischerweise einen HTTP `404 Not Found` zurückgeben. Er findet keine Regel, die für Anfragen an die reine IP-Adresse (oder einen nicht übereinstimmenden Host-Header) und den gegebenen Pfad zuständig ist.

In beiden Fällen signalisiert der 404-Fehler, dass der Ingress Controller die Anfrage zwar empfangen hat, aber keine konfigurierte Route zum Weiterleiten des Traffics finden konnte.

## 4. Welche Vorteile bietet das Routing basierend auf Hostnamen und Pfaden mit Ingress, verglichen mit der Nutzung von NodePorts oder separaten LoadBalancern für jeden Service, besonders wenn du viele Services hast?

Ingress bietet signifikante Vorteile gegenüber NodePorts oder separaten LoadBalancern, besonders bei vielen Services:

*   **Zentralisierter Zugriffspunkt**: Ingress agiert als einziger oder einer von wenigen Eintrittspunkten in den Cluster. Man muss sich nicht mehrere IP-Adressen und Ports merken/verwalten.
*   **Kostenersparnis**: Cloud-Provider berechnen oft Kosten pro LoadBalancer. Mit Ingress kann ein einziger LoadBalancer (der vom Ingress Controller genutzt wird) den Traffic für viele Services verteilen, was Kosten spart. NodePorts können zwar ohne externen LoadBalancer auskommen, sind aber für den direkten externen Zugriff oft unpraktisch (zufällige hohe Portnummern, IP des Nodes).
*   **Hostnamen- und Pfad-basiertes Routing**:
    *   Ermöglicht das Hosten mehrerer Anwendungen/Domains (`app1.example.com`, `app2.example.com`) über dieselbe IP-Adresse und denselben Port (typischerweise 80/443).
    *   Ermöglicht das Aufteilen einer einzelnen Anwendung in Microservices, die unter verschiedenen Pfaden desselben Hostnamens erreichbar sind (`myapp.com/ui`, `myapp.com/api`, `myapp.com/auth`).
*   **SSL/TLS-Terminierung**: Ingress Controller können SSL/TLS-Zertifikate verwalten und den HTTPS-Traffic entschlüsseln, bevor er an die Backend-Services weitergeleitet wird. Dies entlastet die Anwendungs-Services von dieser Aufgabe und zentralisiert die Zertifikatsverwaltung.
*   **Vereinfachte Konfiguration**: Anstatt für jeden Service einen eigenen LoadBalancer oder NodePort zu konfigurieren und freizugeben, werden Routing-Regeln deklarativ in Ingress-Ressourcen definiert.
*   **Standardisierung**: Bietet eine Kubernetes-native Methode für das Layer-7-Routing, was die Portabilität zwischen verschiedenen Umgebungen (lokal, Cloud) verbessert.
*   **Weitere Features**: Viele Ingress Controller bieten zusätzliche Funktionen wie Rewrite-Regeln, Authentifizierung, Ratenbegrenzung etc. an einem zentralen Punkt.

Im Vergleich dazu:
*   **NodePort**: Exponiert einen Service auf einem statischen Port an jedem Node. Man benötigt die IP eines Nodes und den hohen NodePort. Bei vielen Services wird das unübersichtlich und erfordert oft einen vorgeschalteten LoadBalancer.
*   **LoadBalancer (Service-Typ)**: Erstellt für *jeden* Service einen eigenen externen LoadBalancer (in Cloud-Umfeldern). Das ist einfach, wird aber bei vielen Services schnell teuer und komplex in der Verwaltung der vielen IP-Adressen.

Ingress konsolidiert diese Aspekte und bietet eine flexiblere und effizientere Lösung für den externen Zugriff, insbesondere in komplexeren Szenarien mit vielen Services.