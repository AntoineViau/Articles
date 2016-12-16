<!--
--- Layout: post
--- Title: Association Many-To-One générique avec Doctrine
---- Date: 2015-03-25
-->
Imaginons un modèle avec des entités Client, Prestataire, Fabricant. Chacune de ces entités possède une ou plusieurs adresses (livraison, filiale, siège, etc.). Nous sommes donc dans un cas typique de relation ManyToOne : plusieurs adresses pour une entité.

Doctrine nous permet donc d'écrire :

    class Adresse {
      /**
       * @ManyToOne(targetEntity="Client")
       */
       protected $client;
       ...
    }
    class Client {
       protected $adresses;
       
       public function addAdresse($adresse) {
           $adresse->setClient($this);
           $this->adresses[] = $adresse;
    }

Mais comment éviter d'avoir à répéter ce même bout de code pour Prestataire et Fabricant ?

**Solution : l'héritage**

Même si la notion d'héritage n'est pas conceptualisée dans les bases de données relationnelles, on peut compter sur Doctrine pour l'intégrer.

On peut donc tout à faire écrire :

    /**
     * @Entity
     * @InheritanceType("JOINED")
     * @DiscriminatorColumn(name="TypeEntite", type="string")  
     * @DiscriminatorMap({"Fabricant" = "Fabricant", "Prestataire" = "Prestataire", "Client" = "Client"})
     */
    class EntiteAvecAdresse {
       /**
        * @Id @Column(type="integer", length=64, unique=true) @GeneratedValue(strategy="AUTO")
        */
       protected $id;
       protected $adresses;
       public function addAdresse($adresse) {
           $adresse->setOwner($this);
           $this->adresses[] = $adresse;
    }

    class Client extends EntiteAvecAdresse {
       ...
    }
    class Fabricant extends EntiteAvecAdresse {
       ...
    }
    class Prestataire extends EntiteAvecAdresse {
       ...
    }
    class Adresse {
      /**
       * @ManyToOne(targetEntity="EntiteAvecAdresse")
       */
       protected $entiteAvecAdresse;
       ...
    }

Explications :

Pour reproduire l'héritage au sein de la base de données, Doctrine peut fonctionner par une jointure (d'où l'option JOINED dans @InheritanceType). Pour ce faire, il crée une table EntiteAvecAdresse dans laquelle on a l'identifiant $id (clef primaire) et les adresses. Il ajoute en plus une colonne de type (DiscriminatorColumn). Le type est ici stocké sous la forme d'une string, et chaque type est identifié par le nom de classe de l'entité.


Lorsque Doctrine va créer les tables Client, Prestataire et Fabricant, il va reprendre l'identifiant de la table/classe parente.

Par exemple, si nous avons un client avec id=2501, alors nous aurons dans la table EntiteAvecAdresse une ligne avec id=2501 et TypeEntite="Client"

De cette façon, Doctrine sait toujours qui est quoi.

Le prix à payer de cette solution est l'utilisation d'une jointure supplémentaire à chaque fois que l'on accède aux entités filles.  

On peut régler ce problème de performance en utilisant l'option SINGLE_TABLE dans l'annotation @InheritanceType : toutes les entités filles seront stockées avec l'entité parente dans une seule table contenant tous les attributs de tout le monde. Le prix à payer dans ce cas est la quantité de données mortes et la cohérence de la table par rapport au modèle : un grand fourre-tout en opposition avec le découpage en classes héritées.

**Références :**
  
  * [Doctrine Many-To-One, Unidirectional](http://doctrine-orm.readthedocs.org/en/latest/reference/association-mapping.html#many-to-one-unidirectional)
  * [Doctrine Inheritance Mapping](http://doctrine-orm.readthedocs.org/en/latest/reference/inheritance-mapping.html)
  

