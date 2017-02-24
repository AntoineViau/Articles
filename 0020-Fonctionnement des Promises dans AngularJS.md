<!--
--- Layout: post
--- Title: Fonctionnement des Promises AngularJs
--- Date: 2015-03-25
-->

Les promises sont un concept extrêmement pratique mais avec une courbe d'apprentissage souvent fluctuante. La première approche est généralement assez aisée, mais dès que l'on cherche à approfondir on est souvent confronté à des subtilités plus ou moins difficiles à appréhender. Pour avancer plus vite dans la compréhension des Promises il est important d'en saisir le fonctionnement interne.


Prenons un exemple classique et simplifié issu de la documentation d'AngularJs :

    function asyncGreet(name) {
       var deferred = $q.defer();

       setTimeout( function() {
           if ( okToGreet(name) ) {
               deferred.resolve('Hello, ' + name + '!');
           } 
           else {
               deferred.reject('Greeting ' + name + ' is not allowed.');
           }
       });
       , 1000);

       return deferred.promise;
    }

    var promise = asyncGreet('Robin Hood');

    promise.then( function(greeting) {
       alert('Success: ' + greeting);
    }
    ,function(reason) {
       alert('Failed: ' + reason);
    });


Quel est le sens de ce code ? La fonction asyncGreet prend du temps et ne produira un résultat qu'au bout d'une seconde. On peut imaginer à la place du setTimeout tout le code nécessaire pour faire une requête au serveur et attendre le retour de données.

**Comment fonctionne alors l'API Promise ?**

  * un gestionnaire de Promises est créé via $q.defer() ;
  * le code "à retardement" commence immédiatement ;
  * la fonction se termine en retournant une promise issue du gestionnaire ;
  * quand le code à retardement arrive à sa conclusion, il indique au gestionnaire qu'il a terminé (en succès ou en erreur).

Que se passe-t'il alors au niveau du code appelant :

  * asyncGreet a donc retourné une Promise ;
  * celle-ci comprend forcément une méthode then() ;
  * cette méthode prend une fonction qui sera stocké au moment de l'appel ;
  * quand le code à retardement arrive à conclusion et qu'il appelle le gestionnaire, ce dernier n'a plus qu'à appeler la fonction donnée à then(). C'est le boulot de resolve() (pour appeler la fonction de succès) et de reject (pour appeler la fonction d'échec).

On le voit, c'est assez simple : le gestionnaire de promises s'occupe de faire le lien entre le code appelant et le code à retardement. Ce lien se fait en interne par resolve() et reject(), et en externe par then().

**Résumé :**

  * appel de asyncGreet ;
  * création du gestionnaire ; $q.defer() ;
  * exécution du code à retardement : setTimeout() ;
  * renvoi de la promise au code appelant : return deferred.promise ;
  * l'appelant indique quelle fonction devra être appelée à la conclusion : promise.then( function() ... ) ;
  * le code à retardement arrive à conclusion et demande au gestionnaire d'appeler la fonction qui doit faire suite : $q.resolve() ou $q.reject() ;

**Quelques remarques :**

Il faut bien comprendre qu'une Promise créée implique l'appel du code "lent". D'où, d'ailleurs, le terme "Promise" : on obtient une promesse de résultat quoiqu'il arrive, qu'il soit un succès ou un échec. Il ne faut donc pas chercher à créer une Promise pour ensuite l'appeler. C'est l'appel qui crée la Promise et non l'inverse.

