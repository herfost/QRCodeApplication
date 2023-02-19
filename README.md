# QRCode Application
Applicazione che consente di effettuare login mediante lettura di codice QR.

All'apertura del Web Socket, viene effettuata una richiesta al servizio di generazione di codici QR specificando il proprio id di sessione. Il servizio restituisce un codice QR contenente un'URL incompleto utile per l'autenticazione del utente.

Mediante un'applicazione mobile in cui sono memorizzare lo `username`, la `password` e una variabile di identificazione univoca dell'applicazione, `applicationId`, viene letto il codice QR che contiene parte del `URL` all'API per l'autenticazione.

L'applicazine mobile concatena i rimanenti dati di cui dispone all'URL letto e inoltra una richiesta ad un'altra API responsabile di estrapolare l'id di sessione del web socket dall'URL al fine di identificare il web socket associato all'utente e inoltrarli l'URL (completo) per l'autenticazione di cui, il web socket, si occupa, utilizzando l'URL ricevuto.

## Protocollo
Il web socket riceve dei messaggi specificano il  comportamento che esse deve assumere: il messaggio ricevuto prevede l'operazione da eseguire e l'argomento dell'operazione. Questi due parametri sono seprata da un _divisore_. 

La richeista dell'codice QR, ad esempio, prevede: `GENERATE_QRCODE@http://localhost:8080/QRCodeApplication/qrcode/webSocketId` dove `GENERATE_QRCODE`, `@`, `http://localhost:8080/QRCodeApplication/qrcode/webSocketId` sono rispettivamente l'operazione, il _divisore_ e l'argomento.


#### Esempio QRCode Request
```js
ws.onmessage = (event) => {
    const message = event.data;
    const operation = message.split(DIVEDER)[0];
    const URL = message.split(DIVEDER)[1];
    const webSocketId = URL.split("/")[5];

    // URL = http://localhost:8080/QRCodeApplication/qrcode/webSocketId
    if (GENERATE_QRCODE_OPERATION === operation) {
        document.getElementById("qr-code-block").src = URL;
        document.getElementById("qr-string").innerHTML = URL;
        document.getElementById("authentication-request").innerHTML = 
        "http://localhost:8080/QRCodeApplication/authenticate/" + webSocketId + "/admin/admin/applicationId";
    }

    // URL = http://localhost:8080/QRCodeApplication/authenticate/webSocketId/username/password/applicationId
    if (AUTH_OPERATION === operation) document.location = URL;
};
```

## Servizi
### WebSocketHandler 
Servizio che si occupa di gestire l'interazione con la pagine dell'utente.

```java
All'apertura del web socket viene inoltrata una richiesta all'API per la generazione di codici QR, specificando il proprio id (vedi [Richiesta QRCode Web Socket Client](#esempio-qrcode-request)).
@OnOpen
public static void OnOpen(Session webSocketSession) throws IOException {
    String webSocketId = webSocketSession.getId();

    String optionToken = "GENERATE_QR";
    String divider = Configuration.getDIVIDER();
    String protocol = Configuration.getPROTOCOL();
    String serverName = Configuration.getHOST();
    String serverPort = Configuration.getPORT();
    String contextPath = Configuration.getCONTEXT_PATH();
    String servicePath = Configuration.getQRCODEPATH();

    // URL = GENERATE_QR@http://localhost:8080/QRCodeApplication/qrcode/webSocketId
    String URL = optionToken + divider + protocol + "://" + serverName + ":" + serverPort + "/" + contextPath + "/" + servicePath + "/" + webSocketId;

    webSocketSession.getBasicRemote().sendText(URL);
    SessionHandler.addSession(webSocketId, webSocketSession);
}
```

### QRCode Gereator
Servizio che si occupa di generare codici QR: l'API associata `qrcode/{data}`, la stringa `data` viene convertita in codice QR.

Vengono utlizzate le seguenti librerie: [jar/core-3.2.1.jar](jar/core-3.2.1.jar), [jar/javase-3.2.1.jar](jar/javase-3.2.1.jar) e [jar/QRLib.jar](jar/QRLib.jar)

### Authentication 
Servizio che si oocupati dell'autenticazione degli utenti: l'API associata `authenticate/{webSocketId}/{username}/{password}/{applicationId}`.
```java
@GET
@Path("{webSocketId}/{username}/{password}/{applicationId}")
public Response authenticate(
        @PathParam("webSocketId") String webSocketId,
        @PathParam("username") String username,
        @PathParam("password") String password,
        @PathParam("password") String applicationId
) throws URISyntaxException, URISyntaxException, MalformedURLException {
    UserLogin userLogin = new UserLogin(username, password);
    NewCookie autheticationCookie = SecurityHandler.authenticateUserLogin(userLogin, persistence);
    CookieHandler.addCookie(webSocketId, autheticationCookie);

    return autheticationCookie == null
    ? Response.ok().status(Response.Status.UNAUTHORIZED).build()
    : Response.ok().cookie(autheticationCookie).build(); // TODO: Reindirizzamento
}
```
> Al momento applicationId non viene utilizzata.

Viene effettuata l'autenticazione dell'utente mediante i dati ricevuti, in caso sia valido, riceve un cookie che lo identifica e attesta la sua sessione.

### TL;DR:
L'applicazione web genera un QRCode contenente l'URL per l'autenticazione che, alla lettura da parte dell'applicazione mobile, viene completato con le credenziali dell'utente e viene inoltrata ad un'API che fornisce URL completato all web socket associato all'QR letto. Il web socket inoltra quindi la richiesta di autenticazioen con la stinga completa.

### TODO:
- Realizzare persistenza database.
- Realizzare pagine html (login, signin, ...)
- Realizzare reindirizzamento pagine
- Realizzare generazione cookie
- Prevedere utilizzo applicationId
- Realizzare applicazione mobile
- Aggiungere schemi, diagrammi e disegni