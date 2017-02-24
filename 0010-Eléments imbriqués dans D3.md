--- Layout : post
--- Title: Elements imbriqués dans D3
--- Date: 2015-03-25

    <DOCTYPE html>
      <head>
       <meta charset="utf-8">
       <style>
           .chart {
               background:grey;
               width:600px;
               height:200px;
           }
           .chart span.container {
               background-color: bisque;
               height:25px;
               margin-top:3px;
               padding-top:5px;
               display:block;
               text-align:right;
               
           }
           .chart div.bar {
               background-color: steelblue;
               height:20px;
               float:left;
           }
       </style>
       <script src="d3/d3.js"></script>
       </head>
       <body>
          <div class="chart"></div>
       </body>
       </html>

    /**
     * Objectif : comprendre le fonctionnement du binding, des sélections
     * et notamment la différence entre select() et selectAll().
     * Le cas étudié est basé sur des éléments DOM imbriqués : 
     * on affiche un histogramme horizontal dont chaque barre est un 
     * conteneur dans lequel se trouve la barre elle-même. On peut utiliser
     * ce genre de situation pour mettre un texte tout à droite du conteneur
     * pendant que la longueur de la barre représente la valeur.
     */
     var srcData = [4, 8, 15, 16, 23, 42];
     x = d3.scale.linear().domain([0, 42]).range([0, 420]);
     /*
     * Association de srcData avec les nodes de classe .container
     * Ce qu'il faut bien comprendre ici est que le mot association est
     * trompeur : en réalité, la fonction data() fait juste une COPIE des 
     * valeurs dans la propriété __data__ de chaque node du DOM concernée
     * par le selectAll().
     * Etant donné que c'est une copie, si la valeur d'origine change 
     * (dans le tableau srcData) ça n'est pas reflété dans le DOM !
     */
     var join = d3.select(".chart").selectAll(".container").data(srcData);
     /*
     * Aucune node de classe .container n'existe pour les données de srcData,
     * elles sont donc créées. Chaque node contient une propriété __data__
     * dont la valeur est la donnée associée.
     */
     var containers = join.enter().append("span").attr("class", "container");
     /*
     * On ajoute à chaque node .container une node enfant de type DIV et de la
     * classe .bar
     * Et ces enfants héritent chacun de la propriété __data__.
     * ET CE N'EST PAS UN HERITAGE PAR REFERENCE :  C'EST UNE COPIE DE VALEUR.
     * Chaque enfant aura une propriété __data__ de même valeur que celle de 
     * son parent, mais il n'y aura aucun lien. Changer le __data__ du parent 
     * ne change pas automatiquement le __data__ de l'enfant !
     * Comme dans le cas de data() : on copie seulement.
     */
     containers.append("div").attr("class", "bar");
     /*
     * Pour faire bonne mesure et trouver un intérêt à la prise de tête de cette
     * exemple, on affiche l'index de la donnée au bout de chaque barre. 
     * Le DOM généré doit être : 
     * <span class=".container"><div class="bar" style="width:valeur"></div><span>index</span></span>
     * On peut ignorer par la suite cet affichage, il n'a aucune incidence sur le reste.
     */
     containers.append("span").text(function(d,i) {
         return i;
     });
     /*
     * Sélection des nodes de classe .bar à partir des nodes de classe .container
     * Les nodes retournées ont leurs propriétés __data__ de mêmes valeurs que 
     * celles de leurs parents.
     */
     var bars = join.selectAll(".bar");
     /*
     * En conséquence, l'application du style fonctionne sans problème.
     * Attendu : 4, 8, 15, 16, 23, 42
     */
     bars.style("width", function(d) {
         return x(d) + "px";
     });
     /*
     * Modification des données sources.
     */
     for (var i = 0; i < srcData.length; i++) {
         srcData[i] /= 2;
     }
     /*
     * D3 ne fait pas d'observation. Il faut ré-assigner les données
     * aux nodes (re-jointure). Commes ces dernières existent déjà à présent, il y a
     * modification de leurs propriétés __data__ avec les nouvelles valeurs.
     */
     join = d3.select(".chart").selectAll(".container").data(srcData);
     /*
     * Comme il n'y aucune nouvelle donnée, il n'y a pas de création de nouvelles
     * nodes. La section enter() est vide.
     */
     join.enter().append("span").attr("class", "container").append("div").attr("class", "bar");
     /*
     * Et voilà donc le piège : le selectAll() nous retourne bien les nodes de classe
     * .bar dont la propriété __data__ a été héritée à la création. Mais ça a été un
     * héritage sous forme de copie. Il n'y a pas de transfert implicite lors de la 
     * re-jointure et les __data__ des nodes .bar sont donc restées inchangées.
     * 
     * selection.selectAll(selector) : The subselection does not inherit data from 
     * the current selection;
     */
     bars = join.selectAll(".bar");
     /*
     * En conséquence, l'application du style n'applique pas les nouvelles valeurs,
     * mais celles qui sont toujours en place.
     * Attendu : 4, 8, 15, 16, 23, 42
     */
     bars.style("width", function(d) {
         return x(d) + "px";
     });
     /*
     * La solution : select()
     * For each element in the current selection, selects the first descendant element 
     * that matches the specified selector string. 
     * Top subtilité ici : select() va bien retourner les nodes .bar MAIS FAISANT
     * HERITER LES __data__ PAR COPIE DANS LA SELECTION RETOURNEE !
     * 
     * selection.select(selector : If the current element has associated data, this 
     * data is inherited by the returned subselection, and automatically bound to the 
     * newly selected elements.
     * 
     * Les données mises à jour qui sont placées dans les nodes .container sont donc
     * copiées dans les enfants .bar
     */
     bars = join.select(".bar");
     /*
     * Résultat : ce sont bien les données mises à jours qui sont utilisées pour
     * appliquer le style sur les .bar
     * Attendu : 2, 4, 7.5, 8, 11.5, 21
     */
     bars.style("width", function(d) {
         return x(d) + "px";
     });
     /*
     * Peut-on néanmoins quand même utiliser selectAll() ? Oui !
     * Nouvelle modification...
     */
     for (var i = 0; i < srcData.length; i++) {
         srcData[i] /= 2;
     }
     
     /*
     * On refait la jointure entre le DOM et les données : les valeurs de
     * srcData sont recopiées dans les nodes .container
     */
     join = d3.select(".chart").selectAll(".container").data(srcData);
     /*
     * Encore une fois, il n'y aucune nouvelle donnée, La section enter() est vide,
     * aucune node n'est créée.
     */
     join.enter().append("span").attr("class", "container").append("div").attr("class", "bar");
     /*
     * selection.selectAll() : For each element in the current 
     * selection, selects descendant elements that match the 
     * specified selector string.
     * 
     * Donc pour chaque .container on sélectionne chaque .bar.
     * Mais attention, ces derniers sont retournés groupés  par 
     * .container et non sous la forme d'un flat array. 
     * Etant donné qu'il n'y a qu'un .bar par .container, on obtient : 
     * [Array[1], Array[1], Array[1], Array[1], Array[1], Array[1]]
     * En équivalient PHP :
     * array( .container0 => array( bar0 => node0),  .container1 => array( bar1 => node1), etc.)
     */
     bars = join.selectAll(".bar");
     /*
     * On associe à présent les data à cette sélection composée de groupes.
     * Et de nouveau une subtilité !
     * 
     * selection.data(array) : The values array specifies the data *for each group* 
     * in the selection. Thus, if the selection has multiple groups (such as a 
     * d3.selectAll followed by a selection.selectAll), then data should be specified 
     * as a function that returns an array (assuming that you want different data for 
     * each group). 
     * 
     * Encore une fois, D3 ne fait pas de miracles. On lui a demandé (et il nous l'a même
     * un peu imposé) des nodes organisées en groupes, et l'association avec des données
     * ne peut se faire avec n'importe quoi, n'importe comment.
     * On va donc fournir une fonction qui retourne un array correspond à chaque groupe. 
     * D3 va ensuite associer les éléments du array à chaque node du groupe.
     * La fonction reçoit en argument les données en "brut" (1, 2, 3.75, 4, etc.), 
     * on renvoit donc chaque donnée dans un tableau qui ne contiendra donc qu'un seul élément.
     * Résultat : 
     * .container0 reçoit [1] et D3 s'occupe d'associer la valeur 1 à .bar0 
     * .container1 reçoit [2] et D3 s'occupe d'associer la valeur 2 à .bar0 
     * .container2 reçoit [3.75] et D3 s'occupe d'associer la valeur 3.75 à .bar0      
     */
     bars.data(function(d) {
         return [d];
     });
     /*
     * Attendu : 1, 2, 3.75, 4, 5.75, 10.5
     */
     bars.style("width", function(d) {
         return x(d) + "px";
     });

