#Wrapping d'un input sans ControlValueAccessor

##Objectif : 
Nous avons un input qui accepte la directive ngModel pour le 2-ways-data-binding. 
On veut pouvoir wrapper cet input pour abstraire son utilisation.
Prenons comme exemple, l'input de NgbTypeahead : 

    <input type="text" [ngbTypeahead]="search" [(ngModel)]="result" />

 On voudrait avoir un composant qui :
 * pré-rempli la liste des valeurs possibles
 * est capable de faire du 2 ways binding avec son appelant

Par exemple, un composant qui fait une recherche sur une liste de jobs. Ce qui donnerait : 
   
    <job-typeahead [(ngModel)]="myJob" />

##Première approche
L'approche naïve serait de vouloir transférer le ngModel par @Input :

    @Component({
        selector: 'job-typeahead',
        template: '<div><input type="text" [ngbTypeahead]="search" [(ngModel)]="ngModel" /></div>'
    })
    export class JobTypeaheadComponent {
        @Input() ngModel;
    }

Avec un appel : 

    <job-typeahead [(ngModel)]="myJob" />

Cette solution ne fonctionne pas parce que ngModel n'est pas une variable mais une directive d'Angular.  
On peut tout à fait l'intégrer à un composant "maison" pour que celui-ci gère le 2-ways-data-binding mais cela implique des règles strictes, en l'occurrence implément un ControlValueAccessor.
Donc, quand Angular compile notre `<job-typeahead [(ngModel)]="job" />` il s'attend à ce que le composant JobTypeaheadComponent implémente ControlValueAccessor.

##Solution
Pour éviter d'en passer par là - et au prix d'une légère concession - on peut exploiter deux choses : 
 * le fonctionnement interne de la syntaxe `[()]`
 * et son  application à ngModel.

La doc Angular nous explique que la syntaxe `[()]` est en fait un raccourci : 

"_When Angular sees a binding target in the form [(x)], it expects the x directive to have an x input property and an xChange output property._"  

On peut donc écrire : 

    <job-typeahead [(job)]="myJob" />

Qui sera décomposé en interne sous la forme :

    <job-typeahead [job]="myJob" (jobChange)="..." />

Cela implique notre composant doit avoir un @Input job et un @Output jobChange.  
Il se présente donc ainsi : 

    @Component({
        selector: 'job-typeahead',
        template: `<input type="text" [ngbTypeahead]="search" [(ngModel)]="job" />`
    })
    export class JobTypeaheadComponent {
        @Input() job;                               // la property exigée par [job]="..."
        @Output jobChange = new EventEmitter();     // l'évènement exigé par (jobChange)="..."        
    }

A ce stade, on a quelque chose qui fonctionne mais ne fait pas ce qu'on veut.  
Ce qui est saisi par l'utilisateur est bien capté au sein de notre composant (dans la propriété `@Input job`), mais on ne fait rien pour sortir le résultat. jobChange est un émetteur d'événements qui n'émet... rien.  
Il faut donc faire la liaison entre cet émetteur et la propriété job.  
Pour ce faire, nous allons exploiter le raccourci de `[()]` qu'Angular applique sur la directive ngModel elle-même.

    @Component({
        selector: 'job-typeahead',
        template: '<input type="text" [ngbTypeahead]="search" [ngModel]="job" (ngModelChange)="change($event)" />'
    })
    export class JobTypeaheadComponent {
        @Input() job;
        @Output jobChange = new EventEmitter();
        change(value) {
            this.jobChange.emit(value);
        }
    }

_Petite remarque : le $event est nécessaire et réservé par Angular, même si nous n'avons pas à faire à un événement mais bien à une valeur. On ne peut donc pas écrire (ngModelChange)="change(val)" dans le template._  

Et nous y voilà !  
Qu'avons nous fait ? Pour le comprendre il faut se mettre à la place d'Angular.  
Lorsque nous avons :

    <input [(ngModel)]="firstName" />

En réalité nous avons : 

    <input [ngModel]="firstName" (ngModelChange)="firstName = $event" />

Rien ne nous interdit d'écrire la même syntaxe avec nos propres paramètres. En forçant la décomposition de [(ngModel)] nous prenons la main sur le changement de valeur du modèle. Ce faisant, nous lui assignons notre propre méthode qui va émettre l'évènement voulu.  

La seule concession que nous faisons à notre objectif initial réside dans le nom de la propriété @Input de job-typeahead. Nous aurions voulu avoir :

    <job-typeahead [(ngModel)]="myJob" />

Et nous avons :

    <job-typeahead [(job)]="myJob" />

On peut dire qu'on y gagne en sémantique, mais qu'on y perd en continuité.