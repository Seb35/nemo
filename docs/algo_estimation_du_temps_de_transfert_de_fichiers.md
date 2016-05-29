Amélioration de l’estimation du temps de transfert lors de la copie de fichiers
===============================================================================

Cette méthode tente d’améliorer les estimations des temps de transfert lors de la copie de fichiers.
Il est supposé (dans un premier temps) que le transfert est relativement fiable (copie entre disques durs et/ou USB, mais pas via le réseau, moins fiable).

Globalement, l’idée directrice est de décomposer le temps de transfert en un temps fixe d’initialisation de la copie d’un fichier (temps t0) et un temps variable et proportionnel à la taille du fichier (S/v0 avec S la taille du fichier et v0 la vitesse).
t0 et v0 sont supposés constants, et le temps global de transfert d’un fichier et la taille d’un fichier sont supposés être des variables aléatoires dont on va calculer les statistiques (moyenne et variance) pour en déduire t0 et v0.

~ Seb35
Pour me contacter :
Twitter: @sseb35
Site: www.seb35.fr

Maths
-----

Soit t_0 le temps d’initialisation et de finalisation d’un transfert (supposé constant pour tous les fichiers),
     v_0 la vitesse de transfert des données brutes hors initialisation et finalisation (supposée constante pour tous les fichiers),
     N le nombre de fichiers transmis,
     S(n)_{n=1,\ldots,N} la taille de chaque fichier,
     T(N,S) le temps total de transfert pour N fichiers de tailles S(n).

Pour simplifier les expressions, posons w_0 = 1/v_0.

Nous avons comme données N, S(n)_{n=1,\ldots,N} et T(N,S) et nous recherchons t_0 et w_0, que nous supposons constants pour l’ensemble des fichiers transférés. Dans la pratique, cette hypothèse de travail sera vraie dans le cas de transferts fiables, par exemple le transfert de fichiers entre systèmes de fichiers locaux, mais elle trouvera ses limites pour les transferts entre machines distantes reliés par un réseau plus ou moins fiable et le modèle devrait être affiné dans ce cas.

Le temps total se décompose en :
T(N,S) = N t_0 + \Sigma_{n=1}^N S(n)/v_0
T(N,S) = N t_0 + \Sigma_{n=1}^N S(n) w_0

Les tailles des fichiers S(n)_{n=1,\ldots,N} sont indépendantes et identiquements distribuées selon une variable aléatoire que l’on notera S dans la suite.

Soit t_1 la variable aléatoire du temps de transfert d’un fichier.
Nous pouvons modéliser cette variable aléatoire par la formule : t_1 = t_0 + S w_0
Son espérance est : E[t_1] = t_0 + E[S] w_0
Sa variance est : Var(t_1) = Var(S) w_0^2

Pour N fichiers, les temps de transfert étant des variables aléatoires indépendantes et identiquement distribuées, nous pouvons considérer ceci comme N réalisations de la variable aléatoire t_1 :
E[t_1] = t_0 + w_0 E[S]
Var(t_1) = Var(S w_0) = w_0^2 Var(S)

