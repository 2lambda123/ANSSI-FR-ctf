# Challenge RScA

*Fichiers fournis :* `sniff_0` et `sniff_1`.

� la suite d'une intervention, nos �quipes ont r�cup�r� un t�l�phone s�curis� utilis� par un agent appartenant � une organisation criminelle. Depuis sa r�cup�ration, nous avons pris soin de laisser le t�l�phone en permanence sous tension.

Habituellement, ce t�l�phone est notamment utilis� pour recevoir des instructions sign�es par un individu plus haut dans la cha�ne hi�rarchique. Le t�l�phone est charg� de *v�rifier* les signatures de ces messages. Une phase de r�troconception nous a permis d'identifier que l'algorithme utilis� pour cette op�ration est un **RSA**.

La particularit� forte de notre contexte est l'**absence totale d'information sur les param�tres publics utilis�s par l'algorithme**. De plus, la v�rification est impl�ment�e de mani�re s�curis�e, de fa�on � emp�cher la r�cup�ration de ces param�tres.

L'objectif final de notre mission est de parvenir � signer des messages � la place du sup�rieur hi�rarchique, afin de tendre un pi�ge aux membres de l'organisation. Pour cela, nous vous demandons de retrouver tous les param�tres utilis�s.

La r�troconception nous a appris plusieurs points importants.

Premi�rement, nous avons pu retrouver l'impl�mentation du RSA. Voici le pseudo-code que nous avons pu reconstruire, o� on note phi(N) l'indicatrice d'Euler de N :

```
Function RSA(m, e, phi(N), N):
    r  <- random(0, 2**32)
    e' <- e + r * phi(N)
    accumulator = 1
    dummy = 1
    for i from len(e') - 1 to 0:
        accumulator <- (accumulator * accumulator) mod N
        tmp <- (accumulator * m) mod N
        if (i-th lsb of e') == 1:
            accumulator <- tmp
        else:
            dummy <- tmp
    return accumulator
```

Deuxi�mement, il s'av�re que les deux op�rations de multiplications modulaires

```
accumulator <- (accumulator * accumulator) mod N
```
et

```
tmp <- (accumulator * m) mod N
```

sont effectu�es en faisant appel � un acc�l�rateur mat�riel.

Au d�marrage du t�l�phone, le bloc responsable de la signature va chercher dans une carte SIM les param�tres e et N. Le param�tre N est fourni � l'acc�l�rateur et stock� dans un SRAM non lisible. Tout red�marrage du t�l�phone impliquerait le verrouillage de la carte SIM, que nous ne pouvons pas d�bloquer (son propri�taire �tant plut�t muet quant au code PIN n�cessaire).

Du c�t� des bonnes nouvelles, nous sommes parvenus lors de notre intervention � r�cup�rer un morceau de la documentation de cet acc�l�rateur, dont nous vous fournissons un m�mo ci-dessous. De plus, nous sommes parvenus � sniffer le bus de communication entre cet acc�l�rateur et le bloc responsable de la v�rification de signature.

Depuis l'intervention, nous avons re�u deux messages diff�rents de l'ext�rieur. Le *sniffing* du bus lors de la v�rification des signatures de chacun de ses messages vous est donn�. Le contenu des messages en lui-m�me est anecdotique.

Votre objectif est de retrouver les param�tres (N, p, q, e, d) du RSA, avec :

* N : le module "public" du RSA ;
* p, q : les facteurs premiers de N (p * q = N, et p < q) ;
* e : l'exposant "public" stock� sur le t�l�phone (0 < e < phi(N)) ;
* d : l'exposant "priv�" utilis� pour signer les messages (0 < d < phi(N)).

Le flag � retrouver est de la forme ECSC{N+p+q+e+d} avec N+p+q+e+d �crit en hexad�cimal.


# Protocole de communication

Les trames envoy�es par le bloc RSA � l'acc�l�rateur modulaire suivent la forme suivante :

```
| senderId | receiverId | opcode | operand1(opt) | operand2(opt) |
```

Les trames envoy�es par l'acc�l�rateur au bloc RSA suivent la forme suivante :

```
| senderId | receiverId | operand1 |
```

avec :

* senderId : 1 octet codant l'Id du bloc exp�diteur
* receiverId : 1 octet codant l'Id du bloc destinataire
* opcode (optionnel) : 1 octet codant l'op�ration (voir ci-apr�s)
* operand1 : un certain nombre d'octets repr�sentant le premier op�rande
* operand2 : un certain nombre d'octets repr�sentant le second op�rande

## Description des op�rations

### Chargement du module N

Demande le chargement du module N dans l'acc�l�rateur hardware.

* opcode : 0x11
* operand1 : 2 octets repr�sentant la taille t de N en bits
* operand2 : t octets repr�sentant la valeur de N

### Addition modulaire

Demande l'addition modulaire de deux op�randes de taille t.
La taille t est inf�r�e par l'acc�l�rateur gr�ce � sa connaissance de N.

* opcode : 0x22
* operand1 : t octets repr�sentant la valeur du premier op�rande
* operand2 : t octets repr�sentant la valeur du second op�rande

### Soustraction modulaire
Demande la soustraction modulaire de deux op�randes de taille t.
La taille t est inf�r�e par l'acc�l�rateur gr�ce � sa connaissance de N.

* opcode : 0x33
* operand1 : t octets repr�sentant la valeur du premier op�rande
* operand2 : t octets repr�sentant la valeur du second op�rande

### Inversion modulaire
Demande l'inversion modulaire d'un op�rande de taille t. Si l'�l�ment n'a pas d'inverse, retourne 0.
La taille t est inf�r�e par l'acc�l�rateur gr�ce � sa connaissance de N.

* opcode : 0x44
* operand1 : t octets repr�sentant la valeur � inverser
* operand2 : non pr�sent

### Multiplication modulaire
Demande la multiplication modulaire de deux op�randes de taille t.
La taille t est inf�r�e par l'acc�l�rateur gr�ce � sa connaissance de N.

* opcode : 0x55
* operand1 : t octets repr�sentant la valeur du premier op�rande
* operand2 : t octets repr�sentant la valeur du second op�rande

### Mise au carr� modulaire
Demande la mise au carr� modulaire d'un op�rande de taille t.
La taille t est inf�r�e par l'acc�l�rateur gr�ce � sa connaissance de N.

* opcode : 0x66
* operand1 : t octets repr�sentant la valeur � mettre au carr�
* operand2 : non pr�sent

### Demande de r�ponse
Demande de la r�ponse suite � une demande d'op�ration.

* opcode : 0x77
* operand1 : non pr�sent
* operand2 : non pr�sent

La r�ponse de taille t suit le format de trames d�crit au d�but de ce document.
