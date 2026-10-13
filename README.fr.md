# vegas-scripting-notes

**Les pièges du scripting VEGAS Pro qui ne sont pas dans la documentation.**
Cinq pièges vérifiés — le code qui échoue, l'échec mesuré, le correctif — et la
méthode pour vider la vraie surface d'API au lieu de la deviner.

[English version →](README.md)

---

## Pourquoi ce dépôt

L'API `ScriptPortal.Vegas` est un assembly .NET livré sans documentation de
référence utilisable. Le modèle objet est découvrable, mais le **comportement**
ne l'est pas. On obtient donc des échecs qui ressemblent à du code faux alors
que l'API a accepté le code et **ignoré la valeur en silence**.

Trois de ces cinq pièges ont coûté du temps réel en production. L'un d'eux a
produit **un résultat faux sur 75 % des images** alors que chaque ligne de code
était correcte.

C'est écrit pour ceux qui doivent écrire un script VEGAS et qui préfèrent ne pas
tout redécouvrir.

---

## Les pièges en un tableau

| # | Piège | Échec observé | Correctif |
| --- | --- | --- | --- |
| 1 | Régler `VideoMotionKeyframe.Bounds` **avant** `Keyframes.Add()` | **72 / 288** images animées | Ajouter d'abord, régler ensuite |
| 2 | Unités de `Timecode` | valeur fausse d'un facteur 10 000 | `new Timecode(ms)` prend des **millisecondes** ; `Nanos / 10000.0` donne des ms |
| 3 | Espace de coordonnées de `VideoMotionBounds` | la fenêtre de recadrage tombe au mauvais endroit | coordonnées en **pixels source**, normalisées sur 1920 de large |
| 4 | `VideoMotion.ScaleToFill` | bandes noires dans l'aperçu | ce n'est **pas** « faire correspondre le format de sortie » — il faut ce booléen **et** `Bounds` |
| 5 | `TrackEvent.Split(t)` | coupe au mauvais endroit | `t` est un **décalage depuis le début de l'événement**, pas une position absolue |

Détail, avec le code qui échoue et celui qui marche :
**[docs/API-NOTES.md](docs/API-NOTES.md)** (en anglais).

---

## Le piège n°1, en entier

C'est celui qui produisait un résultat faux en ayant l'air correct.

```csharp
// FAUX — 72 images animées sur 288
var k = new VideoMotionKeyframe(project, ev.Start);
k.Bounds = new VideoMotionBounds(tl, tr, br, bl);   // réglé AVANT l'insertion
motion.Keyframes.Add(k);                            // Vegas écrase la valeur
```

```csharp
// JUSTE — 288 sur 288
var k = new VideoMotionKeyframe(project, ev.Start);
motion.Keyframes.Add(k);                            // l'objet EXISTE dans son conteneur
k.Bounds = new VideoMotionBounds(tl, tr, br, bl);   // on règle ensuite
```

**Pourquoi.** Vegas calcule une valeur par défaut **au moment de l'insertion**,
et cette valeur par défaut écrase ce qu'on avait mis avant. L'objet doit
**exister dans son conteneur** avant d'être configuré.

**La règle générale, qui vaut plus que le correctif :** toute API qui vous donne
un `Add` ou une fabrique vous dit que c'est le **conteneur** qui possède l'état
initial. Configurez après, pas avant.

**Comment ça a été trouvé.** Pas en lisant le code — c'était du C# correct. En
**relisant la sortie** : compter combien d'événements portaient réellement des
images clés à la fin, et comparer au nombre écrit. 72 contre 288.

C'est la vraie leçon : **un script qui écrit doit être accompagné d'un script
qui relit.** « Le script a tourné » est une déclaration, pas une preuve.

---

## Le piège n°4 — celui dont le nom trompe

`ScaleToFill` ressemble à l'option « Faire correspondre le format de sortie » de
l'interface. Ce n'en est pas une.

```
Bounds       = QUELLE PARTIE de l'image on voit
ScaleToFill  = si cette partie est AGRANDIE pour remplir le cadre
LES DEUX SONT NÉCESSAIRES.
```

Avec `Bounds` correct et `ScaleToFill` à `false`, la partie rognée garde sa
taille et l'aperçu montre des **bandes noires** — ce qui se lit « mes bornes sont
fausses » alors que les bornes sont bonnes.

Quand un nom est ambigu, **tranchez par réflexion en une ligne** plutôt que de
discuter du vocabulaire (voir [docs/REFLECTION.md](docs/REFLECTION.md)).

---

## Ce que j'ai mesuré

| Affirmation | Comment c'est mesuré | Résultat |
| --- | --- | --- |
| Régler avant d'ajouter est ignoré | comptage des événements portant des images clés après exécution | **72 / 288** mauvais ordre · **288 / 288** bon ordre |
| Un `.cs` sans son `.dll` ne fait rien | script lancé avant compilation | absent du menu Scripts, aucune erreur |
| `new Timecode(4000.0)` est en millisecondes | aller-retour : construire puis `ToMilliseconds()` | **4000** renvoyé, pas 0,0004 |
| `ScaleToFill` est un booléen local | réflexion sur l'assembly installé | `Property Boolean ScaleToFill {get;set;}` |

### Une correction que je dois faire

Une version antérieure de mes notes affirmait que le filtre `acrossfade` de
`ffmpeg` **« raccourcit chaque jonction de 30 ms à cause de son défaut
`overlap=1` »**, et que cela avait coûté 28,29 s sur 943 jonctions.

**J'ai remesuré : cette explication est fausse.**

`acrossfade=d=D` raccourcit le total d'**exactement `D` par jonction** par
rapport à une simple concaténation. C'est la signification documentée de la
durée de fondu, pas un défaut caché, et c'est **identique avec `overlap=0` et
avec le défaut `overlap=1`**. Mesuré sur ffmpeg 7.1.5 : deux sources de 2,000 s
avec `acrossfade=d=0.5` donnent 3,500 s ; `overlap=0` et `overlap=1` donnent la
même durée. Une chaîne de 400 jonctions reproduit l'arithmétique attendue avec
**zéro** dérive supplémentaire.

Le **conseil pratique survit** : quand on assemble de l'audio à partir de
nombreux segments, on vérifie la durée **de sortie** par rapport à la cible, et
on préfère la concaténation simple quand on ne veut aucun changement de durée.
C'est le **mécanisme** que j'avais publié qui était faux ; je le corrige ici au
lieu de le supprimer discrètement.

---

## Portée et honnêteté

- Tout a été observé sur **VEGAS Pro 20, build 411**, sous Windows.
- L'API est stable sur les versions récentes : la plupart de ceci devrait tenir
  pour VEGAS 18 à 22 — mais « devrait » n'est pas « vérifié », et je ne les ai
  pas testées.
- **Aucun binaire n'est livré.** Les `.dll` se compilent sur chaque machine.

## Licence

[MIT](LICENSE).
