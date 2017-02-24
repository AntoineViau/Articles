Supposons que nous voulions un service pour encapsuler des appels de $http. 
Un exemple simple serait d'ajouter un paramètre à l'url de la requête.
Le premier bout de code qui nous vient à l'esprit serait donc ceci :  
angular.module("monModule").factory("monService", function($http) {

	    var _addAuthenticationToUrl = function (url) {
            var newUrl = url + "?token=" + loginService.getLoggedUserToken();
            return newUrl;
        };

        var _request = function (method, url, data) {
            return $http({
                method: method,
                url: _addAuthenticationToUrl(url),
                data: data
            });
        };

        return {
            get: function (url, data) {
                return _request('GET', url, data);
            }
	etc. 
       };
    }]);
});

Comme on le voit, notre service renvoit directement la promise $http.
Son usage sera donc :
monService.get(...).success(function(data)  { ... });

Supposons à présent que nous voulions traiter les données issues de notre service, et ce sur plusieurs niveaux : 
monService.get(...)
.success(function(data) {
	data = "myData = "+data;
	return data;
})
.then(function(data) {
	console.log(data);
});

Et là c'est le drame. Alors qu'on attend l'affichage de "myData = ..." on obtient l'affichage de data seulement.
Le problème vient de $http.success() qui retourne la promise issue de l'appel de $http. 

Plusieurs solutions sont possibles. On peut commencer par encapsuler l'appel $http (ou tout service retournant une promise $http) par une promise "faite maison" :
function getData() {
	deferred = $q.defer();
	monService.get(...).
	success(function(data) {
		data = "myData = " + data;
		return deferred.resolve(data);
	});
	return deferred.promise;
}
getData()
.then(function(data) {
	console.log(data);
});

Ca fonctionne mais c'est plutôt lourd. 
On peut d'ores et déjà alléger la chose en plaçant ce code directement au sein de monService.
Notre service retourne alors une vraie promise, plutôt qu'une promise spécifique à $http.

Mais on peut faire mieux. 
$http retourne en fait une promise étendue. Elle contient success(), error(), notify()... Mais aussi then() !
En effet, on peut faire un équivalent du success/error avec un then à double callback : 
$http.get(...)
.then( function() {
	// success
},
function() {
	// error
});

Partant de ce principe, on peut revoir monService de cette façon : 
angular.module("monModule").factory("monService", function($http) {
	...
        var _request = function (method, url, data) {
            return $http({
                method: method,
                url: _addAuthenticationToUrl(url),
                data: data
            })
		.then(function(data) {
			return data;
		});
        };
	...
    }]);
});

Et obtenir le code client suivant : 
monService.get(...)
.then(function(data) {
	data = "myData = " + data;
	return data;
})
.then(function(data) {
	console.log(data);
});


Concusion : 
Au moment de concevoir nos services, si ceux doivent faire des appels distants via $http, il faut choisir une bonne fois pour toute ce qui sera retourné.
success/error ont l'avantage de la clarté mais viennent compliquer le chaînage.
L'usage du then() assure une plus grande cohérence et un chaînage implicite.

