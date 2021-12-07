# Laboratoire 3 - Load Balancing

> Auteurs : Lucas Gianinetti, Nicolas Hungerbühler, Cassandre Wojciechowski
>
> Cours : AIT
>
> Date : 18.11.2021

## Introduction

// description globale du labo

## Tâche 1 - Installation des outils

> Les outils `Docker` et `docker-compose` sont déjà installés sur l'ordinateur utilisé pour ce laboratoire. 

Nous avons donc installé `JMeter` avec la commande : 

```bash
$ sudo apt install jmeter 'jmeter-*'
```

Nous avons ensuite lancé la première commande suivante dans le dossier racine du laboratoire, et la seconde pour vérifier que les conteneurs aient été lancés :

```bash
$ docker-compose up --build
$ docker ps
```

Nous constatons que trois conteneurs différents ont été démarrés : `balancing_haproxy`, `balancing_webapp1`, `balancing_webapp2`. 

Nous vérifions que la configuration `bridge` est bien appliquée : 

```bash
$ docker network ls
NETWORK ID     NAME                                                      DRIVER    SCOPE
d2ee042a1ac2   bridge                                                    bridge    local
13cfd9b0b7ae   docker_default                                            bridge    local
544dd0d58670   host                                                      host      local
5f9a574fe881   none                                                      null      local
5e029413fa29   teaching-heigvd-ait-2019-labo-load-balancing_public_net   bridge    local

```

En nous connectant via un browser sur l'adresse http://192.168.42.42:80, nous trouvons un document JSON : 

```json
{
  "hello": "world!",
  "ip": "192.168.42.11",
  "host": "8bc128915098",
  "tag": "s1",
  "sessionViews": 1,
  "id": "RYerKjPmA4IZXgnMCUvgwaiVzfoeqcCq"
}
```

Nous avons lancé le script `tester.jmx` depuis `JMeter` et nous observons le résultat suivant : 

![](assets/img/task1_jmeter.png)

Nous constatons que les requêtes sont bien réparties équitablement entre les deux applications : 500 ont été dirigées sur `s1` et 500 sur `s2`. 



**1.1 Explain how the load balancer behaves when you open and refresh the URL http://192.168.42.42 in your browser. Add screenshots to complement your explanations. We expect that you take a deeper a look at session management.**

La première requête que nous faisons nous fait apparaître le tag `s1` avec un id commençant par `Lpqs `.

![](assets/img/task1_1.1_1.png)

La seconde fait apparaître le tag `s2` et un id commençant `3Tsi`. 

![](assets/img/task1_1.1_2.png)

Une requête sur deux aura le tag `s1` et l'autre aura donc le tag `s2`, nous pouvons donc déduire le load balancer alterne les redirections sur les deux web app. 

Par contre, les identifiants changent à chaque rafraichissement de page, donc nous changeons de session à chaque fois. Cela explique que le champ `sessionViews` reste toujours à 1. 



**1.2 Explain what should be the correct behavior of the load balancer for session management.**

Le comportement qui semble logique serait qu'une session reste active tant que l'utilisateur n'a pas quitté l'application web. Cela permettrait d'incrémenter correctement le compteur de requêtes envoyées. 

Par contre, il faut noter que l'utilisateur d'une session doit toujours être redirigé sur le même serveur. 



**1.3 Provide a sequence diagram to explain what is happening when one requests the URL for the first time and then refreshes the page. We want to see what is happening with the cookie. We want to see the sequence of messages exchanged (1) between the browser and HAProxy and (2) between HAProxy and the nodes S1 and S2.**

![](assets/img/task1_1.3.png)

Lors de la première requête, la webapp 1 attribue un cookie à l'utilisateur. Lors de la seconde requête, ce premier cookie est envoyé à la webapp 2. Celle-ci ne le reconnait pas et en attribue donc un nouveau à l'utilisateur. 

Le cookie est donc changé à chaque fois, car les webapps ne reconnaissent pas les cookies attribués par l'autre. 

![](assets/img/task1.3_diag_seq.jpeg)



**1.4 Provide a screenshot of the summary report from JMeter**

 ![](assets/img/task1_jmeter.png)



**1.5 Stop the containers and re-run the test plan in JMeter. Explain what is happening when only one node remains active. Provide another sequence diagram using the same model as the previous one.**

Nous arrêtons le conteneur s1 avec la commande :

```bash
$ docker stop s1
```

Après son arrêt, nous relançons JMeter et nous obtenons le rapport suivant :

![](assets/img/task1.5_jmeter_s2_only.png)

Sur la capture ci-dessus, nous constatons que toutes les requêtes sont donc effectuées sur la webapp s2. Selon les tests manuels que nous avons pu faire, le compteur est bien incrémenté à chaque requête et le cookie utilisé reste systématiquement le même. 

Le diagramme de séquence réalisé suite à cette tentative est le suivant : 

![](assets/img/task_1.5_diagram_seq.jpeg)

La webapp reconnait systématiquement le cookie qui est envoyé car c'est elle qui l'a attributé à la base. Il n'est jamais changé par une autre webapp et permet donc d'incrémenter le compteur `sessionViews`. 



## Tâche 2 - La persistence des sessions (sticky sessions)

**2.1 There is different way to implement the sticky session. One possibility is to use the SERVERID provided by HAProxy. Another way is  to use the NODESESSID provided by the application. Briefly explain the difference between both approaches (provide a sequence diagram with cookies to show the difference).**

Avec la methode SERVERID, le load balancer va séparer le cookie avec une partie SERVERID qui est collée au serveur correspondant. On peut ensuite récupérer le bon cookie pour récupérer la bonne session :

