---
name: sslsockets
title: SSLSockets Tutorial
description: A steb-by-step tutorial on how to create SSLSockets in Java/Android.
comments: true
permalink: /tutos/en/sslsockets
---

DISCLAIMER: This tutorial was written for a French forum in 2015. This version is a translation that I made quickly: it probably has mistranslations. Please share them in the comments, or update this page on GitHub :) ![](/images/sslsockets/langue.png). [French version of the tutorial](/tutos/sslsockets)
{: .notice--warning}

In this tutorial, we'll see how to create secure sockets (encrypted with TLS or SSL), in Java/Android. This tutorial also explains how to handle backward-compatibility with the older versions of Android for TLS/SSL ![](/images/sslsockets/heureux.png). It explains how to create SSLSockets, which replace normal Sockets, but not how to use them!

If you have never used Socket / ServerSocket, this is not where you will learn it. I recommend looking it up.
{: .notice--danger}

A few months ago, I was trying making a server-client system in Java, where each was compiled in .jar. At some point, I thought it would be nice to make the connection using secure Sockets, so that nobody could interoperate my secret messages :)

After a little research on the Web, I learned that it was necessary to use SSLSocketFactory, but couldn't really find any valuable examples, for instance that were not using `System.setProperty` hacks. Some docs are 10 years old, and use inexistent functions. Moreover, using Android, nothing worked, and I got unclear "wrong keystore version" messages.

For those who use Android, it's the same principle, only some points change in the creation of certificates. Those will be clearly marked.
{: .notice--info}

Part 0: Before you start
------------------------

You need to know how to use `Socket` and `ServerSocket`. This tutorial teaches how to create SSLSocket and SSLServerSocket, to replace them; not HOW to use them.

Otherwise, I advise you to look it up online :)
{: .notice--danger}

## Softwares

