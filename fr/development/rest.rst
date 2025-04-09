REST
####

Beaucoup de programmeurs néophytes d'application réalisent qu'ils ont
besoin d'ouvrir leurs fonctionnalités principales à un public plus important.
Fournir un accès sans entrave à votre API du cœur peut
aider à ce que votre plateforme soit acceptée, et permettre les
mashups et une intégration facile avec les autres systèmes.

CakePHP propose des méthodes pour exposer les actions de votre controller via
des méthodes HTTP et pour sérialiser les variables de vue en fonction de la
négociation du type de contenu. La négociation du type de contenu permet aux
clients de votre application d’envoyer des requêtes avec des données sérialisées
et de recevoir des réponses avec des données sérialisées via les en-têtes ``Accept``
et ``Content-Type``, ou des extensions d’URL.

Mise en place Simple
====================

Pour commencer à ajouter une API REST à votre application, nous aurons d’abord
besoin d’un controller contenant les actions que nous voulons exposer en tant
qu’API. Un controller de base pourrait ressembler à ceci::

    // src/Controller/RecipesController.php
    use Cake\View\JsonView;

    class RecipesController extends AppController
    {
        public function viewClasses(): array
        {
            return [JsonView::class];
        }

        public function index()
        {
            $recipes = $this->Recipes->find('all')->all();
            $this->set('recipes', $recipes);
            $this->viewBuilder()->setOption('serialize', ['recipes']);
        }

        public function view($id)
        {
            $recipe = $this->Recipes->get($id);
            $this->set('recipe', $recipe);
            $this->viewBuilder()->setOption('serialize', ['recipe']);
        }

        public function add()
        {
            $this->request->allowMethod(['post', 'put']);
            $recipe = $this->Recipes->newEntity($this->request->getData());
            if ($this->Recipes->save($recipe)) {
                $message = 'Saved';
            } else {
                $message = 'Error';
            }
            $this->set([
                'message' => $message,
                'recipe' => $recipe,
            ]);
            $this->viewBuilder()->setOption('serialize', ['recipe', 'message']);
        }

        public function edit($id)
        {
            $this->request->allowMethod(['patch', 'post', 'put']);
            $recipe = $this->Recipes->get($id);
            $recipe = $this->Recipes->patchEntity($recipe, $this->request->getData());
            if ($this->Recipes->save($recipe)) {
                $message = 'Saved';
            } else {
                $message = 'Error';
            }
            $this->set([
                'message' => $message,
                'recipe' => $recipe,
            ]);
            $this->viewBuilder()->setOption('serialize', ['recipe', 'message']);
        }

        public function delete($id)
        {
            $this->request->allowMethod(['delete']);
            $recipe = $this->Recipes->get($id);
            $message = 'Deleted';
            if (!$this->Recipes->delete($recipe)) {
                $message = 'Error';
            }
            $this->set('message', $message);
            $this->viewBuilder()->setOption('serialize', ['message']);
        }
    }

Dans notre ``RecipesController``, nous avons plusieurs actions qui définissent la logique
pour créer, modifier, visualiser et supprimer des recettes. Dans chacune de nos actions,
nous utilisons l’option ``serialize`` pour indiquer à CakePHP quelles variables de vue doivent
être sérialisées lors de la création des réponses API. Nous connecterons notre controller aux
URL de l’application avec le :ref:`resource-routes`::

    // in config/routes.php
    $routes->scope('/', function (RouteBuilder $routes): void {
        $routes->setExtensions(['json']);
        $routes->resources('Recipes');
    });

Ces routes permettront aux URL comme ``/recipes.json`` de renvoyer une réponse encodée en JSON.
Les clients pourront également faire une requête à ``/recipes`` avec l’en-tête
``Content-Type: application/json``.

Encodage des données de réponse
===============================

Dans le controlleur ci-dessus, nous définissons une méthode ``viewClasses()``. Cette méthode
définit les vues dont votre controller dispose pour la négociation de contenu. Nous incluons
``JsonView`` de CakePHP qui permet des réponses basées sur JSON. Pour en savoir plus à ce sujet et
sur les vues basées sur XML, consultez :doc:`/views/json-and-xml-views`. Ceci est utilisé par
CakePHP pour sélectionner une classe de vue avec laquelle restituer une réponse REST.

Ensuite, nous disposons de plusieurs méthodes qui exposent la logique de base pour créer,
modifier, afficher et supprimer des recettes. Dans chacune de nos actions, nous utilisons
l'option ``serialize`` pour indiquer à CakePHP quelles variables de vue doivent être sérialisées
lors des réponses API.

Si nous souhaitons modifier les données avant qu'elles ne soient converties en JSON,
nous ne devons pas définir l'option de ``serialize``, mais plutôt utiliser des fichiers modèles.
Nous placerions les modèles REST pour notre RecipesController dans **templates/Recipes/json**.

Voir :ref:`controller-viewclasses` pour plus d'informations sur la fonctionnalité de négociation
de réponse de CakePHP.

Parsing des corps de requête
============================

La création de la logique de l'action de modification nécessite une autre étape. Parce
que nos ressources sont sérialisées au format JSON, il serait ergonomique si nos requêtes
contenaient également la représentation JSON.

Dans notre classe ``Application``, assurez-vous que les éléments suivants sont présents::

    $middlewareQueue->add(new BodyParserMiddleware());

Ce middleware utilisera l'en-tête ``content-type`` pour détecter le format des données
de requête et analyser les formats activés. Par défaut, seule l'analyse ``JSON`` est activée
par défaut. Vous pouvez activer la prise en charge XML en activant l'option du
constructeur XML. Lorsqu'une requête est effectuée avec un ``Content-Type`` ``application/json``,
CakePHP décodera les données de la requête et mettra à jour la requête afin
que ``$request->getData()`` contienne le corps analysé.

Vous pouvez également câbler des désérialiseurs supplémentaires pour des formats alternatifs
si vous en avez besoin, en utilisant :php:meth:`BodyParserMiddleware::addParser()`.

.. meta::
    :title lang=fr: REST
    :keywords lang=fr: application programmers,default routes,core functionality,result format,mashups,recipe database,request method,access,config,soap,recipes,logic,audience,cakephp,running,api
