# Le Petit Prince - Repétition

Notes techniques pour préserver les comportements de l'application LPP.

## Architecture

- Application autonome dans `index.html` : React 18 UMD, Babel et Tailwind chargés par CDN; aucun build local.
- Le texte complet est lu depuis `LEPETITPRINCE_Theatre.txt` avec `fetch('./LEPETITPRINCE_Theatre.txt')`.
- Servir le dossier en HTTP/HTTPS. Si le navigateur bloque `fetch` lors d'une ouverture locale, le sélecteur de fichier permet de charger le texte manuellement.
- GitHub Pages publie le dépôt `fredplaine/lpp` depuis `main`, dossier racine. URL : `https://fredplaine.github.io/lpp/`.

## Chargement des scènes

- `loadFullScript(source)` repère les titres avec `/^#\s*SCÈNE\s+(.+)$/gim`. Les deux formes `# SCÈNE` et `#SCÈNE` sont acceptées.
- Le parseur exige exactement 26 titres; sinon il active le message de chargement manuel.
- Les répliques des 26 scènes proviennent exclusivement de `LEPETITPRINCE_Theatre.txt`. Il n'y a pas de copie de secours des répliques dans le code.
- Les lignes non reconnues comme locuteur sont ajoutées à la réplique précédente. Avant la première réplique, elles sont gardées dans `pendingText`, puis ajoutées à celle-ci. Les didascalies ne deviennent donc pas un personnage.
- Tout texte entre parenthèses est supprimé du texte chargé, y compris les parenthèses imbriquées et les didascalies réparties sur plusieurs lignes; il n'est ni affiché ni prononcé.
- Toute modification du nombre ou du format des titres de scène doit être vérifiée avec le parseur et le chargement de la page.

## Personnages

- `speakers` dans `loadFullScript` définit les préfixes reconnus comme locuteurs; la liste est triée du plus long au plus court pour éviter les correspondances prématurées.
- `normalizeCharacter(character)` met les noms en majuscules, uniformise les apostrophes et regroupe les variantes. Cette fonction est utilisée pour le menu, les préférences, l'affichage et `isMyRole`.
- Alias actuels à préserver : `AVIATEUR` / `L’AVIATEUR` -> `L’AVIATEUR`; `ROSE` -> `ROSES`; `RÉVERBÈRES` -> `L’ALLUMEUR DE RÉVERBÈRES`; `HOMME D’AFFAIRES A` -> `HOMME D’AFFAIRES`; `LE PETIT PRINCE` -> `PETIT PRINCE`.
- Dans la réplique `HOMME D'AFFAIRES A acheter...`, le locuteur est `HOMME D'AFFAIRES`; le premier `A` appartient au dialogue. Ne pas ajouter `HOMME D'AFFAIRES A` comme locuteur.
- Pour ajouter un nouvel alias, mettre à jour la liste `speakers` si nécessaire, puis `normalizeCharacter`; vérifier qu'il n'ajoute pas une seconde option au menu.

## Navigation et menu

- `currentSection` et `currentLine` sont des index commençant à zéro.
- Le menu affiche les 26 scènes en chiffres romains via `sceneRomanNumerals` : I à XXVI. Chaque bouton garde un nom accessible `Scène n` et un `title` descriptif.
- Grille actuelle : 7 colonnes sur mobile, 9 dès `sm`, 13 dès `lg`; boutons d'au moins 34 px.
- `handleNext`, `handlePrevious` et `handleReset` arrêtent la lecture vocale et remettent l'état de révélation à zéro avant de naviguer.
- Le swipe est horizontal si le déplacement dépasse 80 px et deux fois le déplacement vertical. Gauche = suivant; droite = précédent.

## Lecture vocale

- La lecture automatique utilise `speakWithPunctuationPauses`; les virgules, points-virgules et deux-points utilisent `commaPause`, les points, points d'exclamation et d'interrogation utilisent `sentencePause`.
- Les didascalies entre parenthèses ne sont pas prononcées. Les voix sont choisies dans les groupes `FEMMES` et `HOMMES`; `gender` détermine voix et hauteur.
- `speakPresidentLine`, `presidentSpeechState` et les refs `president*` gardent des noms historiques, mais agissent sur le rôle sélectionné, quel qu'il soit.
- État de lecture : `idle` démarre, `playing` annule la synthèse et sauvegarde l'index du morceau, `paused` reprend depuis cet index. Ne pas s'appuyer sur `speechSynthesis.pause()`.
- `handleTextTap` : un double-tap de moins de 300 ms revient au premier morceau et laisse la lecture en pause; le tap simple suivant reprend.
- Lors d'une navigation ou d'un changement de rôle, annuler la synthèse, vider `presidentTimeoutRef` et réinitialiser `presidentSpeechState`.

## Mode Turbo

- `isMyRole(line)` compare les noms canoniques, pas les graphies brutes.
- Le mode Turbo saute les répliques qui ne sont ni le rôle sélectionné ni la réplique qui le précède.
- `findCueLineIndex(sectionIdx, fromLine)` renvoie l'index de la réplique précédant la prochaine ligne du rôle, ou `null` si aucune n'existe.
- La cue line est lue avec `getLastSentence(text)`, qui conserve les deux dernières phrases; la réplique du rôle est ensuite lue normalement.
- Préserver le passage à la section suivante et l'arrêt en fin de pièce quand aucune cue line ne reste.

## Souffleur

- Le souffleur ne s'applique qu'à une réplique correspondant à `isMyRole(currentLineData)`.
- États : `hidden` (compte à rebours), `hint` (sept premiers mots), `full` (texte complet), `audio` (lecture).
- `prompterDelay` règle le délai; le timer est nettoyé quand la réplique ou l'état change.
- Clics : `hidden`/`hint` -> `full` -> `audio` -> `full`.

## Préférences et marque-pages

- Les préférences sont stockées dans `localStorage` sous `femmesTRAE_settings`; la sélection de rôle y est normalisée au chargement.
- Les marque-pages sont stockés sous `femmesTRAE_bookmarks`, avec une clé `${currentSection}-${currentLine}`.
- Un marque-page conserve les index de scène et de ligne, le personnage canonique, un extrait, la date et la catégorie.
- Ces clés portent un ancien préfixe FEC, mais sont encore utilisées par LPP. Les renommer nécessite une migration des données existantes.

## Vérifications avant publication

- Le chargement HTTP affiche 26 scènes et les boutons I à XXVI.
- Tester la grille sur mobile étroit : 7 colonnes, aucune barre de défilement horizontale, numéros contenus dans les boutons.
- Vérifier le menu des rôles : aucune entrée en double pour l'Allumeur, les Roses, l'Aviateur ou l'Homme d'affaires.
- Vérifier le changement de rôle sur une réplique, la révélation, lecture/pause/reprise, Turbo, souffleur et navigation.
- Publier uniquement les fichiers LPP nécessaires; ne pas ajouter `fec.html` ni les autres fichiers FEC au commit.