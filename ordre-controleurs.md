 

* @Route("/1/classes/eventMaster/{eventMasterId}")

* @Route("/1/classes/eventMaster/authenticate")

La seconde route n'est jamais matchée car attrapée par la première (eventMasterId = "authenticate")

Il faut blinder le type de eventMasterId : 
* @Route("/1/classes/eventMaster/{eventMasterId}", requirements={"eventMasterId" = "\d+"})

 ou bien inverser l'ordre.