# Uppdatering och tillbakarullning i molnet

Så här såg en uppdatering till en ny version och en tillbakarullning ut i kursklustret (namnrymden
`orhan-ulusoy`). Strategin är rolling update, och motiveringen står i README under *Utrullningsstrategi*.

## 1. Första versionen

Körningen på `main` för commit `b5420ce` rullade ut version 1 via pipelinen, efter godkännande i miljön
`produktion`:

```text
$ kubectl get pods
NAME                      READY   STATUS    RESTARTS   AGE
saldes-74c8d4895b-6xgz2   1/1     Running   0          86s
saldes-74c8d4895b-rrjfv   1/1     Running   0          86s

$ kubectl rollout history deployment/saldes
REVISION  CHANGE-CAUSE
1         deploy b5420cefe91372bdc0290f431a65dfdc29123a08
```

## 2. Uppdatering till en ny version

En ny commit på `main` (`9226cf1`) byggdes om till en image, provades i kind och rullades ut i molnet efter
godkännande. Kubernetes startade nya pods först och tog bort de gamla när de nya var redo:

```text
$ kubectl rollout history deployment/saldes
REVISION  CHANGE-CAUSE
1         deploy b5420cefe91372bdc0290f431a65dfdc29123a08
2         deploy 9226cf1f0b162513f88ad3c21460e2aed1ced375

$ kubectl get pods
NAME                      READY   STATUS    RESTARTS   AGE
saldes-55c596644c-7qgp9   1/1     Running   0          2m20s
saldes-55c596644c-k4gpt   1/1     Running   0          115s
```

Podarnas ReplicaSet bytte namn (`74c8d4895b` till `55c596644c`), alltså kör de nu den nya versionen.

## 3. Tillbakarullning

```text
$ kubectl rollout undo deployment/saldes
deployment.apps/saldes rolled back

$ kubectl rollout status deployment/saldes
Waiting for deployment "saldes" rollout to finish: 1 out of 2 new replicas have been updated...
Waiting for deployment "saldes" rollout to finish: 1 old replicas are pending termination...
deployment "saldes" successfully rolled out

$ kubectl rollout history deployment/saldes
REVISION  CHANGE-CAUSE
2         deploy 9226cf1f0b162513f88ad3c21460e2aed1ced375
3         deploy b5420cefe91372bdc0290f431a65dfdc29123a08

$ kubectl get pods
NAME                      READY   STATUS    RESTARTS   AGE
saldes-74c8d4895b-hghtk   1/1     Running   0          21s
saldes-74c8d4895b-trzsq   1/1     Running   0          35s
```

Det gick bra av tre skäl:
- Poddarna är tillbaka i `74c8d4895b`, samma ReplicaSet som version 1. Det är den gamla versionen och dess modell som kör igen.
- Revision 1 heter nu revision 3. Kubernetes räknar en tillbakarullning som en ny revision.
- Tjänsten svarade hela tiden. Det fanns alltid minst två fungerande pods eftersom `maxUnavailable` är 0.

## Vad som hände före det här

Första försöket i molnet fastnade på 1/3 pods, och det är värt att kunna berätta om:

1. Kursens namnrymd har en kvot på 384 Mi minne i `requests`, och varje pod bad om 256 Mi.
2. Jag sänkte molnmiljön till 2 repliker med 128 Mi (`k8s/moln/kustomization.yaml`).
3. Den gamla podden på 256 Mi låg kvar och blockerade kvoten. Med `maxUnavailable: 0` kunde varken gammal
   eller ny version gå vidare, så deploymenten togs bort en gång (`kubectl delete deployment saldes`) och
   skapades på nytt av pipelinen.

## Att tänka på

Tillbakarullningen gjordes för hand med `kubectl`. Klustret kör därför en äldre version än `main` tills nästa
deploy från pipelinen. Vill man att en tillbakarullning ska stå kvar bör man återställa commiten på `main`.
Annars rullar nästa deploy framåt igen.