We will use Java (of course) and **Porteclé**, a certificate manager software with a downloadable graphical interface [here](https://sourceforge.net/projects/portecle/).

Android uses the BouncyCastle cryptography system, while Java uses its own libraries. Thus, Android uses the BKS (for BouncyCastle) and Java JKS (Java Key Store) formats. For android, you have to pay attention to the notes throughout the tutorial explaining how to adapt the code from Java to Android.
{: .notice--warning}

Part 1: Creating certificates
-----------------------------

**If you know what you are doing** (already have the certificates, or know how to build them using easy-rsa or similar, **feel free to skip this part** :)
{: .notice--info}

To create secure sockets, you must have certificates. To continue, it would be best to understand what an asymmetric certificate is, though it's not compulsory :/

If you do not know what a certificate is, I recommend the wikipedia page.
{: .notice--info}

In this part, we will create certificates in the form of Keystore, a format used by Java and Android. These Keystores are basically envelopes that contain either a public or private certificate. The format used by Java is the **JKS: Java Key Store**.

We can create certificates using easy-rsa, but for simplicity, here we will use **Porteclé**, which runs in Java. It has a graphical interface and is therefore easy to use.
{: .notice--warning}

## Android certificates

This part is about Android ONLY, in java, go to step 1
{: .notice--warning}

For android, you will need to use the **BKS** instead of the JKS. It's an Android specificity. Android only works with BKS, and Java only with JKS.

Thus, it will be necessary to select **BKS** instead of the **JKS** when building the certificates (either private or public) that will be installed and used under android.

## Creating certificates

Below is a step-by-step **Porteclé** tutorial, to easily create the two **JKS** (or **BKS**) keystores, one for the server and the other for the client.

### Step 1: Launch

Start "Porte clé"

 ![](/images/sslsockets/Etappe 1.png)

### Step 2: Choose the type of the server key

For Android: If the SERVER runs on Android (?!), we put **BKS** instead of **JKS**.
{: .notice--warning}

Click on New -> **JKS** for Java (**BKS** for the Android SERVER)

 ![](/images/sslsockets/Etappe 2_5.png)

### Step 3: The private server key

Let's create a private key:

 ![](/images/sslsockets/Etappe 3.1.png)

In RSA 4096 (or 2048) (the more the safer, but the heavier ![](/images/sslsockets/ninja.png))

 ![](/images/sslsockets/Etappe 3.2.png)

And fill in the form, selecting SHA256withRSA (or better). It is up to date and secure.

"Optional" fields can be left blank
{: .notice--info}

![](/images/sslsockets/Etappe 3.31.png)

Then press OK.

Then we enter an alias. This is the nickname of the certificate. You can put whatever you want:

 ![](/images/sslsockets/Etappe 3.4_6.31.png)

Let's enter a password for the certificate.
This password must be saved or stored as it will be used later.

From now on, we will call this password: `PASSWORD1`
{: .notice--warning}

 ![](/images/sslsockets/Etappe 3.5_3.7_6.4.png)

Finally, we save the key:

 ![](/images/sslsockets/Etappe 3.6.png)

With, to make this course more simple, the same password that we noted previously.

This password is also `PASSWORD1`
{: .notice--warning}
 
 ![](/images/sslsockets/Etappe 3.5_3.7_6.41.png)

Let's save it somewhere:

 ![](/images/sslsockets/Etappe 3.8.png)

We just made the private key!
{: .notice--info}

Now we will take care of the public key. For that, we will need to generate the public certificate from the private key.

### Step 4: Export the public key

Let's extract the public certificate. Still on the private key:

\-> Export

 ![](/images/sslsockets/Etappe 4.1.png)

Pick "Head certificate", and "DER".

 ![](/images/sslsockets/Etappe 4.2.png)

Let's save it:

 ![](/images/sslsockets/Etappe 4.3.png)

This file will be called **CERTIFICAT\_EXT** from now on.
{: .notice--warning}

To ensure backward compatibility with older versions of Android (API 16 and below), please follow **Part 2** for this file now (as long as it is open). Then resume the steps from here.
{: .notice--info}

We are now done with the private key!

### Step 5: Choose the type of the public key

If you want to create a client on Android you have to put **BKS**.
{: .notice--warning}

Click on New -> **JKS** for java, and **BKS** for Android.

 ![](/images/sslsockets/Etappe 2_51.png)

### Step 6: Import the certificate

On the public key, select:

 ![](/images/sslsockets/Etappe 6.1.png)

Then, select the file **CERTIFICAT\_EXT** previously saved, and accept the certificate:

 ![](/images/sslsockets/Etappe 6.21.png)

![](/images/sslsockets/Etappe 6.3.png)


Now pick an alias (certificate nickname). For simplicity, we'll put the same thing on the private key:

 ![](/images/sslsockets/Etappe 3.4_6.32.png)

Then save the file. This is the **PUBLIC key**.

![](/images/sslsockets/Etappe 6.5.png)

![](/images/sslsockets/Etappe 3.5_3.7_6.5.png)

With a password that we note, and save.

This password is, from now on, `PASSWORD2`
{: .notice--warning}

Let's save it next to the private key. DO NOT REPLACE THE PRIVATE KEY BY THE PUBLIC! We will need both!

![](/images/sslsockets/Etappe 6.7.png)

To ensure backward compatibility with older versions of Android (API 16 and below), please follow **Part 2** for this file now (as long as it is open). Then resume the steps from here.
{: .notice--info}

We're finally done with the public key.

We now have a **public key** and a **private key**.

We no longer need the **CERTIFICAT\_EXT** file. It can be deleted to no longer confuse.
{: .notice--warning}

Part 2: Certificates for Android back-compatibility
---------------------------------------------------

## Rotten Android versions ahead!

This part will discuss compatibility with the older and darker versions of Android ![](/images/sslsockets/diable.png)

Android versions this old _should_ be dropped. Only those who do maintenance of Apps below API 16 should follow this.
{: .notice--error}

For versions of Android prior to API 16, the BKS which is the modern format had another implementation. Thus **BKS-V1** which corresponds to the old version of BouncyCastle needs to be used (a little prehistoric ![](/images/sslsockets/pirate.png)).

To ensure the compatibility between several versions, you will need both certificates in **BKS** and **BKS-V1**. For instance, if your application should work on Android API 16 and 23, it will be necessary to obtain two files, a **BKS** and a **BKS-V1**, and change between them depending on the version of the API. This will shown below in the code part.

This step transforms a BKS key into BKS-V1:

![](/images/sslsockets/Etappe 7.bmp)

It should be performed on the **BKS** that we want to duplicate. You'll need the passwords matching the file ypu want to double (`PASSEWRD1` or `PASSWORD2`).
{: .notice--info}

Save it under a different file.

Part 3: The code
----------------

Ahhh, finally code ![](/images/sslsockets/zorro.png)

This one is weird, but commented:

Watch out! If you want to use SSL instead of TLS, you have to change it within the code!
{: .notice--error}

Even though TLSv1.2 can be replaced by SSL, to support older versions, it's not recommanded.
{: .notice--warning}

## Utility class for the server:

Note: in the french version of this tutorial, the code is embed and explained. However you really don't need it: they are plenty of comments on the GitHub version.
{: .notice--info}

[SSLServerSocketKeystoreFactory.java](https://github.com/gpotter2/SSLKeystoreFactories/blob/master/src/fr/gpotter2/SSLKeystoreFactories/SSLServerSocketKeystoreFactory.java)

## Utility class for the client:

[SSLSocketKeystoreFactory.java](https://github.com/gpotter2/SSLKeystoreFactories/blob/master/src/fr/gpotter2/SSLKeystoreFactories/SSLSocketKeystoreFactory.java)

## Usage:

Uh, thank you but how do I using this?
{: .notice--primary}

By replacing the creation of your Socket or ServerSocket with::

You'll need the passwords that had to be saved!
{: .notice--warning}

Here we assume that the stream comes from a file packed within the `.jar`, but it could also be external, in which case you'll need to replace `this.getClass().getResourceAsStream()` with some other stream pointing to the keystore.
{: .notice--info}

## For an SSLServerSocket (server):

```java
SSLServerSocket ssocket = SSLServerSocketKeystoreFactory.getServerSocketWithCert(port, this.getClass().getResourceAsStream("/PRIVATE_KEY.jks"), "PASSWORD1")
```                 

## For an SSLSocket (client):

```java
SSLSocket socket = SSLSocketKeystoreFactory.getSocketWithCert(addresse_ip, port, this.getClass().getResourceAsStream("/PUBLIC_KEY.jks"), "PASSWORD2")
```

## Android only ahead

This section is for android only!
{: .notice--warning}

The latest versions of Android (23+) no longer support SSL, considered obsolete. Thus, it is mandatory to replace the SSL code by **TLSv1.2**.

### Android backward-compability:

This section is only for those who wanted to make their app compatible with older versions !!
{: .notice--error}

According to the version of Android, let's choose the correct certificate version to use, knowing that **JELLY\_BEAN\_MR1** is the version where **BKS-v1** became **BKS**:

```java
int bks_version;
if (Build.VERSION.SDK_INT> = Build.VERSION_CODES.JELLY_BEAN_MR1) {
    bks_version = R.raw.publickey; // The BKS file
} else {
    bks_version = R.raw.publickey_v1; // The BKS file (v-1)
}
InputStream stream = getResources (). OpenRawResource (bks_version);
```

And then we use this stream as argument for the classes presented above.

Part 4: Conclusion
------------------

This will create a connection between a server and a client.

That's it ! I hope it helped [](/images/sslsockets/langue.png) !

Annex:
------

[Full code](https://github.com/gpotter2/SSLKeystoreFactories) (MIT [](/images/sslsockets/langue.png))

[Android 4.2: new cryptography](https://android-developers.blogspot.fr/2013/02/using-cryptography-to-store-credentials.html) (explanation of why Android has abandoned the BKS-1)