```sequence
Browser->HaProxy: GET / \n HOST:192.168.42.42
HaProxy->S1: GET / \n HOST:192.168.42.11
S1->HaProxy: {hello:..., sessionViews:1, id: a}
HaProxy->Browser: {hello,..., sessionViews:1, id: a} \n SERVERID=s1 
Browser->HaProxy: GET / \n HOST:192.168.42.42 \n SERVERID=s1 NODESESSID=a
HaProxy->S1: GET / \n HOST:192.168.42.11 \n NODESESSID=a
S1->HaProxy: {hello:..., sessionViews:2, id:a}
HaProxy->Browser: {hello:..., sessionViews:2, id:a} \n SERVERID=s1 NODESESSID=a
```

Dans la deuxième methode, on va insérer le SERVERID directement dans le cookie NODESESSID. Il est séparer par un ~. Comme pour la methode précédente, pour le serveur il n'y a auune différence. C'est uniquement le load balancer qui gère de mettre le SERVERID dans le NODESESSID. 

```sequence
Browser->HaProxy: GET / \n HOST:192.168.42.42
HaProxy->S1: GET / \n HOST:192.168.42.11
S1->HaProxy: {hello:..., sessionViews:1, id: a}
HaProxy->Browser: {hello,..., sessionViews:1, id: a} \n NODESESSID=s1~a
Browser->HaProxy: GET / \n HOST:192.168.42.42 \n NODESESSID=s1~a
HaProxy->S1: GET / \n HOST:192.168.42.11 \n NODESESSID=a
S1->HaProxy: {hello:..., sessionViews:2, id:a} \n NODESESSID=a
HaProxy->Browser: {hello:..., sessionViews:2, id:a} \n NODESESSID=s1~a
```

**2.2 Provide the modified `haproxy.cfg` file with a short explanation of the modifications you did to enable sticky session management.**

Source : https://www.haproxy.com/fr/blog/load-balancing-affinity-persistence-sticky-sessions-what-you-need-to-know/
`

    backend nodes
    
    ...
    
    cookie SERVERID insert indirect nocache
    
    # Define the list of nodes to be in the balancing mechanism
    # http://cbonte.github.io/haproxy-dconv/2.2/configuration.html#4-server
    server s1 ${WEBAPP_1_IP}:3000 check cookie s1
    server s2 ${WEBAPP_2_IP}:3000 check cookie s2`

* `cookie SERVERID insert indirect nocache`
  * Indique à HAProxy qu'il faut configurer un cookie SERVERID uniquement si l'utilisateur n'en a pas fourni un avec sa requête.
* `check cookie s1` et `check cookie s2`
  * Indique à HAProxy sur quel serveur rediriger l'utilisateur en fonction de son cookie.

**2.3 Explain what is the behavior when you open and refresh the URL http://192.168.42.42 in your browser. Add screenshots to complement your explanations. We expect that you take a deeper a look at session management.**

L'utilisateur est toujours redirigé vers la même webapp, une même session est utilisée pour toutes les requêtes de l'utilisateur.

![](./assets/img/task2_3_1.png)
![](./assets/img/task2_3_2.png)

On peut voir ci-dessus que la session est bien identique pour les différentes requêtes d'un utilisateur. On voit aussi que le cookie SERVERID est bien présent afin d'indiquer au load balancer ou rediriger la requête (dans ce cas sur la webapp1).


**2.4 Provide a sequence diagram to explain what is happening when one requests the URL for the first time and then refreshes the page. We want to see what is happening with the cookie. We want to see the sequence of messages exchanged (1) between the browser and HAProxy and (2) between HAProxy and the nodes S1 and S2. We also want to see what is happening when a second browser is used.**



**2.5 Provide a screenshot of JMeter's summary report. Is there a difference with this run and the run of Task 1?**

![](./assets/img/task2_5_1.png)

Nous observons que toutes les requêtes ont été redirigées vers la même webapp et donc que le sticky session est implémenté correctement. C'est différement du cas de la task1 ou l'on était redirigé de manière uniforme sur les deux webapps.

- **Clear the results in JMeter.**
- **Now, update the JMeter script. Go in the HTTP Cookie Manager and ~~uncheck~~verify that the box `Clear cookies each iteration?` is unchecked.**
- **Go in `Thread Group` and update the `Number of threads`. Set the value to 2.**

  

**2.6 Provide a screenshot of JMeter's summary report. Give a short explanation of what the load balancer is doing.**

![](./assets/img/task2_6_1.png)

Nous avons configuré JMeter pour simuler l'envoi par deux utilisateurs de 1000 requêtes chacun. De plus les cookies ne sont pas reset à chaque nouvelle requête.

Nous observons le résultat attendu:
* Lors de sa première requête un des deux utililisateurs a été redirigé vers la webapp1. Lors des 999 requêtes suivantes il a été redirigé vers la même webapp.
* Pour l'autre utilisateur, le scénario est identique mais avec la webapp2.


## Tâche 3 - Le drainage des connexions (drain mode)

**3.1 Open a browser on your machine.**

**3.2 Navigate to http://192.168.42.42. You will reach one of the two nodes. Let's assume it is s1 but in your case, it could be s2 as the balancing strategy is roundrobin.**

**3.3 Refresh the page. You should get the same result except that the sessionViews counter is incremented.**

**3.4 Refresh multiple times the page and verify that you continue to reach the same node and see the sessionViews counter increased.**

**3.5 In a different tab, open HAProxy's statistics report page at http://192.168.42.42:1936 and take a look.**


## Tâche 4 - Le mode dégradé avec Round Robin

## Tâche 5 - Les stratégies de load balancing

## Conclusion 

// ce que ce labo vous a apporté, ce que vous en retenez