# Driftplan

Underlag till den muntliga genomgången. Ungefär fem minuter, i den här ordningen.

## 1. Hur kommer en ändring ut?

1. Pull request granskas av en teamkamrat och slås ihop till `main`.
2. Pipelinen testar, kontrollerar datan, tränar, utvärderar och bygger imagen. Imagen taggas med
   commitens SHA och pushas till ghcr.io.
3. `rulla ut i kind` provar manifesten och imagen i ett tillfälligt kluster.
4. `rulla ut i molnet` väntar på att någon godkänner. Efter godkännandet rullas versionen ut med
   `RollingUpdate`.

Ingen kan alltså nå molnet utan att ha gått via `main`, testerna, kvalitetsgrinden och ett godkännande.

## 2. Vad händer när en utrullning går fel?

- `maxUnavailable: 0` betyder att en gammal pod aldrig tas bort innan en ny är redo. Är den nya imagen
  trasig blir den nya podden aldrig redo och de gamla fortsätter ta emot trafik.
- Blir utrullningen inte klar inom 180 sekunder kör pipelinen `kubectl rollout undo` och jobbet blir rött.
- För hand: `kubectl rollout undo deployment/saldes`, eller `--to-revision=N` efter
  `kubectl rollout history deployment/saldes`. Varje revision har en `change-cause` med commitens SHA.

## 3. Vad gör vi om modellen är dålig fast utrullningen gick bra?

Poddarna kan vara friska fast modellen gissar dåligt. Då går vi tillbaka till en tidigare commit:

1. Hitta förra bra versionen i `kubectl rollout history`, eller i Actions.
2. `kubectl rollout undo` ger tillbaka den direkt.
3. Spåra orsaken: SHA i `/health` och `matvarden.json` visar commit, datafil och kontrollsumma.
4. Rätta i en pull request, och låt pipelinen rulla ut den nya versionen.

## 4. Hur ser vi att det fungerar?

- `readinessProbe` och `livenessProbe` mot `/health`. Kubernetes startar om en pod som slutar svara.
- `kubectl get pods -l app=saldes` och `kubectl logs deployment/saldes`.
- `curl /health` visar vilken commit och vilken ROC AUC som körs just nu.

## 5. Kapacitet

- 3 repliker i molnet, vardera med `requests` 100m CPU och 256 Mi minne och `limits` 500m och 512 Mi.
- Fler repliker: ändra `replicas` i manifestet och rulla ut, eller `kubectl scale deployment/saldes --replicas=N`
  för att prova något tillfälligt.

## 6. Hemligheter och åtkomst

- Enda hemligheten är `KUBECONFIG` i GitHub-miljön `produktion`. Den ligger inte i repot.
- Bara granskarna i miljön kan godkänna en utrullning, och bara från `main`.
- Imagen kör som vanlig användare (inte root).

## 7. Det här har vi inte (ännu)

Det här är ärligare att säga själva än att bli ställda inför:

- **Ingen övervakning eller larm.** Vi märker ett fel först när någon tittar eller när en probe fallerar.
  Nästa steg är mätvärden (till exempel Prometheus) och ett larm på antal pods som inte är redo.
- **Ingen automatisk koll av modellens kvalitet efter utrullning.** Kvalitetsgrinden körs före, inte efter.
- **Samma tagg kan byggas om.** Den schemalagda måndagskörningen tränar om modellen på samma commit, så
  taggen (SHA) kan peka på en ny modell. Därför rullas inget ut från schemat. En bättre lösning är att
  lägga körnumret i taggen.
- **Inga resurskvoter, nätverksregler eller `PodDisruptionBudget`.**
- **Modell och data ligger i imagen.** Det gör tjänsten enkel att rulla tillbaka, men en ny modell eller
  dataleverans kräver en ny image och en ny utrullning.
