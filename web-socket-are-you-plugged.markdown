Title: Web Socket : Are you plugged ?
Author: Damien Feugas
Date: Apr 19 2011 11:46:00 GMT-0200 (CDT)
Categories: java, javascript, mythicforge, graniteds, jetty, JMS, STOMP, websocket

# Un petit peu de contexte

Depuis � peu pr�s deux ans, je r�alise un [MMORPG][1] gratuit et OpenSource avec un serveur Java et eux clients Flex (un d'administration et un de jeu).

J'ai utilis� le merveilleux framework [GraniteDS][2] qui est un pont entre le java et flex, comme le BlazeDS d'Adobe, avec plus de fonctionnalit�s encore.  
  
Malheureusement, j'ai surtout besoin de flexibilit� cot� client de jeu (tout le GameDesign est configurable via l'admin), et j'ai d�cid� finalement de tout r��crire en technologies Html 5/Css 3/Js.

Les 3 grandes fonctionnalit�s de GraniteDS qui doivent donc �tre remplac�es sont les suivantes:

1.  L'invocation distante de m�thodes Java, remplac�e par une API REST. Chaque m�thode du serveur est une url qui produit et consomme du XML ou du JSON. J'ai choisi [Jersey ][3]pour �a (impl�mentation de r�f�rence de la spec JAX-RS), et expliqu� la migration dans un article pr�c�dent.  
     
2.  Le MCV client "tide", remplac� par [RESTHub-js][4], un petit framework client en javascript.  
     
3.  Le push serveur : les clients flex sont constamment connect� au serveur qui leur envoi les mise � jour d�clench�e par les autres joueurs. Ce billet explique comment j'ai remplac� cette partie par l'utilisation des WebSockets.
 
# Alors, comment on joue ?

Les Web sockets sont juste... des socket. C'est un canal connect� entre le navigateur et le serveur, ni plus, ni moins. Vous avez donc besoin d'un navigateur r�cent (Chrome, IE9 or Firefox 4 correctement configur�), et d'un server.

Il y a quelques serveur Java qui impl�ment le protocole : jWebSocket, Kaazing, webbit... Mais aucun d'entre eux n'est aussi un conteneur de Servlet, la base de nos serveurs java. A l'exception de [Jetty][5].  
  
Sans rentrer dans les d�tails, Jetty est un serveur Http+Servlet+WebSocket tr�s puissant �crit en java, qui peut �tre utilis� en mode embarqu� ou standalone. Il impl�mente le brouillon de la norme Websocket depuis un petit moment, et [plut�t simplement][6].

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

Plut�t facile, non ? Cette "esp�ce de servlet" doit �tre d�clar�e dans le descripteur web.xml :

    <servlet>
        </servlet-name>
        <servlet-class>org.dummy.WebSocketDummyServlet</servlet-class>
        <load-on-startup>1</load-on-startup>
    </servlet>
    <servlet-mapping>
        <servlet-name>wsServlet</servlet-name>
        <url-pattern>/*</url-pattern>
    </servlet-mapping>


Vous avez besoin d'envoyer des messages � tous les clients connect� ? Vous n'avez qu'a stocker les instance de  DummyWebSocket cr��es, et ajouter une m�thode qui utilisera _outbound.sendMessage().

# Je suis connect� ! Mais je peux rien faire...

Comme je disais, vous n'avez qu'un tuyau connect�. Il transporte des cha�ne de caract�res et des bits. C'est efficace, mais pas vraiment utilisable en tant que tel.

Il est donc n�cessaire d'impl�menter un protocole au dessus de ce tuyau. Ce dernier d�pendra de vos besoin. Une application de messagerie instantann�e ? Utilisez XMPP. Une application de streaming ? Pourquoi pas RTP. Un jeu FPS ? cr�ez  votre propre protocole.  
GraniteDS proposait un m�canisme de publication-souscription de POJO s�rialis�s, proche de JMS. Heureusement, il existe un �quivalent parfait : [STOMP][7].

Une minute ! Google donne quelques r�sultat pour "STOMP Websocket Java". Notamment ActiveMQ, RabittMQ et HornetMQ, fameux brokers JMS. Utilisons les. Enfin non : c'est vraiment de l'overkill : je n'avais pas besoin de toute cette m�canique complexe...

Alors j'ai impl�ment� le protocole STOMP (sans la gestion transactionnelle) et �a m'a pris deux jours. En fait, STOMP est vraiment tr�s simple (tout est en texte, et les sauts de lignes sont significatifs) :

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

**et le client Y re�oit:**
	
	MESSAGE
	destination:/topic-1
	message-id: <message-identifier>

	hello everyone !
	^@

# Et cot� client justement ?

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
  
Je n'ai pas encore choisi d'impl�mentation STOMP cot� client et d�s que je l'aurai fait, je modifierai cet article.  
  
Juste un avertissement : nous l'avons test� � travers des proxies, et �a fonctionne tr�s bien.  
Pas de d�connexion intempestives, pas de ralentissements.  
Mais cela n�cessite que le client envoi un keep-alive � travers le socket. Le serveur n'a pas besoin de r�pondre.  
  
Un message de keep-alive toutes les 10 secondes marche bien, mais j'imagine que cela d�pend des configuration des proxies et firewall travers�s.

# J'adore ! O� est le code ?

Actuellement, il est accessible sur bitbucket (licenci� en LGPL-3):

*   [Les objets Jetty][8] et leurs tests unitaires
*   [L'impl�mentation Stomp][9] et ses [tests unitaires][10]
*   Le client [Java websocket][11] (classes WebSocketClient, IMessageReceiver, StompWebSocketListener)

Vous aviez devin�, je suis un fanatique du TDD. J'ai donc r�aliser un client Websocket en Java. En effet mes tests unitaire lance le server dans un Jetty en m�moire, et agissent comme s'ils �taient des navigateurs.

D�s que je serai un peu plus disponible, je paquagerai l'ensemble ind�pendamment de mon jeu, sous la forme d'un projet OpenSource. En effet, il y a tr�s peu de d�pendances entre les deux, et je pense que cela peut �tre r�utilis� dans d'autres contextes... Peut �tre par vous :)  
 
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