# Démonstrateur ChaCha20-Poly1305

Application web pédagogique illustrant **pas à pas** le fonctionnement de l'algorithme de chiffrement authentifié **ChaCha20-Poly1305**. L'utilisateur saisit un message, et l'interface permet d'observer l'évolution de la matrice d'état de ChaCha20 round par round, le keystream généré, le XOR avec le message, ainsi que la génération du tag d'authentification Poly1305.

Projet réalisé dans le cadre d'un projet universitaire de Licence 3, en collaboration avec **Anatole Legouix** et **Titouan Louvet**.

> ℹ️ Le dépôt original a été créé par Anatole Legouix et est actuellement privé. Ce dépôt est une republication destinée à présenter le code et le fonctionnement de l'application.

## Ce que fait l'application

- **Implémentation _from scratch_** de ChaCha20 et de Poly1305 en Python — aucune dépendance cryptographique externe, l'objectif étant pédagogique
- **Visualisation de la matrice d'état 4×4** de ChaCha20 (constantes `"expand 32-byte k"`, clé, compteur, nonce)
- **Avancement manuel** des rounds via les boutons _Suivant_ / _Précédent_ / _Reset_
- **Affichage** du keystream, du message chiffré, du résultat de XOR partiel, du MAC Poly1305 et du déchiffrement
- **Découpage automatique** du message en blocs de 512 bits

> ⚠️ **À ne pas utiliser en production.** Cette implémentation est volontairement simple, lisible et instrumentée pour l'enseignement. Pour un usage réel, utilisez `cryptography` (Python) ou `libsodium`.

## Stack technique

- **Backend** : Python 3, Django 5.2
- **Frontend** : HTML, CSS
- **Cryptographie** : implémentation maison de ChaCha20 et Poly1305 (modules `polls/Chacha.py` et `polls/poly1305.py`)
- **Base de données** : SQLite (par défaut Django, non utilisée fonctionnellement ici)

## Structure du dépôt

```
D-monstrateur-Chacha20-Poly1305/
├── README.md
└── application/
    ├── manage.py
    ├── mysite/            # Configuration du projet Django
    │   ├── settings.py
    │   ├── urls.py
    │   └── wsgi.py
    └── polls/             # Application Django du démonstrateur
        ├── views.py       # Vue principale + gestion de l'état pas-à-pas
        ├── urls.py
        ├── Chacha.py      # Implémentation de ChaCha20
        ├── poly1305.py    # Implémentation de Poly1305
        └── templates/
            └── polls/
                └── index.html
```

## Prérequis

- Python 3
- Django 5.2 ou supérieur

## Installation et lancement

### 1. Cloner le dépôt

```bash
git clone https://github.com/OlsonWells/D-monstrateur-Chacha20-Poly1305.git
cd D-monstrateur-Chacha20-Poly1305
```

### 2. Créer un environnement virtuel (recommandé)

```bash
python3 -m venv dev
source dev/bin/activate          # macOS / Linux
# .\dev\Scripts\activate         # Windows
```

### 3. Installer Django

```bash
python3 -m pip install django
```

### 4. Lancer le serveur

```bash
cd application
python3 manage.py runserver
```

L'application est accessible sur **<http://127.0.0.1:8000>**.

## Utilisation

1. Saisir un message en clair dans le champ prévu et le soumettre
2. Le message est encodé en UTF-8, découpé en blocs de 512 bits (complétés par zéro si besoin) et la matrice initiale de ChaCha20 est construite
3. Avancer dans l'algorithme avec les boutons :
   - **Suivant** : exécute le prochain _quarter-round_ (8 par round, 10 rounds = 20 rounds ChaCha20)
   - **Précédent** : revient à l'état précédent
   - **Reset** : revient à l'état initial
4. Observer en temps réel :
   - les 16 mots de 32 bits de la matrice d'état
   - le keystream produit à la fin des 10 rounds
   - le XOR keystream ⊕ message → texte chiffré
   - le tag d'authentification **Poly1305** (calculé à partir d'un premier bloc ChaCha20 avec compteur = 0, conformément au principe du RFC 7539)
   - le déchiffrement obtenu en réappliquant le keystream

## Détails d'implémentation

### ChaCha20 (`polls/Chacha.py`)

- Matrice 4×4 de `c_uint32` (entiers 32 bits) :
  - Ligne 1 : constantes `0x61707865 6e642033 322d6279 7465206b` (`"expand 32-byte k"`)
  - Lignes 2–3 : clé de 256 bits
  - Ligne 4 : compteur (32 bits) + nonce (96 bits)
- Fonction **quarter-round** `QR(a, b, c, d)` implémentée à la main
- Pour chaque bloc : 10 itérations de 8 quarter-rounds (4 _column rounds_ + 4 _diagonal rounds_) = 20 rounds
- Keystream : `init_matrice + matrice_finale`, puis XOR avec le message
- Compteur incrémenté entre chaque bloc

### Poly1305 (`polls/poly1305.py`)

- Clé de 256 bits dérivée à partir d'un bloc ChaCha20 avec **compteur = 0** (les 32 premiers octets du keystream)
- Tag MAC calculé sur le message

### Architecture Django

- Une seule vue : `polls/views.py::index`
- Les états successifs sont conservés dans une variable module `previous_states` (liste d'instances `Chacha`), permettant les commandes `next`, `previous`, `reset`
- Les interactions s'effectuent en **AJAX** (header `X-Requested-With: XMLHttpRequest`) avec des réponses `JsonResponse`

## Auteurs

- Olson Wells
- Anatole Legouix
- Titouan Louvet

## Limitations connues

- Le nonce est généré avec `random.seed(0)` puis `randint` — **non sécurisé**, fait exprès pour rendre les exemples reproductibles
- La clé est constante (`[0, 1, 2, …, 7]`) pour les mêmes raisons pédagogiques
- L'état pas-à-pas est conservé dans une variable globale du module : l'application est mono-utilisateur
- Un commentaire `TODO switch endian` dans `Chacha.py` indique que la gestion de l'endianness n'est pas strictement conforme au RFC 7539
- Pas de gestion d'_Associated Data_ (AEAD) au sens complet du RFC

## Références

- [RFC 7539 — ChaCha20 and Poly1305 for IETF Protocols](https://datatracker.ietf.org/doc/html/rfc7539)
- [The ChaCha family of stream ciphers — D. J. Bernstein](https://cr.yp.to/chacha.html)
- [The Poly1305-AES message-authentication code — D. J. Bernstein](https://cr.yp.to/mac.html)
