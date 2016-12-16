#Wrapping d'un input avec ControlValueAccessor

##Objectif : 
Nous disposons d'une directive sur input. Par exemple, NgbTypeahead : 

    <input type="text" [ngbTypeahead]="search" [(ngModel)]="result" />

On veut abstraire cette ligne dans un composant. Par exemple : 

    <job-typeahead [(ngModel)]="myJob" />

##Problème

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

## Principes ControlValueAccessor
L'objectif est d'avoir du 2-ways-data-binding. Donc notre contrôle doit être informé quand le modèle change depuis l'extérieur, et il doit prévenir Angular quand il a changé depuis l'intérieur (typiquement via la vue).
L'implémentation de ControlValueAccessor requiert la présence de trois méthodes : 
 * `writeValue(value: any): void` sera appelée quand le modèle change. Charge à notre composant de mettre à jour la vue.
 * `registerOnChange(fn: any): void` va nous fournir une fonction de propagation. Nous appelerons cette fonction à chaque fois que la vue de notre composant change.
 * `registerTouched(fn: any): void` est spécifique au touch (non couvert par cet article).

##Mise en oeuvre

    @Component({
        selector: 'job-typeahead',
        template: '<div><input type="text" [ngbTypeahead]="search" [(ngModel)]="job" (keyup)="onKeyUp($event)" /></div>'
    })
    export class JobTypeaheadComponent {

        private job: string;

        writeValue(value: any): void {
            this.job = value;
        }

        propagateChange = (_: any) => { };
        registerOnChange(fn: any): void {
            this.propagateChange = fn;
        }

        onKeyUp($event) {
            this.propagateChange(this.job);
        }

        registerOnTouched(fn: any): void { }

    }

Que se passe t'il au sein de notre JobTypeaheadComponent ?  
Lorsque l'utilisateur entre des données dans l'input, l'événement keyup est intercepté. On exploite cette interception indiquer à Angular que le modèle a changé et qu'il faut propager cette information à tous ceux qui en ont besoin. Cette propagation se fait via la fonction qu'Angular nous a donné en paramètre de registerOnChange.  

Pour l'autre sens, imaginons ceci : 

    <job-typeahead [(ngModel)]="customerJob" />
    <button (click)="customerJob = 'new job'">Change job</button>

Si l'utilisateur clique sur le bouton, cela va changer le modèle customerJob. Or, celui-ci a été passé via le `[(ngModel)]` au composant job-typeahead.  
Comme celui-ci implémente `ControlValueAccessor.writeValue()`, cette méthode va être appelée avec le contenu de customerJob. Le composant peut alors se mettre à jour. Ici, cela se fait implicitement par `[(ngModel)]="job"`.  
Pour bien le voir, on peut "sucrer" notre code ainsi : 

    @Component({
        selector: 'job-typeahead',
        template: `
        <div>
            {{jobUpperCase}}
            <input type="text" [ngbTypeahead]="search" [(ngModel)]="job" (keyup)="onKeyUp($event)" />
        </div>`
    })
    export class JobTypeaheadComponent {

        ...

        writeValue(value: any): void {
            this.job = value;
            this.jobUpperCase = value.strtoupper();
        }

        ...

On voit alors bien le rôle de writeValue : mettre à jour la vue de notre composant lorsque le modèle évolue depuis l'extérieur.

##One more thing
Le fait d'avoir implémenté ControlValueAccessor ne suffit pas. La notion d'interface est inexistante en ES5 : l'information disparaît une fois notre code Typescript transpilé. Angular ne peut donc pas savoir que notre composant est apte à gérer NgModel via ControlValueAccessor.  
De manière générale, Angular injecte l'information via NG_VALUE_ACCESSOR. Il s'agit d'un token d'injection de dépendance (DI) qui a la particularité d'injecter toutes les implémentations disponibles de ControlValueAccessor (celle de input type="text", ou input type="checkbox", etc.).  
Ce que nous voulons donc faire c'est étendre ce token en ajoutant notre implémentation à la suite de celles déjà présentes. Cela se fait au niveau du composant avec le code suivant : 

    @Component({
        selector: 'job-typeahead',
        template: `...`,
        providers: [
            {
                provide: NG_VALUE_ACCESSOR,
                useExisting: forwardRef(() => JobTypeaheadComponent),
                multi: true
            }
        ]
    })
    
