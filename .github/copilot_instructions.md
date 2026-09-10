# CONTEXTE & RÈGLES D'ARCHITECTURE PROJET : HEXXAGON / ATAXX WEB

## 1. Vue d'ensemble du projet
Jeu de stratégie au tour par tour sur grille hexagonale inspiré d'Ataxx/Hexxagon.
- **Frontend :** React (TypeScript, Vite), composants SVG purs pour le rendu géométrique.
- **Backend :** Google Cloud Platform (GCP) avec Cloud Functions (Node.js/TypeScript, Gen 2).
- **Persistance & Synchronisation :** Firebase Firestore (mode natif).
- **Contrainte budgétaire :** 100 % éligible au quota gratuit permanent GCP (*Always Free*).

---

## 2. Règles du jeu & Spécifications métier
- **Plateau :** Grille hexagonale fermée de rayon $R$ (par défaut 3 ou 4), orientation *pointy-topped*.
- **Déplacements autorisés :**
  - **Duplication (distance = 1) :** Le pion source reste en place, un nouveau pion apparaît sur la case cible vide (+1 pion).
  - **Saut (distance = 2) :** Le pion source quitte sa case d'origine et atterrit sur la case cible vide (+0 pion).
  - Distance > 2 ou case occupée : Mouvement illégal.
- **Capture (Contamination) :**
  - Dès qu'un pion arrive sur une case, tous les pions adverses situés à distance = 1 de cette case changent de camp et prennent la couleur du joueur actif.
- **Fin de partie :**
  - Le plateau est complet, OU un joueur n'a plus aucun pion, OU plus aucun coup n'est jouable.
  - Vainqueur : Joueur avec le plus de pions.

---

## 3. Système de coordonnées hexagonales
Le projet utilise strictement les **coordonnées axiales `(q, r)`** pour le stockage et **cubiques `(q, r, s)`** pour les calculs :
- Contrainte fondamentale : `s = -q - r`
- Clé de stockage Firestore : chaîne `"${q},${r}"` (ex: `"0,0"`, `"-2,1"`).
- Distance axiale :
  `distance(A, B) = (abs(A.q - B.q) + abs(A.r - B.r) + abs((-A.q - A.r) - (-B.q - B.r))) / 2`
- Voisins immédiats (distance = 1) :
  `[(1, 0), (1, -1), (0, -1), (-1, 0), (-1, 1), (0, 1)]`
- Projection écran (SVG Pointy-topped, rayon R) :
  - `x = R * sqrt(3) * (q + r / 2)`
  - `y = R * (3 / 2) * r`

---

## 4. Modèle Firestore & Optimisation des coûts
Pour respecter le quota gratuit (50 000 lectures / 20 000 écritures par jour) :
- **Règle absolue : 1 partie = 1 document Firestore unique** sous le chemin `games/{gameId}`.
- Aucun sous-document par case ou par coup.
- Le plateau est stocké sous forme de dictionnaire creux (seules les cases occupées sont stockées).

### Schéma TypeScript d'un document de partie :
```typescript
interface GameDocument {
  meta: {
    createdAt: string;
    updatedAt: string;
    boardRadius: number;
    status: 'IN_PROGRESS' | 'FINISHED';
    winner: string | null;
  };
  players: {
    p1: { uid: string; displayName: string; color: string; isBot: boolean };
    p2: { uid: string; displayName: string; color: string; isBot: boolean };
  };
  turn: {
    activePlayerId: string;
    turnNumber: number;
    deadline?: string;
  };
  score: {
    p1: number;
    p2: number;
  };
  board: Record<string, string>; // clé "q,r" -> playerId
  lastMove?: {
    playerId: string;
    type: 'DUPLICATE' | 'JUMP';
    from: { q: number; r: number };
    to: { q: number; r: number };
    captured: string[]; // liste des clés "q,r" assimilées
  };
}