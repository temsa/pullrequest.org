Title: Web Socket : Are you plugged ?
Author: Damien Feugas
Date: Apr 19 2011 11:46:00 GMT-0200 (CDT)
Categories: java, javascript, mythicforge, graniteds, jetty, JMS, STOMP, websocket

# Un petit peu de contexte

Depuis à peu près deux ans, je réalise un [MMORPG][1] gratuit et OpenSource avec un serveur Java et eux clients Flex (un d'administration et un de jeu).

J'ai utilisé le merveilleux framework [GraniteDS][2] qui est un pont entre le java et flex, comme le BlazeDS d'Adobe, avec plus de fonctionnalités encore.  
  
Malheureusement, j'ai surtout besoin de flexibilité coté client de jeu (tout le GameDesign est configurable via l'admin), et j'ai décidé finalement de tout réécrire en technologies Html 5/Css 3/Js.

Les 3 grandes fonctionnalités de GraniteDS qui doivent donc être remplacées sont les suivantes:

1.  L'invocation distante de méthodes Java, remplacée par une API REST. Chaque méthode du serveur est une url qui produit et consomme du XML ou du JSON. J'ai choisi [Jersey ][3]pour ça (implémentation de référence de la spec JAX-RS), et expliqué la migration dans un article précédent.  
     
2.  Le MCV client "tide", remplacé par [RESTHub-js][4], un petit framework client en javascript.  
     
3.  Le push serveur : les clients flex sont constamment connecté au serveur qui leur envoi les mise à jour déclenchée par les autres joueurs. Ce billet explique comment j'ai remplacé cette partie par l'utilisation des WebSockets.
 
# Alors, comment on joue ?

Les Web sockets sont juste... des socket. C'est un canal connecté entre le navigateur et le serveur, ni plus, ni moins. Vous avez donc besoin d'un navigateur récent (Chrome, IE9 or Firefox 4 correctement configuré), et d'un server.

Il y a quelques serveur Java qui implémentent le protocole : jWebSocket, Kaazing, webbit... Mais aucun d'entre eux n'est aussi un conteneur de Servlet, la base de nos serveurs java. A l'exception de [Jetty][5].  
  
Sans rentrer dans les détails, Jetty est un serveur Http+Servlet+WebSocket très puissant écrit en java, qui peut être utilisé en mode embarqué ou standalone. Il implémente le brouillon de la norme Websocket depuis un petit moment, et [plutôt simplement][6].

	public class WebSocketDummyServlet extends WebSocketServlet{
		
		/**
		 * Invoked by Jetty during the handshake: Return a WebSocket object to allow 
		 * the connection establishement.
		 *
		 * <em>@param request</em> Http upgrade request
		 * <em>@return</em> The WebSocket Channel.
		 */
		protected WebSocket doWebSocketConnect(HttpServletRequest request, String protocol) {
			return new DummyWebSocket();
		}
		
		/**
		 * Websocket channel dummy implementation.
		 */
		implements WebSocket {

			/**
			 * Object that sends message to the connected client with method 
			 * sendMessage(String message).
			 *
			 */
			Outbound _outbound;

			/**
			 * Channel connexion.
			 */
			public void onConnect(Outbound outbound) {
				_outbound=outbound;
			}
		  
			/**
			 * Invoked when the client sends a message.
			 * <em>@param</em> <em>data </em>The sent data
			 */
			public void onMessage(byte frame, String data){}

			/**
			 * Channel disconnexion.
			 */
			public void onDisconnect(){}
		}
	}

