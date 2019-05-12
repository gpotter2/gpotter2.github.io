---
name: sslsockets
title: Tutoriel SSLSockets
description: Comment créer facilement des Sockets sécurisés en Java.
comments: true
---

**MISE EN GARDE**
Le tutoriel qui va suivre a été [rédigé en 2015-2016, en tant que tutoriel non officiel pour OpenClassroom](https://openclassrooms.com/forum/sujet/tutorial-ssl-socket-en-java-android), puis a été retravaillé en une [seconde version en attente sur le courselab](https://openclassrooms.com/courses/3449986?status=waiting-for-publication). Après trois ans d'attente, le courselab ayant fermé, il a été reposté ici (sous format Markdown).
Une traduction anglaise est disponible [ici](/tutos/en/sslsockets). Il n'est peut être malheureusement plus autant à jour qu'à l'époque :/
{: .notice--warning}


Bonjour à tous !

Dans ce tutoriel, on va apprendre à créer des sockets sécurisées, donc cryptées en SSL ou TLS, le tout en Java/Android. Ce tutoriel explique également comment faire fonctionner les sockets en compatibilité avec les versions les plus anciennes d'Android pour TLS/SSL ![](/images/sslsockets/heureux.png). Ce tutoriel explique comment créer des SSLSockets, par lesquelles on replace les Sockets normales, mais pas comment s'en servir !

Si vous n'avez jamais utilisé les Socket / ServerSocket, c'est pas ici que vous apprendrez... Je vous conseille [ce cours](https://openclassrooms.com/courses/introduction-aux-sockets-1).
{: .notice--danger}

Il y a quelque mois, alors que j'effectuais des connexions entre un serveur et un client en Java, le tout compilé dans des exécutables (pas simple a dire ), j'ai voulu effectuer une connexion avec des Sockets sécurisés, afin que personne ne puisse intercépter mes messages secrets...... ![](/images/sslsockets/ninja.png)

Une petite recherche sur le Web, m'a appris qu'il fallait pour cela utiliser des SSLSockets, ce dont je n'avais jamais entendu parler. ![](/images/sslsockets/huh.png)

En gros, il me fallait replacer mes sockets, par des sockets cryptées...

Après environ 2 semaines de recherche, je n'avais trouver que des Tutos incomplets et des explications foireuses.... Les pages en question dataient toutes d'il y a 10 ans, et utilisaient des fonctions inexistantes... ![](/images/sslsockets/unsure.gif) De plus, utilisant Android, rien n'était expliqué sur la rétro-compatibilité des versions...

Alors je me suis épluché les docs ![](/images/sslsockets/pinch.png), les forums, et finit par comprendre mes erreurs, et arriver à un code correct.

**Dans ce mini-tutorial, je vais vous expliquer pas à pas comment créer un certificat auto-signé puis effectuer une connexion entre un serveur et un client, en SSL/TLS, pour vous éviter de passer vos soirées à chercher une vague solution ![](/images/sslsockets/hihi.png)**

Pour ceux qui utilisent Android, c'est le même principe, seul quelque points changent dans la création des certificats...

Partie 0 : Avant de commencer
-----------------------------

Il faut avoir des bases en Socket et ServerSocket. Ce tutoriel apprend à créer des SSLSocket et SSLServerSocket, qui remplacent uniquement les Sockets et ServerSockets. Il n'y est pas expliqué comment s'en servir !

Rappel: Si vous n'avez jamais utilisé les Socket / ServerSocket, c'est pas ici que vous apprendrez... Je vous conseille [ce cours](https://openclassrooms.com/courses/introduction-aux-sockets-1), vous êtes prévenus ![](/images/sslsockets/langue.png)
{: .notice--danger}

##  Les logiciels

On va utiliser Java (bien sûr ![](/images/sslsockets/smile.png)) et _**Porteclé**_, logiciel de certificat avec une interface graphique téléchargeable [ici](https://sourceforge.net/projects/portecle/)

Android utilise le système de cryptographie de BouncyCastle, tandis que Java utilise ses propres libraries. Ainsi, Android utilise les formats BKS (pour BouncyCastle) et Java JKS (Java Key Store). Pour android, il faut donc faire attention aux notes au long du tutoriel expliquant comment adapter le code ici pour Java vers Android. Pas de panique , il faudra juste replacer trois lettres ![](/images/sslsockets/rire.gif)
{: .notice--warning}

Partie 1 : La création de certificats
-------------------------------------

Pour créer les Sockets sécurisées, il faut avoir des certificats. Il permettent d'encrypter les données qui passent par les sockets, et les rendrent indéchiffrables pour les hackers ou autres intercepteurs. Pour continuer, le mieux serait de comprendre ce qu'est un certificat asymétrique...

Si vous ne savez ce qu'est un certificat, je vous conseille la [page wikipédia](https://fr.wikipedia.org/wiki/Certificat_%C3%A9lectronique) ![](/images/sslsockets/ange.png)
{: .notice--info}

Dans cette partie, on va donc créer les certificats sous forme de **Keystore**, format englobant les certificats nécessaire pour Java et Android. Ces **Keystore** sont en gros, des enveloppes pouvant être lues par **Java/Android** contenant les certificats aussi bien publics que privés. Le format utilisé par Java est le **JKS : Java Key Store**.

On peut créer un certificat en ligne de code, mais pour simplifier, ici on va utiliser le logiciel _**Porteclé**_, qui fonctionne en Java. Il possède une interface graphique et est donc facile à utiliser.
{: .notice--info}

## Les certificats sous Android

Cette partie concerne UNIQUEMENT Android, en java, passez votre chemin vers l'étappe 1
{: .notice--warning}

Pour android, il faudra utiliser le BKS au lieu du JKS. C'est une spécificité Android. Android ne fonctionne qu'avec le BKS, et Java qu'avec le JKS.

Ainsi, il faudra sélectionner du BKS au lieu du JKS pour les certificats (soit privé, soit public, soit les 2) qui seront installés et utilisés sous android.

## Création des certificats

Voici les étapes à suivre pour créer facilement deux fichiers JKS ou BKS, l'un pour le serveur et l'autre pour le client:

### Etape 1: Lancement

Lancer Porte clé

 ![](/images/sslsockets/Etappe 1.png)

### Etape 2: Choisir le type de la clé serveur

Pour Android: Comme on est sur le serveur, il est très peu probable de créer un serveur SSL/TLS Android.... mais bon... on ne sait jamais ![](/images/sslsockets/clin.png) **Donc si le SERVEUR est en Android, on met BKS au lieu de JKS**
{: .notice--warning}

Cliquer sur Nouveau --> JKS pour du java (BKS pour l'Android **SERVEUR**)

 ![](/images/sslsockets/Etappe 2_5.png)

### **Etape 3: La clé serveur privée**

On crée une clé privée:

 ![](/images/sslsockets/Etappe 3.1.png)

En _**RSA 4096 (ou 2048)**_ (comme ça c'est bien solide ![](/images/sslsockets/ninja.png))

 ![](/images/sslsockets/Etappe 3.2.png)

Et on rempli le formulaire adapté, en sélectionnant _**SHA256withRSA**_. C'est à jour et sécurisé ![](/images/sslsockets/magicien.png).

Les "Optionnel" peuvent être laissés blancs
{: .notice--info}

![](/images/sslsockets/Etappe 3.31.png)

On appuie ensuite sur OK.

On rentre ensuite un alias. C'est le surnom du certificat. On met ce que l'on veut:

 ![](/images/sslsockets/Etappe 3.4_6.31.png)

On rentre ensuite un mot de passe pour le certificat.

**Ce mot de passe doit être sauvegardé ou mémorisé car on s'en servira plus tard.**

Pour la suite, on appellera ce mot de passe: `MOTDEPASSE1`
{: .notice--warning}

 ![](/images/sslsockets/Etappe 3.5_3.7_6.4.png)

Enfin, on sauvegarde la cle:

 ![](/images/sslsockets/Etappe 3.6.png)

Avec, pour rendre ce cours plus simple, le même mot de passe que l'on note, et mémorise.

Ce mot de passe est, pour la suite, également `MOTDEPASSE1`
{: .notice--warning}

 ![](/images/sslsockets/Etappe 3.5_3.7_6.41.png)

On l'enregistre quelque part:

 ![](/images/sslsockets/Etappe 3.8.png)

On vient de fabriquer la clé privée.
{: .notice--info}

Maintenant, on va s'occuper de la clé publique. Pour celle la, on aura besoin du certificat publique de la clé privée.

### Etape 4 : Exporter la clé publique

On va maintenant extraire le certificat public. On va donc effectuer, toujours sur la clé Privée:

\-> **Export**

 ![](/images/sslsockets/Etappe 4.1.png)

Puis, on sélectionne "**Head certificate**", et "**DER**": on récupère le certificat public, encodé dans le bon langage compatible avec les JKS, BKS

 ![](/images/sslsockets/Etappe 4.2.png)

Et on l'enregistre:

 ![](/images/sslsockets/Etappe 4.3.png)

On appellera ce fichier _**CERTIFICAT\_EXT**_ pour la suite des étapes.
{: .notice--warning}

Pour assurer la rétro-compabilité vers les vielles versions d'Android (API 16 et moins), suivre maintenant la partie 2 pour ce fichier (tant qu'il est ouvert). On reprendra ensuite les étapes à partir d'ici.
{: .notice--info}

On en a maintenant finit avec la clé privée !

### Etape 5 : Choisir le type de la clé publique

Cette fois si, si vous voulez créer un client sur Android, plus fréquent, il faut mettre BKS.
{: .notice--warning}

Cliquer sur Nouveau --> JKS pour du java, et BKS pour du Android.

 ![](/images/sslsockets/Etappe 2_51.png)

### Etape 6 : Importer le certificat

Sur la clé publique, on sélectionne:

 ![](/images/sslsockets/Etappe 6.1.png)

Puis, on sélectionne le fichier _**CERTIFICAT\_EXT **_enregistré précédemment, et on accepte le certificat:

 ![](/images/sslsockets/Etappe 6.21.png)

 ![](/images/sslsockets/Etappe 6.3.png)

Et on rentre ensuite un alias (surnom du certificat). Par simplicité, on met le même que sur la clé privée:

 ![](/images/sslsockets/Etappe 3.4_6.32.png)

On enregistre ensuite le fichier. C'est la clé PUBLIQUE.

![](/images/sslsockets/Etappe 6.5.png)

![](/images/sslsockets/Etappe 3.5_3.7_6.5.png)

Avec un mot de passe que l'on note, et mémorise.

Ce mot de passe est, pour la suite, `MOTDEPASSE2`
{: .notice--warning}

On l'enregistre a côté de la clé privée. NE PAS REPLACER LA CLÉ PRIVÉE PAR LA PUBLIQUE ! On aura besoin des 2 ! 

![](/images/sslsockets/Etappe 6.7.png)

Pour assurer la rétro-compabilité vers les vielles versions d'Android (API 16 et moins), suivre maintenant la partie 2 pour ce fichier (tant qu'il est ouvert). On reprendra ensuite les étapes à partir d'ici.
{: .notice--info}

On en a enfin finit avec la clé Publique.

On a donc ainsi obtenu une clé Publique et une clé Privée. ![](/images/sslsockets/soleil.png)

On n'a plus besoin du fichier CERTIFICAT\_EXT. On peut le supprimer pour ne plus confondre.
{: .notice--warning}

Partie 2 : Certificats pour rétro-compabilité sous Android
----------------------------------------------------------

## Section spéciale Android pourri  !!

Cette partie abordera la compatibilité avec les versions les plus anciennes et sombres d'Android........![](/images/sslsockets/diable.png)

Les versions d'Android aussi vielles devraient être abandonnées.... Cette étape est vraiment pour ceux font de la maintenance des versions sous l'API 16... Vaudrait mieux pas se compliquer la tête ![](/images/sslsockets/smile.png)
{: .notice--error}

Pour les versions d'Android antérieurs à API 16, le **BKS** qui est l'équivalent moderne n'était pas encore pris en compte. Ainsi on utilisera **BKS-V1** qui correspond à l'ancienne version de BouncyCastle (un peu préhistorique ![](/images/sslsockets/pirate.png)).

Pour assurer la compatibilité entre plusieurs version, on aura besoin aussi bien du certificat en **BKS** et en **BKS-V1**. Autrement dit, si votre application doit fonctionner aussi bien en Android API 16 que 23 par exemple, il faudra obtenir deux fichiers, un BKS et un BKS-V1, et changer entre eux en fonction de la version du programme, on verra ce changement dans la partie code.

Cette étape transforme une clé **BKS** en **BKS-V1**:

![](/images/sslsockets/Etappe 7.bmp)

On effectue cette étape sur les **BKS** que l'on veut doubler. On a besoin des mots de passe correspondants au fichier que l'on veut doubler.
{: .notice--info}

Ensuite, on la **ré-sauvegarde A COTE de l'ancienne clé BKS**.

Partie 3 : Le code
------------------

Ahhh, enfin du code ![](/images/sslsockets/zorro.png)

Celui-ci est assez simple, et commenté:

Prenez garde ! Si vous vouler utiliser SSL au lieu de TLS, il faut le changer dans le code !
{: .notice--error}

**Idem pour utiliser android (explications en détail plus bas) !!**

On peut remplacer **_TLSv1.2_**par **_SSL_**  , pour supporter par les vielles versions. Je vous le déconseille cependant ![:)](/images/sslsockets/smile.png)
{: .notice--warning}

## Classe utilitaire pour le serveur:

```java
/**
 * Ca c'est une fonction utilitaire permettant de récupérer le "vérifieur de certificat", on prend uniquement celui correspondant au certificat pour minimiser le temps de chargement à la connexion
 * 
 * Pas très intéressante...
 */
private static X509TrustManager tm(KeyStore keystore) throws NoSuchAlgorithmException, KeyStoreException {
    TrustManagerFactory trustMgrFactory = TrustManagerFactory.getInstance(TrustManagerFactory.getDefaultAlgorithm());
    trustMgrFactory.init(keystore);
    //on prend tous les managers
    TrustManager trustManagers[] = trustMgrFactory.getTrustManagers();
    for (int i = 0; i < trustManagers.length; i++) {
        if (trustManagers[i] instanceof X509TrustManager) {
            //on renvoie juste celui que l'on va utiliser
            return (X509TrustManager) trustManagers[i];
        }
    }
    return null;
};
/**
 * Ca c'est une fonction utilitaire permettant de récupérer le "gestionnaire de mot de passes des clés" (en gros)
 * 
 * Pas très intéressante...
 */
private static X509KeyManager km(KeyStore keystore, String password) throws NoSuchAlgorithmException, KeyStoreException, UnrecoverableKeyException {
    KeyManagerFactory keyMgrFactory = KeyManagerFactory.getInstance(KeyManagerFactory.getDefaultAlgorithm());
    keyMgrFactory.init(keystore, password.toCharArray());
    //on prend tous les managers
    KeyManager keyManagers[] = keyMgrFactory.getKeyManagers();
    for (int i = 0; i < keyManagers.length; i++) {
        if (keyManagers[i] instanceof X509KeyManager) {
            //on renvoie juste celui que l'on va utiliser
            return (X509KeyManager) keyManagers[i];
        }
    }
    return null;
};



/**
 * Le vrai morceau, que l'on utilisera
 */
public static SSLServerSocket getServerSocketWithCert(int port, InputStream pathToCert, String passwordFromCert) throws IOException,
       KeyManagementException, NoSuchAlgorithmException, CertificateException, KeyStoreException, UnrecoverableKeyException{
           TrustManager[] tmm = new TrustManager[1];
           KeyManager[] kmm = new KeyManager[1];
           //On charge le lecteur de Keystore en fonction du format
           //ATTENTION
           //POUR LES SERVEURS android, remplacez le JKS par BKS, si c'est juste le client qui est android, laissez JKS
           KeyStore ks  = KeyStore.getInstance("JKS");
           //On charge le Keystore avec sont stream et son mot de passe
           ks.load(pathToCert, passwordFromCert.toCharArray());
           //On lance les gestionnaires de clés et de vérification des clients
           tmm[0]=tm(ks);
           kmm[0]=km(ks, passwordFromCert);
           //On démarre le contexte, autrement dit le langage utilisé pour crypter les données
           //ATTENTION
           //Ici, on peut remplacer  TLSv1.2 par SSL, mais il faudra le faire aussi bien dans le client que le serveur
           SSLContext ctx = SSLContext.getInstance("TLSv1.2");
           ctx.init(kmm, tmm, null);
           //On lance la serversocket sur le port indiqué, avec le contexte fourni
           SSLServerSocketFactory socketFactory = (SSLServerSocketFactory) ctx.getServerSocketFactory();
           return (SSLServerSocket) socketFactory.createServerSocket(port);
       }
```         

## Classe utilitaire pour le client

```java
/**
 * Ca c'est une fonction utilitaire permettant de récupérer le "vérifieur de certificat", on prend uniquement celui correspondant au certificat pour minimiser le temps de chargement à la connexion
 * 
 * Pas très intéressante...
 */
private static X509TrustManager[] tm(KeyStore keystore) throws NoSuchAlgorithmException, KeyStoreException {
    TrustManagerFactory trustMgrFactory = TrustManagerFactory.getInstance(TrustManagerFactory.getDefaultAlgorithm());
    trustMgrFactory.init(keystore);
    //on prend tous les managers
    TrustManager trustManagers[] = trustMgrFactory.getTrustManagers();
    for (TrustManager trustManager : trustManagers) {
        if (trustManager instanceof X509TrustManager) {
            X509TrustManager[] tr = new X509TrustManager[1];
            //on renvoie juste celui que l'on va utiliser
            tr[0] = (X509TrustManager) trustManager;
            return tr;
        }
    }
    return null;
}

//Le vrai du code 

public static SSLSocket getSocketWithCert(InetAddress ip, int port, InputStream pathToCert, String passwordFromCert) throws IOException,
       KeyManagementException, NoSuchAlgorithmException, CertificateException, KeyStoreException {
           X509TrustManager[] tmm;
           //ATTENTION
           //Android, remplacez JKS par BKS :)
           //On charge le lecteur de Keystore en fonction du format
           KeyStore ks  = KeyStore.getInstance("JKS");
           //On charge le Keystore avec sont stream et son mot de passe
           ks.load(pathToCert, passwordFromCert.toCharArray());                        //On démarre le gestionnaire de validation des clés
           tmm=tm(ks);
           //On démarre le contexte, autrement dit le langage utilisé pour crypter les données
           //On peut replacer TLSv1.2 par SSL
           SSLContext ctx = SSLContext.getInstance("TLSv1.2");
           ctx.init(null, tmm, null);
           //On créee enfin la socket en utilisant une classe de création SSLSocketFactory, vers l'adresse et le port indiqué
           SSLSocketFactory SocketFactory = ctx.getSocketFactory();
           SSLSocket socket = (SSLSocket) SocketFactory.createSocket();
           socket.connect(new InetSocketAddress(ip, port), 5000);
           return socket;
       }
``` 

## Utilisation:

Euh, merci mais comment on s'en sert ?
{: .notice--primary}

On va juste remplacer la création de votre socket ou serveur socket, par ça:

Oh ! Mais qui voilà ??? Les mots de passe qu'il fallait conserver !
{: .notice--warning}

Ici on suppose que le stream provient d'un fichier intégré au .jar, il peut aussi être externe, auquel cas on replacera le this.getClass().get..... par n'importe quel autre stream pointant vers la clé
{: .notice--info}

### Pour une SSLServerSocket (serveur):

```java
SSLServerSocket ssocket = SSLServerSocketKeystoreFactory.getServerSocketWithCert(port, this.getClass().getResourceAsStream("/CLE_PRIVEE.jks"), "MOTDEPASSE1")
```                 

### Pour une SSLSocket (client):

```java
SSLSocket socket = SSLSocketKeystoreFactory.getSocketWithCert(addresse_ip, port, this.getClass().getResourceAsStream("/CLE_PUBLIQUE.jks"), "MOTDEPASSE2")
```

Des classes toute faites comportant le code sont disponibles en anglais. Le lien est dans l'annexe.
{: .notice--info}

## Section spéciale Android !!

Cette section est pour android uniquement !
{: .notice--warning}

Les dernières versions d'Android (23+) ne supportent plus le SSL, jugé obsolète. Ainsi, on doit **obligatoirement** remplacer le **SSL **du code par **TLSv1.2**. C'est aussi conseillé car c'est plus sûr ![](/images/sslsockets/magicien.png)

### Section spéciale Android rétro-comptabilité :

Cette section est uniquement pour ceux qui avaient voulu rendre compatible leur app avec les anciennes versions !!
{: .notice--error}

En fait, on choisi en fonction de la version d'Android le bon certificat à utiliser, sachant que **JELLY\_BEAN\_MR1** est la version du changement du **BKS-v1** au **BKS**:

```java
int bks_version;
if (Build.VERSION.SDK_INT >= Build.VERSION_CODES.JELLY_BEAN_MR1) {
    bks_version = R.raw.publickey; //Le fichier BKS
} else {
    bks_version = R.raw.publickey_v1; //Le fichier BKS (v-1)
}
InputStream stream = getResources().openRawResource(bks_version);  
```

Et on utilise ensuite ce **stream** pour la classe plus haut ![](/images/sslsockets/smile.png)

Partie 4 : Conclusion
---------------------

Ainsi on aura créé une connexion entre un serveur et un client. Le tout en SSL ou TLS...

On arrive donc a la fin de ce tuto... 

En espérant avoir aidé certains ![:)](/images/sslsockets/smile.png)

Annexe:
-------

[Le code en classes toutes faites, et plus complètes, mais en anglais... ](https://github.com/gpotter2/SSLKeystoreFactories) (pas de problème j'ai les droits dessus ![](/images/sslsockets/langue.png))

[Android 4.2: nouvelle cryptographie](https://android-developers.blogspot.fr/2013/02/using-cryptography-to-store-credentials.html) (explication de pourquoi Android a abandonnée le BKS-1)