Si Var(S) = 0, il n’y a pas besoin d’isoler t_0 et w_0.
Sinon, nous obtenons de la variance précédente :
w_0 = \sqrt{\frac{Var(t_1)}{Var(S)}
Puis :
t_0 = \frac{T(N,S) - w_0 E[S]}{N}


Algorithme
----------

À partir de ces deux dernières formules, nous pouvons dériver l’algorithme :

 Nombre de fichiers := N
 Temps passé := T
 Moyenne de la taille des fichiers := SE
 Variance de la taille des fichiers := SV
 Moyenne du temps de transfert := t1E
 Variance du temps de transfert := t1V
 Inverse de la vitesse de transfert hors initialisation et finalisation := v0
 
 Initialisation :
  N = 0
  T = 0
  SE = 0
  SV = 0
  TE = 0
  TV = 0
 À chaque finalisation du transfert d’un fichier, de taille S :
  Incrémenter le nombre de fichiers N = N+1
  Obtenir le temps de transfert du fichier t1 = temps depuis le début du transfert - T
  Calculer la moyenne t1E et la variance t1V du temps de transfert selon les formules itératives (*)
  Calculer la moyenne SE et la variance SV de la taille des fichiers selon les formules itératives (*)
  Calculer w0 = sqrt(t1V/SV)
  Calculer t0 = (T - SE x w0)/N
  À partir de ces données t0 et w0, nous pouvons obtenir une estimation du temps restant prenant en compte le nombre de fichiers restants et le volume à transférer restant :
   temps restant = nombre de fichiers restants x t0 + volume à transférer restant x w0

Le temps restant prend donc en compte le nombre de fichiers et étant donné que le temps d’initialisation et finalisation n’est pas négligeable lorsque le nombre de fichiers est important, ce temps restant est donc plus fiable qu’une estimation naïve négligeant le temps d’initialisation.

La vitesse de transfert affichée peut prendre en compte ou non le temps d’initialisation, c’est juste une histoire de définition. Une vitesse négligeant le temps d’initialisation sera plus volatile qu’une vitesse le prenant en compte, ce qui donne l’impression à l’utilisateur que la vitesse de transfert varie alors que celle-ci ne varie pas (forcément) au sens strict, c’est juste que le nombre de fichiers transféré pendant une période de temps est variable.

Implémentation
--------------

L’algorithme a été testé avec l’explorateur de fichiers Nemo sous Linux Mint LMDE.

https://github.com/seb35
Ce dépôt est une version de développement de Nemo, en particulier l’interface de copie de fichiers comprend de nombreuses variables intermédiaires de l’algorithme.

(*) Voir https://en.wikipedia.org/wiki/Algorithms_for_calculating_variance#Online_algorithm notamment l’algoritme online_variance(data)
Dans le cas où plusieurs échantillons (fichiers) sont ajoutés en même temps, cela devient :
    def online_variance(data):
        n = 0
        mean = 0.0
        M2 = 0.0
        
        for (x,m) in data:
            n += m
            delta = x - mean
            mean += delta*m/n
            M2 += delta*(x - mean)*m
        
        if n < 2:
            return float('nan')
        else:
            return M2 / (n - 1)

Détails de l’implémentation :
* Les variances ne sont pas directement calculées mais seulement les sommes des carrés centrés par leurs moyennes
* L’unité de temps est la microseconde, l’unité de taille est l’octet
* Si le temps d’initialisation est inférieur à 10µs (voire négatif), celui-ci est fixé à ce minimum de 10µs et l’inverse de la vitesse est recalculé en fonction de cela
* Dans le cas où plusieurs fichiers sont ajoutés en même temps, cela ne pose pas de problème moyennant un coefficient du nombre de fichiers
* Dans le cas où le fichier en copie est le même que lors du dernier appel, la moyenne est corrigée pour ajouter "ce bout supplémentaire de fichier" et la somme des carrés est amputée de l’ajout du dernier appel (mais il faudrait bien recalculer les quantités, je pense que ça n’est pas bon actuellement, il faudrait que les quantités pour les carrés prennent en compte l’entièreté du fichier et pas seulement le bout ajouté – mais la moyenne est normalement correcte)

Résultats
---------

Après plusieurs tests, il en ressort les conclusions suivantes :
* le temps d’initialisation est parfois calculé comme étant négatif, surtout lors de la copie de nombreux petits fichiers : peut-être que le système fait des copies en batch, ce qui ferait qu’il n’y a qu’une initialisation pour plusieurs fichiers
* la vitesse (interne) de transfert est généralement supérieure à la vitesse calculée par la méthode classique ; cela semble logique cas le temps d’initialisation est déduit ; quand cette vitesse interne est inférieure à la vitesse de la méthode classique, je crois que c’est corrélé aux temps d’initialisation négatif, il faudrait revoir le modèle pour essayer de comprendre si c’est un problème numérique ou alors dans le modèle lui-même
* cette méthode est beaucoup plus réactive aux changements liés à la répartition des tailles des fichiers : pour un transfert commençant par 39000 petits fichiers puis se terminant par 90 gros fichiers, cette méthode détecte rapidement la transition et la vitesse passe de 1Mio/s à 50Mio/s, et le temps de transfert passe rapidement de 3h à 1 minute, contrairement à la méthode classique qui rattrape lentement en partant également de 3h.
* dans le cas de fichiers de tailles similaires, les deux méthodes sont globalement équivalentes

Améliorations envisagées
------------------------

Dans le cas du transfert de 39000 petits fichiers puis 90 gros fichiers, les deux méthodes commencent par afficher une estimation de 3h ou 4h (le temps réel est de 5 minutes) ; c’est logique pour la méthode classique car elle s’appuie sur le volume des fichiers ; pour cette méthode, on aurait pu s’attendre à ce qu’elle donne une estimation plus juste, mais elle est toutefois limitée par la vitesse de transfert calculée (entre 0,7 et 1,3 Mio/s) : la vitesse de transfert pour les petits fichiers ne décolle pas les transferts sont sans cesse réinitialisés, contrairement aux gros fichiers (vitesse 50 Mio/s).

Il faudrait un mécanisme supplémentaire pour détecter cette disproportion et anticiper que le taux va augmenter pour les plus gros fichiers. Les questions sont :
* quel ratio calculer pour détecter cette disproportion entre les variables temps, tailles, et nombre de fichiers
* dans quelle proportion est-il raisonnable d’envisager une accélération (resp. décellération) pour les gros (resp. petits) fichiers puisque cela dépend du type de technologie utilisée pour le transfert

Interface utilisateur
---------------------

La méthode classique affiche un pourcentage en fonction du volume, une estimation de temps, et une vitesse moyenne depuis le début de la copie, et ce actualisé toutes les 100ms.

Je verrais bien une diminution de la fréquence d’actualisation pour l’estimation de temps, toujours supérieure à 1s et éventuellement une fréquence logarithmiquement proportionnelle au temps restant estimé (par exemple si Test=5min => freq=10s; Test=20min => freq=1min; Test=1h => freq=5min, etc.), avec une possibilité de saut en cas de changement brutal de l’estimation (par exemple de 3h à 2min), mais avec une fréquence de saut jamais inférieure à 30s par exemple.

La barre de progression devrait être globalement liée au temps restant estimé, c’est-à-dire une pondération entre le nombre de fichiers et le volume, mais avec la contrainte de ne jamais revenir en arrière même en cas d’erreur d’estimation. La difficulté est de trouver le coefficient de pondération entre les deux quantités. Peut-être celui-ci peut-il être fixé en fonction du rapport volume/nombre, par exemple alpha*arctan(beta*volume/nombre) en fixant alpha et beta après expérimentations pour trouver un seuil où la taille des fichiers devient prépondérante sur le volume.

