Reflexion: Externer Zugriff auf Services mit Kubernetes Ingress
1. Warum ist es notwendig, die lokale Hosts-Datei anzupassen, um das Hostnamen-basierte Routing mit Ingress in meinem lokalen Cluster zu testen?
Ich muss die lokale Hosts-Datei anpassen, weil das Domain Name System (DNS) im Internet den Hostnamen myapp.local nicht kennt. Es handelt sich dabei um einen fiktiven Hostnamen, den ich nur für lokale Tests verwende. Damit mein Browser (oder ein anderes Tool auf meinem Rechner) weiß, wohin Anfragen an myapp.local geschickt werden sollen, hinterlege ich diese Zuordnung manuell in der Hosts-Datei meines Betriebssystems. Diese Datei wirkt wie ein lokales DNS-Override. Ohne diesen Eintrag würde mein System versuchen, myapp.local über öffentliche DNS-Server aufzulösen – das würde fehlschlagen oder als Websuche behandelt werden. Mit dem Eintrag <Cluster-IP> myapp.local sorge ich dafür, dass Anfragen direkt an die IP-Adresse des Ingress Controllers im lokalen Kubernetes-Cluster gehen.

2. Wie steuert die Ingress Resource (my-ingress.yaml), welcher Service (Nginx oder HTTPBin) eine Anfrage erhält, wenn ich http://myapp.local/ oder http://myapp.local/api/anything im Browser aufrufe?
Das Routing wird über folgende Felder in der Ingress-Ressource my-ingress.yaml gesteuert:

spec.ingressClassName: Gibt an, welcher Ingress Controller zuständig ist, z. B. nginx.

spec.rules: Enthält eine Liste der Routingregeln.

host: myapp.local: Diese Regeln gelten nur, wenn die Anfrage an diesen Hostnamen gerichtet ist. Der Ingress Controller prüft dabei den Host-Header.

http.paths: Enthält Pfadregeln innerhalb des angegebenen Hosts.

Für http://myapp.local/:

path: / mit pathType: Prefix matcht alle Anfragen, die mit / beginnen.

Der Traffic wird an den Service nginx-service auf Port 80 weitergeleitet.

Für http://myapp.local/api/anything:

path: /api mit pathType: Prefix matcht alle Pfade, die mit /api beginnen (also auch /api/anything).

Der Traffic geht an den httpbin-service, ebenfalls auf Port 80.

Der Ingress Controller empfängt also die Anfrage, prüft den Hostnamen und dann die Pfade. Die erste Regel, die passt, bestimmt den Service, an den die Anfrage weitergeleitet wird.

3. Was passiert mit einer Anfrage an http://andererhost.local/ oder an http://<Cluster-IP>/some/path/, wenn diese nicht in meinen Ingress-Regeln definiert sind (und kein Default Backend existiert)?
Anfrage an http://andererhost.local/:
Wenn dieser Hostname in keiner Ingress-Ressource vorkommt und auch kein defaultBackend definiert ist, antwortet der Ingress Controller in der Regel mit einem HTTP 404 Not Found. Die Anfrage wurde zwar empfangen, aber es gibt keine passende Regel dafür.

Anfrage an http://<Cluster-IP>/some/path/:
Diese Anfrage wird meist ohne einen Hostnamen im Host-Header gesendet (oder mit der IP selbst als Host). Wenn es in meiner Ingress-Konfiguration keine "Catch-all"-Regel ohne host:-Feld gibt, findet der Controller auch hier keine passende Route. Das Ergebnis ist ebenfalls typischerweise ein HTTP 404 Not Found.

In beiden Fällen bedeutet der 404-Fehler, dass der Ingress Controller zwar erreichbar ist, aber keine Route konfiguriert wurde, um die Anfrage weiterzuleiten.

4. Welche Vorteile bietet das Routing basierend auf Hostnamen und Pfaden mit Ingress, verglichen mit der Nutzung von NodePorts oder separaten LoadBalancern für jeden Service – besonders, wenn ich viele Services betreibe?
Ingress bietet mir viele Vorteile gegenüber NodePorts oder einzelnen LoadBalancern:

Zentraler Zugriffspunkt:
Ich muss mir nur eine IP-Adresse und Portkombination merken, da der Ingress Controller als zentraler Einstiegspunkt fungiert.

Kostenersparnis (in der Cloud):
In einer Cloud-Umgebung würde jeder LoadBalancer extra kosten. Mit einem zentralen Ingress Controller kann ich viele Services mit nur einem LoadBalancer betreiben.

Flexibles Routing mit Hostnamen und Pfaden:

Ich kann mehrere Anwendungen auf einer IP/Port-Kombination betreiben, z. B. app1.example.com, app2.example.com.

Ich kann einzelne Pfade an unterschiedliche Services weiterleiten – z. B. /api, /auth, /ui – und so Microservices einfach strukturieren.

TLS/SSL-Terminierung:
Der Ingress Controller kann HTTPS-Zertifikate verwalten und den verschlüsselten Traffic entschlüsseln, bevor er an die Services geht. Ich entlaste damit meine Anwendungen und verwalte Zertifikate zentral.

Einfache Konfiguration und Verwaltung:
Ich konfiguriere nur Routingregeln in einer YAML-Datei und muss keine Ports oder IP-Adressen manuell verwalten.

Standardisierte, Kubernetes-native Lösung:
Ingress-Ressourcen lassen sich leicht in Versionskontrolle und Automatisierung einbinden und sind unabhängig vom Cloud-Anbieter nutzbar.

Im Gegensatz dazu:

NodePort:
Jeder Service bekommt einen eigenen hohen Port, den ich manuell kennen muss. Das ist unübersichtlich und nicht besonders benutzerfreundlich.

LoadBalancer:
Für jeden Service brauche ich einen eigenen LoadBalancer – das ist teuer und aufwändig zu verwalten.

Ingress vereinfacht meinen Setup deutlich, spart Ressourcen und bietet eine elegante, skalierbare Lösung für den externen Zugriff auf viele Services.








