Quand on définit un Observable, il s'agit d'une source de donnée à laquelle on ajoute une chaîne de traitement qui finira par donner un résultat "traité" aux Observers qui ont subscribe().  
Chaque traitement correspond à opérateur. L'opérateur retourne un nouvel Observable qui peut à son tour être traité par un autre opérateur, ou bien qui peut être subscribe().   
Un opérateur prend en argument des paramètres pour faire son traitement. Par exemple, l'opérateur filter() va prendre une fonction pour filtrer (duh !). Cette fonction reçoit chaque valeur émise par l'observable précédent (émis par le précédent opérateur) et retourne true si l'on veut conserver cette valeur, false sinon.  

En code : 

    Rx.Observable.of([10, 20, 30])          // of() génère un Observable à partir d'un tableau
        .map(x => x*2)                      // opérateur map, qui transforme chaque valeur émise
        .filter(x => x < 50)                // filter
        .subscribe(x => console.log(x));    // affiche donc 20 et 40

TODO: fabriquer un observable à la main. Reproduire of() ?

Dans le cas ci-dessous, on doit gérer deux sources Observables : 
 * le contenu d'input text
 * les données issues de JobService.search()
Concernant l'input : l'observable est issu de valueChanges. Si on subscribe dessus, on obtient la valeur de l'input à chaque changement. Si on tape 'a' on obtient 'a'. Si tape ensuite 'u' on obtient 'au', etc.  

Ce flux est traité par deux opérateurs :   
 * debounce va émettre quand le flux est stabilisé au bout d'un certain temps (ici 400ms). Autrement dit, les frappes vont être groupées par paquet temporel.
 * le résultat de ce flux (issu de debounce) va alors atterrir dans distinctUntilChanged qui s'assurera que l'on ne requête pas plusieurs fois pour une même valeur (mais là je ne vois pas comment ça peut arriver  vu que le flux vient d'un valueChanges qui n'émet que lorsqu'il y a des changements)

Si l'on subscribe au bout de ces deux opérateurs on a donc ce qui est tapé comme ça nous arrange, 
et l'on peut faire notre requête. Ce qui donnerait :

    this.term.valueChanges.debounceTime(400).distinctUntilChanges()
        .subscribe(term => this.jobService.search(term).subscribe(items => this.items = items)

Comme on peut le voir, c'est lourdingue.  
A ce stade, on voudrait la clarté des then() des Promise.
On pourrait utiliser l'opérateur map() :

    this.term.valueChanges.debounceTime(400).distinctUntilChanges()
        .map(term => this.jobService.search(term))
        .subscribe(term => console.log(term));

Oui, mais non. Notre JobService.search() retourne un Observable. Cela veut dire que nos Observers vont recevoir des Observable, et non pas les valeurs elles-mêmes.

Heureusement, nous disposons de flatMap dont le boulot est précisément de résoudre ce problème : il prend une fonction qui va retourner un Observable, et il va s'occuper d'émettre la valeur issue de cette Observable, plutôt que l'Observable lui-même. 

    this.term.valueChanges.debounceTime(400).distinctUntilChanges()
        .flatMap(term => this.jobService.search(term))
        .subscribe(items => this.items = items);

Pour être plus précis et rigoureux : la raison d'être de flatMap est la même qu'un flatMap sur Array.

    [ [10, 20], [30, 40] ] -> flatMap -> [10, 20, 30, 40]

En code :
 
    [1, 2, 3].flatMap(x => [x*2, x*3]) nous donnera [2, 3, 4, 6, 6, 9]

flatMap prend donc une fonction de type map qui retourne un array. Le résultat donné par flatMap sera un array aplati. Si la fonction retourne une valeur non-array, flatMap fonctionne comme map.  

La logique est la même pour les Observables. Ceux-ci fournissent des flux/séquences, que l'on peut assimiler à des array. 

    Observable.from([1, 2, 3]).flatMap(x => Observable.from([x*2, x*3]))
    .subscribe(x => console.log(x)) 

Affichera : 2, 3, 4, 6, 6, 9  

Pour reprendre la documentation :  
_"The FlatMap operator transforms an Observable by applying a function that you specify to each item emitted by the source Observable, 
where that function returns an Observable that itself emits items. 
FlatMap then merges the emissions of these resulting Observables, emitting these merged results as its own sequence."_

    this.term.valueChanges // Observable<any>
        .filter(term => term.length > 2)
        .debounceTime(400)
        .distinctUntilChanged()
        .flatMap(term => this.jobService.search(term))
        .subscribe((items) => {
            this.awesomplete.list = items.map(i => i.title);
            this.items = items;
        });
    //  .switchMap(term => {
    //      console.log('switchMap', term);
    //      return this.jobService.search(term)
    //  })