Plutôt facile, non ? Cette "espèce de servlet" doit être déclarée dans le descripteur web.xml :

    <servlet>
        </servlet-name>
        <servlet-class>org.dummy.WebSocketDummyServlet</servlet-class>
        <load-on-startup>1</load-on-startup>
    </servlet>
    <servlet-mapping>
        <servlet-name>wsServlet</servlet-name>
        <url-pattern>/*</url-pattern>
    </servlet-mapping>


Vous avez besoin d'envoyer des messages à tous les clients connecté ? Vous n'avez qu'a stocker les instance de  DummyWebSocket créées, et ajouter une méthode qui utilisera _outbound.sendMessage().

# Je suis connecté ! Mais je peux rien faire...

Comme je disais, vous n'avez qu'un tuyau connecté. Il transporte des chaîne de caractères et des bits. C'est efficace, mais pas vraiment utilisable en tant que tel.

Il est donc nécessaire d'implémenter un protocole au dessus de ce tuyau. Ce dernier dépendra de vos besoin. Une application de messagerie instantannée ? Utilisez XMPP. Une application de streaming ? Pourquoi pas RTP. Un jeu FPS ? créez  votre propre protocole.  
GraniteDS proposait un mécanisme de publication-souscription de POJO sérialisés, proche de JMS. Heureusement, il existe un équivalent parfait : [STOMP][7].

Une minute ! Google donne quelques résultat pour "STOMP Websocket Java". Notamment ActiveMQ, RabittMQ et HornetMQ, fameux brokers JMS. Utilisons les. Enfin non : c'est vraiment de l'overkill : je n'avais pas besoin de toute cette mécanique complexe...

Alors j'ai implémenté le protocole STOMP (sans la gestion transactionnelle) et ça m'a pris deux jours. En fait, STOMP est vraiment très simple (tout est en texte, et les sauts de lignes sont significatifs) :

**client X, client Y :**
	
	CONNECT
	login: X <or> Y
	passcode: <passcode>

	^@

**client Y:**
	
	SUBSCRIBE
	destination: /topic-1
	ack: client

	^@

**client X:**
	
	SEND
	destination: /topic-1

	hello everyone !
	^@

**et le client Y reçoit:**
	
	MESSAGE
	destination:/topic-1
	message-id: <message-identifier>

	hello everyone !
	^@

# Et coté client justement ?

Le client en javascript est vraiment simple :

	var location = document.location.toString().replace('http:','ws:');
	this._ws=new WebSocket(location);
	this._ws.onopen=this._onopen;
	this._ws.onmessage=this._onmessage;
	this._ws.onclose=this._onclose;

	_onopen: function(){
	},

	_send: function(message){
	  this._ws.send(message);
	},

	_onmessage: function(message) {}
  
Je n'ai pas encore choisi d'implémentation STOMP coté client et dès que je l'aurai fait, je modifierai cet article.  
  
Juste un avertissement : nous l'avons testé à travers des proxies, et ça fonctionne très bien.  
Pas de déconnexion intempestives, pas de ralentissements.  
Mais cela nécessite que le client envoi un keep-alive à travers le socket. Le serveur n'a pas besoin de répondre.  
  
Un message de keep-alive toutes les 10 secondes marche bien, mais j'imagine que cela dépend des configuration des proxies et firewall traversés.

# J'adore ! Où est le code ?

Actuellement, il est accessible sur bitbucket (licencié en LGPL-3):

*   [Les objets Jetty][8] et leurs tests unitaires
*   [L'implémentation Stomp][9] et ses [tests unitaires][10]
*   Le client [Java websocket][11] (classes WebSocketClient, IMessageReceiver, StompWebSocketListener)

Vous aviez deviné, je suis un fanatique du TDD. J'ai donc réaliser un client Websocket en Java. En effet mes tests unitaire lance le server dans un Jetty en mémoire, et agissent comme s'ils étaient des navigateurs.

Dès que je serai un peu plus disponible, je paquagerai l'ensemble indépendamment de mon moteur de jeu, et je le reverserai à [RESThub-js][12]. 
En effet, il y a trés peu de dépendances entre les deux, et je pense que cela peut être réutilisé dans d'autres contextes... Peut être par vous :)  
 
 [1]: https://bitbucket.org/feugy/myth/wiki/Home
 [2]: http://www.graniteds.org/confluence/pages/viewpage.action?pageId=229378
 [3]: http://jersey.java.net/
 [4]: https://bitbucket.org/ilabs/resthub-js/src
 [5]: http://jetty.codehaus.org/jetty/
 [6]: http://blogs.webtide.com/gregw/entry/jetty_websocket_server
 [7]: http://stomp.codehaus.org/Protocol
 [8]: https://bitbucket.org/feugy/myth/src/1a56ca416b5a/chronos-webapp/src/main/java/org/mythicforge/tools/websocket/
 [9]: https://bitbucket.org/feugy/myth/src/1a56ca416b5a/chronos-webapp/src/main/java/org/mythicforge/tools/stomp/
 [10]: https://bitbucket.org/feugy/myth/src/1a56ca416b5a/chronos-webapp/src/test/java/org/mythicforge/tools/stomp/
 [11]: https://bitbucket.org/feugy/myth/src/1a56ca416b5a/chronos-webapp/src/test/java/org/mythicforge/tools/
 [12]: https://bitbucket.org/feugy/myth/src/1a56ca416b5a/chronos-webapp/src/test/java/org/mythicforge/tools/