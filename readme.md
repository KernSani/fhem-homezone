# Die Idee:
Das Haus wird in Zonen aufgeteilt - das können einzelne Räume sein, aber auch z.B. ein Stockwerk. Ziel ist es jederzeit zu wissen, ob sich jemand (idealerweise auch wer - aber dahin ist noch ein weiter Weg) in dieser Zone aufhält.

## Anwesenheitswahrscheinlichkeit: 
Wenn eine Bewegung (oder ein anderes Ereignis, das "Anwesenheit" bedeutet) erkannt wird, ist es ziemlich wahrscheinlich (default 99%), dass jemand anwesend ist, je länger keine weitere Bewegung erkannt wird, desto unwahrscheinlicher wird es, dass noch jemand anwesend ist. Daher startet das Modul bei erkannter Bewegung einen Timer, je länger der Timer läuft, desto geringer wird die Wahrscheinlichkeit dass jemand da ist, bis sie schließlich auf 0 sinkt  (um keine zu Hohe Last zu erzeugen erfolgt der countdown in 10%-Schritten). Die Dauer des Timers kann tageszeit-abhängig gesteuert werden.

Offen/geschlossen: Eine Zone kann "offen" oder "geschlossen" sein. (repräsentiert z.B. durch einen Türkontakt). Wird innerhalb einer geschlossenen Zone eine Bewegung erkannt, wird Anwesenheit als sicher angenommen (100%) und bleibt auch so, bis die Zone geöffnet wird (Wasp-in-a-box Konzept). Der Klassiker für diesen Fall ist die ausgedehnte Toilettensitzung - solange ich auf der Toilette sitze wird keine Bewegung erkannt, ich will aber trotzdem nicht plötzlich im Dunklen sitzen.
Eine Ergänzung dazu (insbesondere für Zonen, in denen kein "geschlossen" Zustand erkannt werden kann ist folgendes Modell: Es wird bei erkannter Anwesenheit grundsätzlich angenommen, dass die Person weiterhin anwesend ist (auch wenn keine Bewegung erkannt wird) - Erst wenn in einer angrenzenden Zone (siehe nächster Punkt) eine Bewegung erkannt wird, wird angenommen, dass die Person sich weiter bewegt hat (und der Counter beginnt zu laufen)

Angrenzende Zonen. Häufig beeinflussen sich die Zonen gegenseitig, um obiges Beispiel nochmal aufzugreifen: Wenn ich aus der Toilette komme, will ich nicht in einen komplett dunklen Flur kommen, daher kann ich den Flur als "angrenzende Zone" der Toilette definieren. Dadurch wird die Anwesenheitswahrscheinlichkeit der Toilette auf den Flur gespiegelt (sofern Flur > 0% und Toilette > Flur).
Umgekehrt kann aber Bewegung in einer angrenzenden Zone bedeuten, dass jemand die aktuelle Zone verlassen hat. In diesem Fall ist eine Bewegung in der angrenzenden Zone ein Trigger für den Start des Counters (Attribut boxMode).

Hierarchie von Zonen: Mein ursprünglicher Gedanke war auch Es ist möglich Hierarchien von Zonen zu bauen, d.h. Zone Erdgeschoss enthält Zone Flur, Zone Wohnzimmer und Zone WC etc... Zumindest für meinen aktuellen Zweck reicht aber das Konzept der angrenzenden Räume aus. WC, Wohnzimmer und Flur haben alle "Erdgeschoss" als angrenzende Zone. Einer Zone können "Kinder" zugeordnet werden. Dadurch wird immer die höhste Anwesenheitswahrscheinlichkeit eines Raumes auf das Erdgeschoss gespiegelt (Und ich schalte die indirekte Beleuchtung z.B. erst aus, wenn sich niemand mehr im EG befindet).
Anmerkung: Derzeit verhält sich eine Zone mit Kindern genauso wie jede andere Zone, d.h. sie kann einen decay-Wert etc. enthalten - aus meiner Sicht macht das wenig Sinn - aber vielleicht hat jemand einen passenden Anwendungsfall.

Im Augenblick bildet das Modul nur den Status ab - die Steuerung muss dann über DOIF/notify etc... dazu gebaut werden. Das ist dann allerdings i.d.R. aber ziemlich trivial. Ich kann mir aber durchaus vorstellen einzelne (einfache) Steuerungen (Licht an bei > 0%, Licht aus bei 0%) ins Modul selbst zu integrieren.

Ich habe noch einige weitere Ideen und Gedanken (z.B. eine irgendwie geartete Integration mit RESIDENTS/ROOMMATES oder eine "intelligentere" Steuerung der Timer)

Das Modul ist - wie gesagt - bisher nur rudimentär ausgeprägt (keine Doku, keine Validierungen z.B. von Attributen usw... , Coding kann sicher optimiert werden etc...) aber wer Lust hat kann es gerne ausprobieren. Ich werde die nächste Zeit sicher noch weiter daran basteln und greife gerne neue Ideen und Vorschläge auf.

Definiert wird eine Zone ohne irgendwelche Parameter:
Code: [Auswählen]
define myZone homezone

Als Attribute gibt es bisher:
hz_occupancyEvent: Event das eine Anwesenheit signalisiert (also typischerweise Bewegung, kann aber auch z.B. "Fenster  auf" sein). Anzugeben in der Form <device regex>:<Event regex>, also z.B. myMotionSensor:state:.motion. Bei allen Event-Attributen können auch Komma-getrennte Listen von <device regex>:<Event regex> Paaren angegeben werden.
hz_absenceEvent: Event das eine sichere Abwesenheit signalisiert, z.B. wenn manuell das Licht ausgemacht wird, oder ein RESIDENTS device auf absent wechselt.
hz_openEvent: Event (z.B. Türkontakt) das anzeigt, dass ein Raum geöffnet wurde. Wenn eine Zone mehrere Türen hat, werden die Regexe für jede Tür durch Leerzeichen getrennt angegeben 
mySensorLeftDoor:state:.open mySensorRightDoor:state:.open
hz_closeEvent: Event (z.B. Türkontakt) das anzeigt, dass ein Raum geschlossen wurde:
mySensorLeftDoor:state:.close mySensorRightDoor:state:.close
hz_decay: "Verfallszeit" in Sekunden, also die Zeit, die vergehen soll bis der Timer nach erkannter Bewegung auf 0 runter gezählt hat. Wird ggf. überteuert von tageszeitabhängiger Steuerung (siehe hz_dayTimes)
hz_adjacent: Komma-getrennte Liste von angrenzenden Zonen (homezone Devices)
hz_state: Leerzeichen getrennte Liste von Wert:State Paaren -> Wenn Anwesenheitswahrscheinlichkeit >= Wert, dann wird State als state gesetzt (wird beim define wie folgt gesetzt: 100:present 50:likely 1:unlikely 0:absent)
hz_children: Komma-getrennte Liste von "Kindern", d.h. homezone devices. Der höhste Anwesenheitswert wird der Kinder wird an die Zone "vererbt". 
hz_dayTimes: Leerzeichen-getrennt Liste von Uhrzeit:Text Paaren. Dient zur Steuerung tageszeitabhängiger decay-Werte. Für jeden "text" wird ein zugehöriges decay-Userattribut erzeugt, mit dem die korrespondierenden Decay-Werte gepflegt werden. Das Attribut wird beim Define automatisch gesetzt und - sofern vorhanden - mit dem Wert aus HOMEMODE gefüllt. Die variablen $SUNRISE und $SUNSET können anstatt Uhrzeit verwendet werden.
hz_cmd_<state>: Userattribute werden beim setzen von hz_state erzeugt. Jedem der Attribute kann ein Kommando (z.B. set myLight on) mitgegeben werden, das beim Eintritt des states ausgeführt wird. Mehrere Kommandos werden durch ; getrennt. Statt FHEM Kommandos ist auch Perl Code (durch {} umschlossen) erlaubt. In Perlcode wird $name durch den Devicenamen ersetzt.
hz_luminanceReading:Reading in der Form <device>:<reading> das den Helligkeitswert für die Zone angibt
hz_lumiThreshold zwei durch ":" getrennte Werte für die untere und obere Schwelle der Helligkeit (aus hz_luminanceReading) bei der geschaltet werden soll (hz_cmd_<state). z.B. 0:40 (nur wenn Helligkeit < 40 ist wird Kommando ausgeführt), untere und obere Schwelle können auch leer bleiben (z.B. "200:" Es wird nur geschaltet, wenn die Helligkeit über 200 liegt) 
hz_lumiThreshold_<state>: Userattribute werden beim setzen von hz_state erzeugt. Jedem der Attribute kann ein threshold übergeben werden, der den default threshold für diesen state übersteuert.
disabled/disabledForIntervals wie üblich, dekativiert das Device
hz_disableOnlyCmdsWenn ein device disabled ist (egal ob permanent oder "for Intervall) wird - sofern dieses Attribut auf "1" steht trotzdem noch Anwesenheit erkannt und geloggt wie üblich, es werden aber keine Cmds ausgeführt.

Set Befehle gibt es Folgende:
occupied: Manuelles setzen einer Zahl zwischen 0 und 100 für die Anwesenheitswahrscheinlichkeit
open: manuelles setzen des open Status, als optionaler Parameter kann das auslösende device mitgegeben werden (reading lastZone)
closed: manuelles setzen des closed Status, als optionaler Parameter kann das auslösende device mitgegeben werden (reading lastZone)
active: aktivieren eines deaktivierten Devices
inactive: deaktivieren eines Devices, als optionales Argument kann ein Zeitraum (in Sekunden) mitgegeben werden, für den das Device deaktiviert wird (Attribut hz_disableOnlyCmds wird bei set inactive identisch zum Attribut disable behandelt)

Als readings werden erzeugt:
occupied: Zahl zwischen 0 und 100 für die Anwesenheitswahrscheinlichkeit
condition: open oder closed (im Falle mehrerer Türen auch "partly closed"
state: siehe Attribut hz_state
lastLumi: der letzte gelesene Helligkeitswert (bei der letzten Statusänderung)
lastZone: das letzte Device, das einen occupied Wert im aktuellen Device gesetzt hat
lastChild: das letzte Child-Device, das einen occupied Wert im aktuellen Device gesetzt hat - bei mehrstufigen Parent-Child-Hierarchien wird das hier das unterste Device (das Blatt des Baumes) berichtet.
doorN: bei mehreren Türen je ein reading pro Tür mit dem jeweiligen Status (open oder closed)

Anmerkung: Die grundsätzliche Idee ist zwar mit Bewegungssensoren und Türkontakten zu arbeiten, Anwesenheit kann aber auch durch irgendein Event in der Zone ausgelöst werden, also z.B. wenn ein Schalter gedrückt wird, ist ziemlich sicher jemand da. Ebenso kann ein Raum "geschlossen" werden, ohne dass ein Türkontakt (oder gar eine Türe) vorhanden ist. Die Zone "Fernsehecke" wird bei mir "geschlossen", wenn der Fernseher angeht.

Todo/Backlog
* Commandref
* Code Cleanup
* Unterschiedliche Kommandos abhängig von der Tageszeit (z.B.: hz_cmd_likely_evening)
* "Probably Asscociated with" sollte alle Devices, die als trigger dienen oder in Kommandos genutzt werden auflisten (erledigt)
* "Do Always"-Attribut: Kommandos werden bei jedem Trigger-Event ausgelöst, nicht nur bei Status-Änderung
* Bug fixes   